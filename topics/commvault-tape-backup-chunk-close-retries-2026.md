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
- **时钟校准**：同一 Chunk 的 MA 确认事件与 CS 提交事件在 11-09～10 相差 **28～29 秒**（例如 `8748316`：CS 05:48:01、MA 05:48:29）；**完整 Job 周期存在约 25→29 秒的逐日变化**，详见 5.7 节。跨主机须按日期校正，并先验证两端 NTP；两条日志时间差含响应处理延迟，不能直接当作精确钟差，也不能说“CS 提前 25 秒卸载导致 MA 断连”。

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

## 5. 2026-11-10 增补：TapeMark / write time 与 Chunk 提交结果的批量对照

> **这是一次基于当前可用 MA CVD 及轮转日志的阶段性提取统计，不是磁带机底层 I/O 遥测，也不是完整周期的统一统计。** Job `699224` 仅覆盖 2026-10-20～21 末段，不能称作全周期；Job `700556` 基本覆盖 11-05～10。日志归并以 `Job ID + 备份流/Reservation ID` 为键，将 `Writing TapeMark`、`Size ... write time`、`CloseChunkInfo` 和最终的 `Closed the chunk with Id` / `Failed in Close chunk` 关联。必须按 Chunk 去重：一个正常 Chunk 通常伴随两次 TapeMark；`Successfully closed chunk on Media` **不是** Archive Manager 提交成功。

### 5.1 提取结果及边界

| Job | 本次可分析 TapeMark 行数 | 成功提交 Chunk | 失败提交 Chunk | 说明 |
| --- | ---: | ---: | ---: | --- |
| `699224` | 2,870 | 1,428 | 7 | **只有尾段日志**；1 个成功 Chunk 因截断缺少对应 TapeMark，未纳入按耗时分组 |
| `700556` | 9,920 | 4,946 | 11 | 基本覆盖本次 Job 周期；部分 TapeMark/Chunk 存在未完全配对边界，不能简单用 TapeMark 行数除以 2 得到 Chunk 数 |

按**该 Chunk 的最大 `write time`**分组（不把相邻第二个约 2 秒的 Filemark 单独视作新 Chunk）：

| Job | 最大 `write time` 区间 | Chunk 提交成功 | Chunk 提交失败 |
| --- | --- | ---: | ---: |
| 699224 | ＜300s | 1,427 | 0 |
| 699224 | 300～599s | 0 | 7 |
| 700556 | ＜300s | 4,944 | 0 |
| 700556 | 300～599s | 0 | 11 |
| 700556 | ≥600s | **2** | 0 |

**不能把“超过约 300/360 秒必然失败”当成规律**：Job 700556 存在两次 `write time` 约 **759s、813s** 但最终成功提交的反例。反过来，也不能以此否定“某条旧 TCP 会话曾空闲失效”，因为重连可能恢复了后续的 Archive Manager 提交。

### 5.2 异常样本清单（MA 本地时间）

| Job | 完成/报错时刻 | TapeMark→对应 `Size` 日志间隔 | `write time` | Chunk 结果 |
| --- | --- | ---: | ---: | --- |
| 699224 | 10-20 05:54:48 | 326s | 390s | 失败 |
| 699224 | 10-20 15:09:48 | 328s | 393s | 失败 |
| 699224 | 10-20 18:44:48 | 269s | 334s | 失败 |
| 699224 | 10-20 20:14:48 | 322s | 322s | 失败 |
| 699224 | 10-21 01:34:48 | 268s | 332s | 失败 |
| 699224 | 10-21 03:44:48 | 285s | 350s | 失败 |
| 699224 | 10-21 08:54:48 | 291s | 356s | 失败 |
| 700556 | 11-05 22:50:11 | **693s** | **759s** | **成功，重新连接** |
| 700556 | 11-05 23:15:11 | **746s** | **813s** | **成功，重新连接** |
| 700556 | 11-07 01:45:13 | 426s | 488s | 失败 |
| 700556 | 11-07 08:45:13 | 397s | 461s | 失败 |
| 700556 | 11-07 12:00:13 | 435s | 499s | 失败 |
| 700556 | 11-08 03:25:13 | 388s | 451s | 失败 |
| 700556 | 11-08 10:20:13 | 425s | 488s | 失败 |
| 700556 | 11-08 10:50:13 | 417s | 417s | 失败 |
| 700556 | 11-08 14:15:13 | 410s | 410s | 失败 |
| 700556 | 11-08 20:50:13 | 396s | 458s | 失败 |
| 700556 | 11-09 01:45:13 | 408s | 408s | 失败 |
| 700556 | 11-09 05:55:14 | 342s | 405s | 失败 |
| 700556 | 11-10 02:35:14 | 360s | 424s | 失败 |

