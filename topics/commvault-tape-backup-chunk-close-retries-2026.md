# Commvault 磁带全备：Chunk 关闭异常、会话失效与重试上限（2026-10～11）

> **状态：阶段性排查，根因未确认。** 最近核对：2026-11-10。案例涉及 Commvault 11、Linux MediaAgent 兼客户端、IBM LTO-6 FC 磁带库。本文保留后续接续排查所需的证据、已纠正判断和操作边界，不把相关性写成因果结论。

## 1. 现象与影响

- Windows CommServe：`cvcommserve`；Linux MediaAgent/客户端：`HL_HHY_HL10BURP1MA2002`（MA 地址 `10.160.49.83`）。主备份子客户端 `nas_cos_1`，存储策略 `SP-PTL-HHY-HL`，Linux File System 完全备份，目标 IBM TL3573 磁带库、`ULT3580-TD6` 驱动器。没有启用去重。
- **第一次**：Job `699224`，运行约 99 小时，写入超过 20 TB，期间多次自动重启，累计达到上限后失败。
- **第二次**：Job `700556`，2026-11-05 启动，11-07 至 11-10 多次相同异常，累计 10 次自动重启后以 `19:1253` 结束；两次均非“首次 Chunk 报错就立刻终止全任务”。
- 常见故障链：`6:22`（无法关闭元数据文件）／`40:39`（无法向 CS Archive Manager 更新 Chunk 信息）／MA `Connection has been lost`、`Broken pipe`、`Failed to Update Archive Manager Tables`。**一轮 Backup 阶段失败，Commvault 可以重新进入 Backup 阶段继续推进**；最终失败的直接原因是重启次数耗尽，底层偶发故障仍需解决。
- 一次长时间全备涉及多盒磁带。后续若要回收失败任务介质，应通过 Job/数据老化流程核对各盘是否含其他有效作业；不要未经确认直接 Erase/Delete Contents。

## 2. 已确认的关键信息和时间线

### 网络策略变更后仍复现

- 防火墙 TCP 空闲超时原为 **300 秒**，2026-11-04 已改为 **3600 秒**；新 Job 700556 在变更后才启动，却仍复现故障。故 **“300 秒防火墙超时是唯一根因”不成立**。
- 仍需核实实际 MA↔CS 连接命中的策略、可能的 NAT/其他安全设备，以及既有连接与新建连接适用规则；不能仅凭已修改配置就证明每条真实 TCP 会话都得到 3600 秒。

### MA CVD 中两次典型事件

| MA 时间（2026） | 上一 Chunk 正常提交 | 开始记录 TapeMark | 读 MediaManager 响应失败 | 异常耗时 | 未成功提交到 CS 的 Chunk |
| --- | --- | --- | --- | --- | --- |
| 11-09 | 05:48:29 | 05:49:32（TapeMark 1242） | 05:55:14 | 日志 `write time=405` 秒；TapeMark 两条记录相差约 342 秒 | `8748317`，Afile `799622`，VolumeId `58228` |
| 11-10 | 02:28:10 | 02:29:14（TapeMark 220） | 02:35:14 | `write time=424` 秒；TapeMark 两条记录相差约 360 秒 | `8750777`，Afile `799623`，VolumeId `58226` |

- 上述时间跨度**不是已证实的单条 SCSI WRITE FILEMARKS 命令耗时**；`write time` 也可能包含较大范围写入/设备等待。MA 日志表明磁带侧随后报告 `Successfully closed chunk on Media`，但向 CS 发 `CVA_CLOSE_CHUNK_110_REQ` 时 `ErrNo.32 Broken pipe`。
- 11-10 02:35:14：`DM_MONITOR` 同时报 `Unable to talk to the CS to check validity of reservations`，说明不止一处单独的 Chunk 提交调用感知到 CS 通信失效。
- Linux MA 通过 `/dev/tape/by-id/...-nst`（`st` 原生磁带驱动路径）操作磁带；勿未经确认就用 Pass-through 的 `ScsiTimeouts` 参数解释或修改实际 SCSI 超时。

### CS 端日志相互印证

