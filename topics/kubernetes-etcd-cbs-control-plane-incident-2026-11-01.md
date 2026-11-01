# Kubernetes 控制面集体超时与 etcd WAL 长尾延迟（2026-11-01，疑似关联 CBS 存储故障）

> **状态（2026-11-01）**：已定位到 etcd Leader 侧一次约 12.28 秒的 WAL `fdatasync` 长尾延迟，并观测到 Raft 心跳 / ReadIndex 异常、API Server 请求超时、多个 Controller 因 Leader Lease 续约失败退出。**CBS 存储池硬盘/副本故障与该事件在时间和机制上高度关联，但底层因果仍待核对存储卷映射及 I/O 记录。** 此文为阶段性排障记录，供后续追加，不代表最终根因报告。

## 1. 场景与部署（脱敏）

- 环境：灾备 Kubernetes 集群，三台 Master（本文称 **Master-1 / Master-2 / Master-3**，分别对应现场的节点尾号 1001 / 1002 / 1003）；各部署一套 kube-apiserver 与 etcd，etcd 为三成员堆叠式部署。
- Kubernetes 内置 `kubernetes` Service 的 ClusterIP:443 转发到三台 Master 的 API Server:6443。各底座控制器通过 ClusterIP 访问；kube-controller-manager / kube-scheduler 的现场日志则通过本机 `127.0.0.1:6443` 访问 API Server。
- 本次 Master-3 上的 kube-controller-manager 和 kube-scheduler 是活动 Leader，故障时因续约超时主动退出；Master-1 / Master-2 上同类非活动实例未见同一时间的重启。
- 三台 kube-apiserver 未见本次重启，三台 etcd 未见本次容器重启。**不重启不等于服务无超时。**
- 为避免将生产网络/资产详情提交到公开仓库，此文不记录 Master 内网 IP、CBS 资产 IP、完整 Pod/主机命名及原始截图。

## 2. 事件时间线（北京时间 UTC+8）

| 时间（2026-11-01） | 证据与含义 |
| --- | --- |
| **约 05:16:50** | 按 Master-2 etcd 一次 WAL `fdatasync` 的完成时刻及约 12.28 秒耗时倒推，写入操作可能在此时段开始被阻塞；其他 etcd 成员出现 `waiting for ReadIndex response took too long, retrying`。开始时刻为推算，不是独立采样记录。 |
| **05:16:55 左右** | Master-3 etcd 出现 `lost leader`、`became pre-candidate at term 5`、`MsgPreVote`。 |
| **05:16:55～05:17:00** | Master-3 API Server 调用本地 etcd Client 端点发生 `DeadlineExceeded`；Lease / Node 更新出现约 10 秒的 etcd 存储访问耗时、HTTP Handler timeout。 |
| **约 05:17:00** | Master-3 kube-controller-manager、kube-scheduler 均因访问本机 API Server 的 Lease 续约请求超时而 `leader election lost`、退出。 |
| **约 05:17:02** | Master-2 etcd 记录 **`slow fdatasync`，约 12.28 秒**，以及 `leader failed to send out heartbeat on time; ... likely from slow disk`。Master-3 恢复识别原 Leader。 |
| **05:17:01～05:17:03** | CSI Controller、存储/平台相关 Controller 等分布于不同节点的组件，访问 API Server ClusterIP 时 Lease 续约失败、Leader 丢失、随后重启。 |
| **05:18** | CBS 平台通报“存储池单副本故障达到 **1350**，连续三次达到告警阈值”；处置描述为**硬盘故障，待变更更换**；告警后来恢复。05:18 为告警触发时间，不一定是首次故障时间。 |
| **05:20:14** | 监控告警：API Server 对 `GET namespaces` 请求的 P99 延迟达到 **29.875 秒**；采集/评估窗口可导致告警晚于实际异常。 |

## 3. 已证实的技术事实