正常成功的 TapeMark→`Size` 日志间隔（排除超长反例）中位数约 **2s**；Job 699224 最大约 **24s**，Job 700556 最大约 **23s**。**这些是应用日志时间差，不是已实测的单条 SCSI `WRITE_FILEMARKS` 持续时间。**

### 5.3 关键反例：会话失效后成功重连

两次超长但**成功**的样本（11-05 22:50:11、23:15:11）附近也记录过：
```text
MLMMessageInstance::GetResponseFromCVMLM() - receiveMessage error
Connection has been lost
[DM_MONITOR] Unable to talk to the CS to check validity of reservations
ArMgrMsgHandler::EstablishArchiveManagerConnection()
Shutting down expired Archive Manager Connection
CVArchive::ConnectSessionToArchiveManager()
Attached to CommServe
Closed the chunk with Id ... Got New Chunk Id ... from ArMgr
```

11-05 22:50 样本对应 Chunk `8732060`，该段日志中在 **22:50:11** 感知会话异常后约 2 秒出现对 Archive Manager 的过期连接清理与新建连接，最终取得新 Chunk `8732075`。另一个超长样本有类似序列。**MediaManager 连接失效与 Chunk 提交最终失败不是一一对应**：有重连成功的反例。

与此相对，18 次 300～599s 的失败样本在对应提交前未观察到成功的 Archive Manager 重连，随后 `CVA_CLOSE_CHUNK_110_REQ` 发送 `Broken pipe`。当前解释是**“旧会话已失效，但未被及时判为过期并重建”**的候选机制，而非已确认实现原理。切勿将 600s 当成已确认的 Commvault 内部阈值；更不能仅凭应用日志定位到某台防火墙。

### 5.4 DM_MONITOR 与五分钟刻度

- 本次提取观察到 `DM_MONITOR MonitorThread: Checking ... reservations` **770 条**，`Unable to talk to the CS to check validity of reservations` **14 条**。异常集中于长时间等待附近；两次成功超长样本中也出现过该类通信错误。
- 部分 `Checking` 日志间隔约 10 分钟，**不等于每次都成功获得 CS 响应**。无法从现有文本证明每轮检查使用新连接还是复用长连接；需要连接 ID、握手记录或抓包。
- 所有上述 20 次长时间异常的结束时间都集中在**五分钟刻度附近**：Job 699224 多为 `xx:x4/:x9:48`，Job 700556 多为 `xx:x0/:x5:11～14`。两批相位不同，且 CS、MA 时间需校正约 28～29s；**强烈提示周期性机制值得排查，但不证明其一定由某个 300s 定时器直接触发。**
- CS `ArchMgr.log` 的 Job 700556 Pipeline 参数存在 `jobUpdateInterval[300]`。**这仅是配置字段，尚未证明它控制 TCP 会话过期、TapeMark 操作或任务失败。**

### 5.5 新的工作假设与优先验证

**优先假设：**长时间设备/备份流等待使一个或多个 MA↔CS 会话变得不可用；当 Commvault 尚未把旧 Archive Manager 会话识别为过期时，`CVA_CLOSE_CHUNK_110_REQ` 继续走旧连接而失败；等待更久时反而触发 expired-session 处理，主动重新连接并成功提交。这能解释两个超长成功反例，但目前缺少底层会话状态/精确超时参数证据。

下次取证按优先级：
1. 对**成功重连**（11-05 22:50、23:15）与**失败未重连**（11-09 05:55、11-10 02:35）做同字段、同连接 ID 的 CS `MediaManager`、`ArchMgr`、MA `CVD/CVMA` 对照；锁定创建时刻、最后一次业务消息、检测过期与尝试重连的日志。
2. 核查 `jobUpdateInterval[300]` 对应代码/官方参数说明及 Reservation Monitor 连接模式，排除把“周期性统计日志”误判成“断链原因”。
3. MA/CS **双端抓 TCP**：确认最后一个包、FIN/RST、TCP Keepalive、是否静默丢包、新旧连接的 4-tuple；检查实际网络策略 3600s 是否命中，先校准两端时钟。
4. **并行保留 FC/SCSI 路线**：用磁带机 Drive Dump、设备层 I/O 耗时和 FC HBA/交换机事件验证“等待”真正发生在哪一层。不可仅凭规律性时间点排除磁带机。
5. 在下一次全备前，仍可独立评估**自动重启上限**与单设备流/单 Data Reader 调整作为短期缓解，记录调整结果。不得将缓解手段写成已解决根因。