- `ArchMgr.log`：11-09 05:48:01 提交 `8748316`、11-10 02:27:41 提交 `8750776`；没有记录后续故障 Chunk `8748317`／`8750777` 的成功提交。其他 Job 的 Chunk 仍持续提交，**未见 Archive Manager 整体停摆证据**，但不排除某个请求/线程/会话的问题。
- `JobManager.log`：11-09 **05:54:48** 收到 Job 700556 `Failed to update the chunk information to the archive manager on the CommServe` 和连接相关失败；**05:55:13** 记录 `Phase [7-Backup] ... Failed. Backup will continue with phase [Backup]`。即 Commvault 可自动重试，并非单次错误必然全任务失败。
- `MediaManager.log`：11-09 **05:54:50**、11-10 **02:34:49** 收到 `LOGICAL UNMOUNT:1, PHYSICAL UNMOUNT:0, RELEASE VOLUME:0, RELEASE DRIVE:0`；事件并未要求物理卸载/释放驱动器。**CS 同机 JobManager 的故障记录先于该逻辑卸载**，所以不应把 `LOGICAL UNMOUNT` 当作首发根因。
- `CVD(1).log`（Windows CS）：对应窗口未见足以独立解释故障的明确 CVD 错误；须注意 CS 的 CVD.exe 与 MediaManager.exe 是不同进程，前者正常不等于后者 TCP 会话正常。
- **时钟校准**：同一成功 Chunk 的 CS 与 MA 时间分别相差 **28～29 秒**（例如 `8748316`：CS 05:48:01、MA 05:48:29）。跨主机日志排列事件必须先核查时钟/NTP 偏差，不能直接说“CS 提前 25 秒卸载导致 MA 断连”。

## 3. 资源配置问题：与主故障分开处理

- 存储策略 `SP-PTL-HHY-HL` 的 **Device Streams=1**；子客户端 `nas_cos_1` 的 **Data Readers=2**、Network Agents=1；“允许一个（源）驱动器或装载点内有多个数据读取器”未勾选。后者指源端文件系统挂载点，不等于允许两份作业共用同一磁带机。
- JobManager 中 `279 There are not enough drives in the drive pool` 的大量记录 **Job ID 是 700556 本身**，表明主备份也反复申请额外 Stream；不能全部归因于另一个 Index Backup Job。推荐将该子客户端 **Data Readers 暂调为 1**，以匹配单 Device Stream，作为下一次运行前消除资源噪声的配置修正；记录调整前后值并验证新任务实际资源使用。
- 自动 `IndexBackup` 确实会与仅有一台可用驱动器的存储策略争用资源，但在已对照的 **11-10** 故障中，Index Backup 类型的作业是在主备份资源释放后才获分配并成功提交，**无证据证明它抢占驱动器导致 Chunk 失败**。可考虑错峰，不宜长期禁用索引保护。

## 4. Linux 内核 / FC 现场证据

- 11-09 05:48～05:57、11-10 02:28～02:38 的 `journalctl -k` 均未发现记录；`/var/log/messages` 对应时段没有观察到 SCSI reset/timeout/I/O error。无内核异常不等于无阻塞，可能是命令最终成功返回或日志缺失。
- **历史但不是本次直接证据**：
  - 2026-05-25：内核 `INFO: task cvd ... blocked for more than 120 seconds`，栈包含 `st_write → st_do_scsi → wait_for_completion`，说明曾在磁带 SCSI 写入路径长时间阻塞。
  - 2026-10-12：`blocked FC remote port time out`、`rejecting I/O to offline device`，随后重新识别 IBM 磁带机；10-14 有设备 reset 提示。
- `host1` 当前 `port_state=Online`，`speed=8 Gbit`，WWPN `5001438028caae98`；累计 `link_failure_count=0x5`（5）、`nos_count=0x3`（3），`loss_of_signal_count`、`loss_of_sync_count`、`invalid_crc_count`、`invalid_tx_word_count`、`error_frames` 均为 0。**累计计数没有故障日期**，不能认定 11 月新增；部分 `0xffffffffffffffff` 指标表示不支持/不可用，不得当作大量错误。
- 现场命令（只读）：
  ```bash
  cat /sys/class/fc_host/host1/port_state
  cat /sys/class/fc_host/host1/speed
  cat /sys/class/fc_host/host1/port_name
  grep -H . /sys/class/fc_host/host1/statistics/* 2>/dev/null
  journalctl -k --since "2026-11-10 02:28:00" --until "2026-11-10 02:38:00" --no-pager
  ```