1. **控制面请求大面积超时**：多个不同节点上的底座 Controller 在同一分钟因 Leader Lease 续约失败退出，不能简单归因为单个 CSI/业务插件缺陷。
2. **两种 API 访问路径均受影响**：底座 Controller 经 Kubernetes Service ClusterIP 请求超时；核心 kube-controller-manager / kube-scheduler 使用 Master-3 的 loopback API Server 地址仍超时，因此**单纯的 Service 转发故障不足以解释全部现象**。
3. **API Server 对 etcd 请求超时**：API Server 日志显示 `etcd-client`、`rpc error: code = DeadlineExceeded`、`GuaranteedUpdate etcd3` 约 10 秒及 Handler timeout，涉及 Lease 和 Node 等资源。
4. **etcd 一致性读取/Leader 识别异常**：Master-3 记录 ReadIndex 等待过长、短暂 `lost leader` 和 PreVote，随后识别到**相同 Leader ID、相同 term 5**；这并不等于已证明整个集群正式更换 Leader。
5. **etcd WAL 出现单次严重长尾延迟**：Master-2 的 `wal/wal.go` 日志包含约 **12.28s `slow fdatasync`**，并伴随 Leader 心跳发送超时告警。根据当时 Raft 日志，Master-2 高度疑似为该时刻实际 Leader，仍可用成员状态指标进一步核实。
6. **Prometheus 的 WAL P99 并未显示相应秒级峰值**：此前在 2 分钟窗口查询 `etcd_disk_wal_fsync_duration_seconds_bucket`，各实例 P99 大致为数毫秒；**少量 12 秒级极端值可能被滚动窗口内大量正常样本掩盖**，不能据此否定明确的单次慢 fsync 日志。
7. **Leader change 指标的解释**：在 `increase(etcd_server_leader_changes_seen_total[5m])` 上，Master-3 出现约一次变化，另外两台未同步呈现；05:17～05:22 的 5 分钟平台源于滚动窗口，不能理解为异常持续 5 分钟。
8. **同时有 TLS 证书告警**：etcd 日志中故障前后存在多条 `rejected connection: tls: bad certificate`，但尚未证明这些是 Peer 连接，亦未证明与本次卡顿有因果关系，暂作为独立核查线索。

## 4. 阶段性故障链（高置信部分与待验证部分分开）

**已观测到的链路**：

```text
Master-2 etcd WAL fdatasync 长尾延迟（约 12.28s）
    + Raft Leader heartbeat 发送超时
                ↓
etcd ReadIndex 等待/部分成员短暂失去 Leader 识别
                ↓
API Server 向 etcd 发起的资源请求耗时过长/超时
                ↓
各 Controller 无法在租约期限内完成 Leader Lease 续约
                ↓
核心 controller-manager / scheduler 及多个底座 Controller 主动退出、由编排系统重启
```

**最可能但尚未证实的上游诱因**：

```text
CBS 存储池硬盘/副本异常（05:18 正式告警）
             ?
    Master-2 etcd WAL 所在存储发生 I/O 阻塞或尾延迟
```

关键待证点是 CBS 故障池 **是否承载 Master-2 的 etcd 数据盘**，以及相同故障窗口内是否存在 CBS 层写延迟/重试/副本重构。另一种仍需排除的解释是 Master-2 宿主机的 CPU 调度长时间停顿、虚拟化宿主机 I/O 抖动或 Raft 通信相关异常。**`slow fdatasync` 是 etcd 观测到的时间消耗，不单独构成“硬盘坏道导致”的证明。**

## 5. CBS 告警解读和影响边界

- CBS 通报指出单副本故障数达到 **1350**、连续三次越阈、告警恢复、后续拟更换故障硬盘；**1350 的指标单位尚未确认**，不能理解为 1350 块磁盘故障或 1350 次 I/O 超时。
- 告警时间晚于 Pod 退出一分钟不妨碍关联：需核实 CBS 采集/评估周期、首次异常时间和硬盘故障的时间戳。
- 通报写“无业务影响”，但 Kubernetes **控制面可用性已被日志证明受到短暂影响**；运行中的业务 Pod/业务交易是否受影响另行评估，不应在未检查之前直接声称业务完全中断或完全无影响。