### 5.6 追踪 11-10 故障线程：最后记录为 Writing TapeMark

对 2026-11-10 **MA 02:29:14～02:35:16** 的 `cvd.log` 按进程/线程过滤（Job `700556`、流 `1699763-4849878`）：

| MA 时间 | PID / 日志线程 ID | 关键记录 |
| --- | --- | --- |
| 02:29:14 | `3975 / 2426` | `[MEDIAFS] Writing TapeMark 220` |
| 02:29:14～02:35:14 | 同线程 | **应用日志中无其他该线程记录** |
| 02:35:14 | `3975 / 2426` | `MLMMessageInstance::GetResponseFromCVMLM()` 报 `Connection has been lost` |
| 02:35:14 | `3975 / 2426` | `write time=424`、`Successfully closed chunk on Media`，随后 `Writing TapeMark 221` |
| 02:35:16 | `3975 / 2426` | 第二次 TapeMark 记录 `write time=2`，随后 `CVA_CLOSE_CHUNK_110_REQ` 发送 `Broken pipe`、`Failed in Close chunk` |

**观测结论：**关键六分钟空档之前，该线程最后一条可见日志是 `Writing TapeMark 220`，并无“已先向 CS 发起请求并等待”的可见记录。其后出现通信异常，更吻合“磁带收尾操作附近长时间未返回，然后发现连接已失效”的路径。

**证据边界：**应用层日志沉默不能**证明**线程一直阻塞在 SCSI `write` 或 `WRITE_FILEMARKS` 系统调用；进程也可能在该日志语句之后进入其他未打点的同步逻辑。必须靠现场线程栈、`strace`（谨慎）、内核设备跟踪或 IBM Drive Dump 明确等待点。

`CVMA.log` 来自另一进程（PID `3977`），**不能跨进程按相同 TID 机械拼接成同一线程**。其网络读取线程 `70ea` 在约 **02:35:14** 也发现到 CS MediaManager 的通信失效；MA CVD 的监控线程 `1da2` 同秒报错，形成跨执行上下文旁证。

### 5.7 同一 Chunk 的 CS/MA 时间差：逐日漂移，不能固定使用 28～29 秒

以 `ArchMgr.log` 的 `ArMgrCS::closeChunk Committed a chunk for archChunkId [...]` 与 MA CVD 中同一 Chunk 的 `Closed the chunk with Id [...]` 对照。先前分析报告为 Job `700556` 匹配到约 **4,946 个成功 Chunk**，逐日时间差中位数约如下（MA 记录时间减去 CS 记录时间）：

| 日期 | 11-05 | 11-06 | 11-07 | 11-08 | 11-09 | 11-10 |
| --- | ---: | ---: | ---: | ---: | ---: | ---: |
| 日志事件差中位数（秒） | 25 | 26 | 27 | 28 | 29 | 29 |

具有可复核 Chunk ID 的代表性对照：

| 日期 | Chunk | CS ArchMgr 提交时间 | MA CVD 确认时间 | 事件差 |
| --- | --- | --- | --- | ---: |
| 11-05 | `8732060` | 22:49:48 | 22:50:13 | 25s |
| 11-07 | `8736259` | 01:36:38 | 01:37:05 | 27s |
| 11-10 | `8750776` | 02:27:41 | 02:28:10 | 29s |

因此，之前文档里“时间差约 28～29 秒”**只适用于后几天，不适用于全周期**。这是**同一 Chunk 的两次应用日志事件差值**，包含网络与客户端接收/处理时延，并非 NTP 直接测出的绝对系统时钟偏差；逐日中位数来自本轮分析，建议后续保留提取脚本和匹配清单复验。

以当日典型差值将 **MA 异常结束时间**换算到 CS 参考时间：

| 事件 | MA 异常时间 | 使用当日差值 | 估算 CS 参考时间 |
| --- | --- | ---: | --- |
| 11-05 超长但重连成功 | 22:50:11 | 25s | 22:49:46 |
| 11-07 提交失败 | 01:45:13 | 27s | 01:44:46 |
| 11-10 提交失败 | 02:35:14 | 29s | 02:34:45 |

**新发现：**原先 MA 侧异常相位从 `:11` 漂到 `:14`，换算后却聚在 CS 侧五分钟网格的约 `:45～46`。这使 **CS 侧或与 CS 时钟对齐的周期性机制**更值得优先检验。但**不能仅凭相位关联就证明定时器在 CS，更不能直接归罪于 `jobUpdateInterval[300]`**：尚需排除网络设备、其他 CommCell 定时过程以及对照匹配的计算误差。

