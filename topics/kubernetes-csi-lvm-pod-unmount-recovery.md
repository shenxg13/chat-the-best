# Kubernetes CSI/LVM：TiKV Pod 停止后卡在 Terminating（vol_data.json 丢失与重复 umount）

## 适用场景及处理结果

- **现场日期**：2026-10-14。
- **环境**：Kubernetes 上的 TiKV（有状态 Pod），PV 由本地 LVM CSI 驱动 `lvmplugin.csi.alcor.io` 管理。
- **操作约束**：生产节点上不能重启 kubelet；不希望删除或重新挂载承载业务数据的 LV；尽量只操作故障 Pod 的卷目录。
- **结果**：通过恢复故障 Pod 的 CSI 元数据、补齐空 `mount` 目录，并在确认真实 LV 已卸载后，对该空目录执行 **self bind mount**，使 CSI 下一次 `umount` 有可卸载的挂载点，Pod 清理最终成功。
- **性质**：现场验证有效的应急恢复技巧，**不是 Kubernetes/CSI 官方通用修复流程**，需严格满足前置条件。

## 故障现象与逐步定位

### 阶段一：缺失 `vol_data.json`

Pod 已正常停止，LV 本身仍存在；发现故障 Pod 的整个 CSI volume 目录不见了，但 kubelet 持续尝试卸载该卷并报错：

~~~text
operationExecutor.UnmountVolume failed
kubernetes.io/csi: unmounter failed to load volume data file
.../volumes/kubernetes.io~csi/<volume>/vol_data.json:
open .../vol_data.json: no such file or directory
~~~

通过检查**另一个正常运行的 TiKV Pod**，确认当前环境的正确目录布局：

~~~text
/var/lib/kubelet/pods/<pod-uid>/volumes/kubernetes.io~csi/<volume>/
├── vol_data.json       # kubelet 的 CSI 元数据，存在于宿主机文件系统
└── mount/              # 实际 PV/LV 挂载目标，数据在其下
~~~

因此：**不能把元数据缺失理解成 PV 已经 umount、所以 PV 内的 `vol_data.json` 消失**；元数据与 `mount/` 是同级，不在 LV 文件系统内。