## 6. 后续需补证据（继续调查时优先）

1. **存储映射：最高优先级。** 确认 Master-2 etcd 数据目录、宿主机文件系统和底层块设备，以及该设备是否由报障 CBS 存储池承载；若 Master 为虚拟机，进一步核对虚拟磁盘与物理存储池映射。
2. **CBS 时间与 I/O 证据。** 向存储团队获取 05:15～05:20 的故障磁盘事件、写入延迟 P99/Max、队列长度、I/O 重试或超时、存储节点负载、副本切换/重建；确认告警评估周期和“1350”含义。
3. **Master-2 宿主机历史指标与日志。** 检查相同窗口块设备写延迟/最大延迟、iowait、CPU 抢占和资源争用、内核 I/O timeout/reset/hung-task、虚拟化宿主机存储性能。当前 `iostat` 不能替代历史数据。
4. **另两台 etcd 日志对照。** 尤其 Master-1 是否有相同 ReadIndex/heartbeat/慢 fsync；核实故障当时 Raft Leader 成员和异常是否为 1002 局部停顿。
5. **Peer TLS 告警独立核查。** 根据远端端口/etcd 配置确认 `bad certificate` 出现在客户端 2379 还是 Peer 2380，评估是否为无关健康探测/失效证书。
6. **影响范围与恢复指标。** 追踪故障窗口 API Server 请求延迟、错误率、Controller 恢复、集群调度是否暂时中断，以及工作负载是否实际受影响。

### 可复用只读检查（使用通用占位符）

```bash
# 查看 etcd Pod 数据盘挂载，避免假定 /var/lib/etcd 一定在本地盘
kubectl -n kube-system get pod <etcd-pod> -o jsonpath='{range .spec.volumes[*]}{.name}{"  "}{.hostPath.path}{"\n"}{end}'

# 在对应 Master 宿主机核对实际 etcd 路径
findmnt -T /var/lib/etcd
lsblk -o NAME,TYPE,SIZE,FSTYPE,MOUNTPOINT

# 宿主机时间需要确认时区；以下示例按北京时间
journalctl -k --since "2026-11-01 05:15:00" --until "2026-11-01 05:20:00" --no-pager

# etcd 单节点日志关键字
kubectl -n kube-system logs <etcd-pod> --timestamps \
  --since-time="2026-10-31T21:15:00Z" | \
  grep -Ei 'fdatasync|heartbeat|ReadIndex|leader|fsync|slow|timed out'
```

Prometheus 可复用的检查方向：

```promql
increase(etcd_server_leader_changes_seen_total[5m])
```

```promql
histogram_quantile(0.99, sum by (instance, le) (rate(etcd_disk_wal_fsync_duration_seconds_bucket[2m])))
```

```promql
etcd_server_is_leader
```

```promql
etcd_server_has_leader
```

注意：滚动分位数不能代替最大值/异常日志；窗口长度与 Prometheus 抓取间隔决定能否看到几秒级停顿。

## 7. 本阶段判断（后续继续更新）

- **直接故障路径基本查明**：etcd 存储调用与 Raft 协调出现十秒级阻塞，API Server 超时，导致多个使用 Lease 选主的核心与底座组件集中失去 Leader 并重启。
- **最强底层假设**：CBS 同期硬盘/副本故障造成 etcd Leader 所在盘 I/O 长尾异常。时间、机制、日志相互符合，但要以**存储映射 + 故障窗口 CBS I/O**两项补证作为正式归因条件。
- **处置原则**：故障已经恢复时，优先保留日志、监控与 CBS 侧证据；未验证前不应贸然重启 etcd、执行 defrag/compact 或修改 Raft 参数。
- **续接提示**：下一次讨论可从“1002 Master etcd WAL 约 12.28 秒慢 fsync + 05:18 CBS 单副本故障 1350”切入，并更新本文件的存储映射、底层 I/O、最终影响及根因状态。