辅助验证：比较 CS 与 MA 进程启动时刻、Job 创建时刻在 300s 网格上的相位；对照不同 Job 的同相位事件；同时以 `chronyc tracking` / `w32tm /query /status` 核实两端实际时钟。必须留意 `ps -o lstart -p ...` **只表示当前存活进程**，若故障时 PID 对应的进程已经重启，其结果不能代表历史进程启动时间。

### 5.8 DM_MONITOR 与 CS MediaManager 请求核对：日志能判定什么

**DM_MONITOR：**11-10 MA 日志：
```text
02:20:26 [DM_MONITOR] Checking [1] reservations for validity
02:30:26 [DM_MONITOR] Checking [1] reservations for validity
02:35:14 [DM_MONITOR] Unable to talk to the CS to check validity of reservations
```
部分检查间隔约 **10 分钟**，与异常在 `:x0/:x5:14` 的五分钟相位**不直接相同**。不同事件“上一次 Checking→报错”的时间也不恒定：11-07 12:00:13 为约 74s，11-08 03:25:13 为约 278s，11-10 02:35:14 为约 288s。现有日志**不能证明**单次检查一直等待这些秒数，不能确定 DM_MONITOR 用短连接还是长连接，更不足以称它为 300s 失败定时器。

**CS MediaManager：**对照 11-10：
- CS `02:27:41`：`ArchMgr` 提交 `8750776`。
- CS `02:28:59`、`02:33:59`：`MediaManager` 有线程池统计，未见整体线程池堆积证据。
- CS `02:34:49`：Job `700556` 的 `LOGICAL UNMOUNT`；它不等于物理卸载磁带。
- MA `02:35:16`：发送 `CVA_CLOSE_CHUNK_110_REQ` 报 `Broken pipe`，按近 29s 的**事件差**估算 CS 约 `02:34:47`；注意该换算不是精确的网络包时间。
- CS `MediaManager.log` 在 TapeMark 刚开始后的约 `02:28:45` **没有明确记录收到 CloseChunk 请求**，这**不构成网络丢包证据**：正常 Chunk 也未必在该日志逐条打印该请求，而且故障请求实际发生在 TapeMark 之后，而不是刚开始时。

所以当前无法根据“日志中查不到请求”来判定“请求未到 / 回复丢失 / CS 收到但没回复”三分支中的哪一个；**只有带连接 ID 的详细会话记录或 MA/CS 双端 TCP 包才能分流**。客户端 `Broken pipe` 只证明发包时连接已不可用，并不能确定最初由网络设备、CS 进程还是客户端关闭。

### 5.9 本轮调查对下一步优先级的修正

1. **第一优先：用 CS 参考时间检查每五分钟 `:45～46` 的周期事件**，将 JobManager、MediaManager、Archive Manager 的日志连同其他成功 Job 一起对照，检验 `jobUpdateInterval[300]` 究竟对应什么动作。
2. **同时：查清 MA `Writing TapeMark` 后的真实执行点**，以同 PID/TID 对比正常约 2 秒样本与异常数分钟样本；仅凭最终 `Connection has been lost` 不能将磁带设备 I/O 排除。
3. **优先对比异常后重连成功与未重连成功**的实际连接生命周期：旧会话创建/最后活跃/识别过期/关闭/新会话建立时间，尤其是 11-05 两次成功反例与 11-10 失败。
4. **必要时双端抓包**：判定 RST/FIN 首发方、TCP 重传与最后有效负载、是否复用坏会话及是否存在中间设备清理；若需复现应优先使用可控的低风险测试窗口。
5. 原有短期保障措施仍有效：评估重启次数上限、单 Device Stream 下将 Data Readers 由 2 降为 1、错峰 Index Backup；**所有措施均为待实施/验证的缓解方案而非已修复**。

## 6. 当前判断（明确证据等级）