## 5. 当前判断（明确证据等级）

| 议题 | 状态 |
| --- | --- |
| 单次 Chunk 元数据提交失败、MA↔CS MediaManager 会话失效 | **已确认现象** |
| 失败阶段可被 JobManager 自动重启，累计 10 次触顶导致整体 Job Failed | **已确认机制/直接终止条件** |
| TapeMark 周围长时间等待（约 5～6 分钟） | **已确认时间关联**，精确 SCSI 命令耗时未知 |
| MA/CS 时钟差约 28～29 秒 | **多事件交叉验证**；需现场核实时间同步 |
| FC/SCSI 历史异常 | **已确认历史记录**，尚未证明与 11 月同源 |
| 300 秒防火墙空闲超时是唯一根因 | **已被变更后复现否定** |
| Archive Manager 整体卡死、逻辑卸载先触发故障、Index Backup 抢占造成故障 | **当前证据不支持** |
| 是防火墙仍清理会话、Commvault 内部会话管理，还是磁带/SCSI 等待先引发问题 | **待判定根因** |

## 6. 下一步：短期保障与根因调查并行

### A. 保障下次长时间全备可完成（缓解而非根治）

1. 在正式重跑前确认 Data Readers 调为 1 是否合适，记录原值；将自动 Index Backup 尽可能错开主备份窗口。
2. **评估提高“Job 自动重启最大次数”**：当前 10 次阈值导致前期有效进度最终作废。可在该 Job/子客户端的 Advanced→Job Retry（若当前 v11 界面支持）或 `Control Panel → Job Management → Job Restarts` 查找对应设置。初步考虑针对本次长任务提高到约 20 次，**需先确认该版本该 Job 类型确实支持以及作用范围**，优先任务级、避免影响全局，不应当作普遍最佳实践。
3. 核实 Restart Point 是否使重启**真正续传**以及重复处理的量、磁带消耗、恢复可用性；提交和校验失败的末尾 Chunk 不应被视为有效数据。不要因提高重试次数就放弃根因分析。
4. 故障 Job 不会因后续调参自动变为成功；开始新 Job 前留存事件、日志、磁带条码和介质关联信息。

### B. 根因取证优先顺序

1. **确定磁带侧等待在哪里**：在维护窗口读取 IBM `ULT3580-TD6` drive dump、TapeAlert/Log Sense、FC HBA 与交换机的端口事件；暂不执行影响介质的写入、擦除、固件更新。优先对照 11-09 05:49～05:56 和 11-10 02:29～02:36。
2. **FC 计数器采样比较增量**：对 `host1/statistics` 在空闲及写入期间重复取样，同时以 WWPN `5001438028caae98` 查交换机侧 Link/CRC/RSCN/端口重登录事件；当前累计数本身不能关联失败时段。
3. **网络双端抓包复现**：在 MA 和 CS 抓取真实 MA↔MediaManager TCP 会话，核实 300→3600 秒变更实际命中情况、是否出现 RST/FIN、静默丢弃和哪个方向先终止；注意 `10.2.3.238` 与 `40.1.18.94` 两种日志所见 CS 地址可能涉及不同路径/NAT，不能据此直接断言。
4. **校时**：分别查 Linux `chronyc tracking`、Windows `w32tm /query /status` 和当前时钟偏差；以校准后的时间线对照 TapeMark、MediaManager、JobManager 与抓包。
5. 必要时用一盘明确可覆盖的 Scratch 带、在无其他备份任务时做厂商支持的非生产隔离验证；不使用带有有效备份数据的介质。

## 7. 继续排查的最小上下文

- 待关联的三个标识：Job `700556`，故障 Chunk `8748317`／`8750777`，资源预约 `4849877`／`4849878`。
- 需保留的核心日志：MA `CVD.log`、`CVMA.log`，CS `MediaManager.log`、`JobManager.log`、`ArchMgr.log`、`CVD.log`，以及磁带设备与 FC 诊断信息。
- **关键思路修正**：真正导致整个多日全备最终失败的是“偶发可恢复错误累计 10 次”，不是首次报 `40:39`；缓解可提高重试预算，但根因仍聚焦在 TapeMark 周围耗时和 MA↔CS 会话生命周期。