表现与 Kubernetes 历史问题 [kubernetes/kubernetes#85280](https://github.com/kubernetes/kubernetes/issues/85280) 类似：卷已完成部分清理，但残留的卸载/卷状态触发后续重试，重试再读取已消失的元数据。本案尚未从 kubelet/驱动源码或完整事件序列证明具体的首次删除顺序，因此**这部分属于根因线索，不是已确认根因**。

### 阶段二：恢复元数据后遇到 `umount: not mounted`

按正常 TiKV 的 JSON 结构恢复故障卷的 `vol_data.json` 后，错误前进到 CSI 驱动执行卸载：

~~~text
UnmountVolume.TearDown failed
rpc error: code = Internal
desc = unmounting failed: exit status 32
cmd: umount /var/lib/kubelet/pods/<pod-uid>/volumes/kubernetes.io~csi/<volume>/mount
umount: .../mount: not mounted
~~~

**这证明恢复的元数据已被 kubelet 成功读取，且已进入 CSI 卸载流程。** 原 LV 早已 umount，此时 `mount/` 即使是空目录也不是挂载点；CSI 驱动仍直接执行 `umount` 并将 `not mounted` 当作失败，形成重试循环。

另有 `orphaned pod ... remove the volumes dir: not a directory` 的日志，但出现的 UID 与故障卷不一定相同，不能未经核实就认定为同一根因。

## 本次有效的 Pod 级恢复步骤

### 0. 先核实安全前提（必须）

- 明确故障 Pod UID、CSI volume 目录名、PV 的 `driver` / `volumeHandle` 以及所属节点，核对日志，**不能套用其他正常 Pod 的卷 ID**。
- TiKV 进程/容器已退出；目标 LV 没有被其他 Pod、业务进程或新实例使用。
- 故障 Pod 目录及其下方没有残留真实挂载；尤其要在宿主机和必要时 CSI 插件的挂载命名空间中核查。
- 不修改正常 TiKV 的卷目录；不删除或格式化 LV；操作前退出目标挂载目录（`cd /`）。

示意检查（将占位符替换为实际值）：

~~~bash
POD_UID="<fault-pod-uid>"
VOL="<fault-csi-volume-dir>"
LV="/dev/<vg>/<lv>"

BASE="/var/lib/kubelet/pods/$POD_UID/volumes/kubernetes.io~csi/$VOL"
TARGET="$BASE/mount"

cd /
ls -ld "$BASE" "$TARGET" 2>&1
findmnt -R "/var/lib/kubelet/pods/$POD_UID"
findmnt -M "$TARGET"
findmnt -S "$LV"
lsblk -f "$LV"
~~~

如果发现 LV 仍挂载、已有新 TiKV 占用、或者目标路径仍是原始数据卷的挂载点，**立即停止**，不可执行后续 self bind。

### 1. 恢复准确的 CSI 元数据

先从**同节点、同 CSI 驱动的正常 Pod**读取一份 `vol_data.json`，仅用于确认当前部署实际使用的键名和结构：

~~~bash
cat "/var/lib/kubelet/pods/<normal-pod-uid>/volumes/kubernetes.io~csi/<normal-volume>/vol_data.json"
~~~

本案文件中包含如下字段（示意，值必须属于故障卷）：

~~~json
{
  "attachmentID": "csi-<hash>",
  "driverName": "lvmplugin.csi.alcor.io",
  "nodeName": "<k8s-node-name>",
  "specVolID": "<pod-volume-spec-id>",
  "volumeHandle": "<csi-volume-handle>",
  "volumeLifecycleMode": "Persistent"
}
~~~

`specVolID`、`volumeHandle` 及 CSI 目录名不应在未知时一概假定相同，需根据正常卷模式、Pod/PV 资源及故障日志核实。

本案中，经与正常 TiKV 元数据校验，`attachmentID` 使用如下计算方式（供**当前环境**恢复参考，不能对所有版本无条件套用）：

~~~bash
ATTACH_ID="csi-$(printf '%s%s%s' "$VOLUME_HANDLE" "$DRIVER" "$NODE" | sha256sum | awk '{print $1}')"
~~~

其中 `VOLUME_HANDLE`、`DRIVER`、`NODE` 是**故障卷**真实参数。可同时核对 Kubernetes `VolumeAttachment` 对象；若有现存对象，优先使用其标识并对照。用 `mkdir -p "$BASE"` 建目录、按已核对的字段写入 `"$BASE/vol_data.json"`（建议先写临时文件并原子重命名）。不要机械拷贝正常卷的 JSON。

### 2. 补齐空目录并观察下一轮错误

~~~bash
mkdir -p "$TARGET"
ls -la "$BASE"
journalctl -u kubelet --since "5 minutes ago" --no-pager | grep -F "$POD_UID" | tail -30
~~~

如果出现上述 `umount: .../mount: not mounted`，说明已越过原始元数据读取错误。单纯创建空目录**不会让它变成挂载点**。

### 3. 对空目录执行 self bind mount，等待驱动卸载

**仅在确认目标没有挂载真实 LV、业务已经退出、无其他使用者时**，执行：

~~~bash
# 复核：此时 TARGET 不应该是挂载点
if mountpoint -q "$TARGET"; then
  echo "TARGET is already a mount point; stop"
else
  mount --bind "$TARGET" "$TARGET"
fi

findmnt -M "$TARGET"
journalctl -u kubelet -f | grep --line-buffered -F "$POD_UID"
~~~

机制：`mount --bind "$TARGET" "$TARGET"` 把**空目录绑定到自身**，建立一层临时挂载，而不是重新挂载真正的 LVM 数据卷。CSI 后续重试 `umount "$TARGET"` 时，有机会成功卸载这层 bind mount、返回成功，kubelet 继续进行 Pod/volume 清理。

注意：

- CSI 驱动容器与宿主机的 **mount propagation / mount namespace** 会影响它能否看到宿主机创建的 bind mount；并非所有集群都适用。
- kubelet 可能在创建挂载后很快把它卸载，`findmnt` 未必总能捕获短暂的状态；以对应日志和 Pod 最终状态为准。
- 如果后续仍报错，暂停修改并重新排查；**不能为了消错而对真实 LV 强制卸载**。
- 回退时先确认目标仍挂载的是刚才创建的临时 bind mount，才可以考虑 `umount "$TARGET"`；不可盲目 `umount -l` / `umount -f`。
- 本案例中无需重启 kubelet，也无需手工重新挂载 LV。

## 根因边界与长期修复方向

### 已确认的事实

1. 正常卷的 `vol_data.json` 位于 `<volume>/vol_data.json`，与实际挂载点 `<volume>/mount/` 同级。
2. 故障卷的 CSI 卷目录曾消失，而 kubelet 后续仍发起 CSI 卸载操作。
3. 恢复准确元数据后，kubelet 成功进入 CSI `NodeUnpublishVolume` 对应卸载链路。
4. CSI 驱动执行 `umount` 遇到 `not mounted`（exit status 32）后返回错误。
5. 通过 self bind mount 绕过这一阶段的错误，现场确认 Pod 删除问题已解决。

### 尚待验证的机制

- kubelet 为什么在卷目录清理后仍继续认为卷需要卸载：可能是部分清理失败、状态同步或重复 reconcile；需要结合部署版本、插件实现和前后日志确认。
- `lvmplugin.csi.alcor.io` 的 `NodeUnpublishVolume` 对已卸载 target 的处理是否缺少幂等性。按 CSI 规范，卸载操作应允许重复请求；如果目标已完成卸载，通常应安全返回成功，而不是将 `not mounted` 无限重试作为失败。

### 推荐改进

1. 在 CSI 驱动 `NodeUnpublishVolume` 中，优先判断实际挂载状态；对已经卸载的 target 做幂等成功处理，同时安全清理目录，避免对 `not mounted` 直接失败。
2. 核查 kubelet 与定制 CSI 插件的版本兼容性、卷目录删除顺序及挂载传播配置。
3. 对将来同类故障保留完整时间序列：首次卸载成功/失败、`vol_data.json` 删除、CSI `NodeUnpublishVolume`、Pod volume 状态与重试时间。
4. 将 **self bind mount** 仅作为明确前置条件下的单 Pod 应急办法，不做日常自动化或批量修复；优先从驱动代码修复幂等性。

**关键词**：Kubernetes、TiKV、PV、CSI、LVM、kubelet、Terminating、vol_data.json、UnmountVolume、NodeUnpublishVolume、umount not mounted、self bind mount。