| 议题 | 状态 |
| --- | --- |
| 单次 Chunk 元数据提交失败、MA↔CS MediaManager 会话失效 | **已确认现象** |
| 失败阶段可被 JobManager 自动重启，累计 10 次触顶导致整体 Job Failed | **已确认机制/直接终止条件** |
| TapeMark 周围长时间等待（约 5～6 分钟） | **已确认时间关联**，精确 SCSI 命令耗时未知；另有两次 >600s 成功重连反例，详见第 5 节 |
| CS Chunk 提交至 MA 确认的事件时间差 | **逐日约 25→29 秒**（同一 Chunk 对照）；不等于精确 NTP 钟差。异常换算到 CS 参考时刻约每五分钟的 :45～46，周期性来源待验证 |
| FC/SCSI 历史异常 | **已确认历史记录**，尚未证明与 11 月同源 |
| 300 秒防火墙空闲超时是唯一根因 | **已被变更后复现否定** |
| Archive Manager 整体卡死、逻辑卸载先触发故障、Index Backup 抢占造成故障 | **当前证据不支持** |
| 是防火墙仍清理会话、Commvault 内部会话管理，还是磁带/SCSI 等待先引发问题 | **待判定根因**；重点验证 expired-session 重建、CS 参考时间的 :45～46 周期事件和 TapeMark 的实际等待点 |

## 7. 下一步：短期保障与根因调查并行

### A. 保障下次长时间全备可完成（缓解而非根治）

1. 在正式重跑前确认 Data Readers 调为 1 是否合适，记录原值；将自动 Index Backup 尽可能错开主备份窗口。
2. **评估提高“Job 自动重启最大次数”**：当前 10 次阈值导致前期有效进度最终作废。可在该 Job/子客户端的 Advanced→Job Retry（若当前 v11 界面支持）或 `Control Panel → Job Management → Job Restarts` 查找对应设置。初步考虑针对本次长任务提高到约 20 次，**需先确认该版本该 Job 类型确实支持以及作用范围**，优先任务级、避免影响全局，不应当作普遍最佳实践。
3. 核实 Restart Point 是否使重启**真正续传**以及重复处理的量、磁带消耗、恢复可用性；提交和校验失败的末尾 Chunk 不应被视为有效数据。不要因提高重试次数就放弃根因分析。
4. 故障 Job 不会因后续调参自动变为成功；开始新 Job 前留存事件、日志、磁带条码和介质关联信息。

### B. 根因取证优先顺序

> **2026-11-10 优先级更新：先按第 5.6～5.9 节追踪异常线程、逐日校准 CS/MA 事件差、排查 CS 参考时间 :45～46 的周期机制，并对比成功重连与提交失败样本；设备层和网络层仍需并行取证。下列历史任务清单保留作为补充。**

1. **确定磁带侧等待在哪里**：在维护窗口读取 IBM `ULT3580-TD6` drive dump、TapeAlert/Log Sense、FC HBA 与交换机的端口事件；暂不执行影响介质的写入、擦除、固件更新。优先对照 11-09 05:49～05:56 和 11-10 02:29～02:36。
2. **FC 计数器采样比较增量**：对 `host1/statistics` 在空闲及写入期间重复取样，同时以 WWPN `5001438028caae98` 查交换机侧 Link/CRC/RSCN/端口重登录事件；当前累计数本身不能关联失败时段。
3. **网络双端抓包复现**：在 MA 和 CS 抓取真实 MA↔MediaManager TCP 会话，核实 300→3600 秒变更实际命中情况、是否出现 RST/FIN、静默丢弃和哪个方向先终止；注意 `10.2.3.238` 与 `40.1.18.94` 两种日志所见 CS 地址可能涉及不同路径/NAT，不能据此直接断言。
4. **校时**：分别查 Linux `chronyc tracking`、Windows `w32tm /query /status` 和当前时钟偏差；以校准后的时间线对照 TapeMark、MediaManager、JobManager 与抓包。
5. 必要时用一盘明确可覆盖的 Scratch 带、在无其他备份任务时做厂商支持的非生产隔离验证；不使用带有有效备份数据的介质。

## 8. 继续排查的最小上下文

- 待关联的三个标识：Job `700556`，故障 Chunk `8748317`／`8750777`，资源预约 `4849877`／`4849878`。
- 需保留的核心日志：MA `CVD.log`、`CVMA.log`，CS `MediaManager.log`、`JobManager.log`、`ArchMgr.log`、`CVD.log`，以及磁带设备与 FC 诊断信息。
- **关键思路修正**：真正导致整个多日全备最终失败的是“偶发可恢复错误累计 10 次”，不是首次报 `40:39`；缓解可提高重试预算。第 5 节新发现 **超长等待 + 连接失效但最终成功重连** 的反例，并进一步发现 **MA 线程在 TapeMark 之后长时间无日志、MA/CS 事件时间差逐日 25→29 秒、异常按 CS 参考时间落在 :45～46 周期附近**。重点由“单一固定写入超时”调整为 **会话复用/过期识别/重连及 CS 周期性机制的时序验证**，同时保留磁带/SCSI 等待的因果验证。
