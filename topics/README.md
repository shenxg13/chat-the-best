# Topics 专题索引

本目录保存可跨会话复用的技术专题、现场故障复盘与环境实践。**按问题找文章，不必先猜英文文件名**；每篇摘要均依据对应正文整理。

- **范围**：仅索引 `topics/` 下的专题文档；工作日志见 [工作日志入口](../summaries/work-log/README.md)。
- **阅读提示**：留意“状态/边界”列，区分现场已验证的操作、阶段性判断和待验证的设计，不要把推测当成最终根因。
- **索引核对日期**：2026-11-10（共 **16** 篇，按主要问题归类；跨领域内容可通过关键词搜索）。

## 按领域浏览

- [数据库、SQL 与备份（6）](#数据库sql-与备份6)
- [Kubernetes、容器与存储（3）](#kubernetes容器与存储3)
- [Nginx、网络与对象存储（3）](#nginx网络与对象存储3)
- [桌面系统、WSL 与 Agent 工具（4）](#桌面系统wsl-与-agent-工具4)

## 数据库、SQL 与备份（6）

| 文章 | 什么时候查 / 核心内容 | 状态 / 边界 |
| --- | --- | --- |
| [SQL Baseline 系统方法论](./sql-baseline.md) | 设计**运行中慢 SQL 的历史基线告警**时查：离线构建与在线检测分离、结构指纹统一、Greenplum/HashData 历史事实来源、统计分位数与防污染、采集与存储选型。 | 方案与方法论；多项数据源和参数待确认 |
| [Oracle 批量 DELETE：全表扫描、AWR 与 HWM](./oracle-batch-delete-full-scan-hwm.md) | 排查 **DELETE 逐条极慢、执行计划全表扫描**时查：`OR` / `IS NULL` 对索引的影响、HWM、AWR 证据、应急止损与长期约束整改。 | 现网案例与整改建议，勿将建议当作已实施 |
| [Oracle RMAN / OSBWS 恢复 S3 备份报 unsupported-algorithm](./oracle-rman-osbws-s3-compatible-aws-chunked.md) | 遇到 `KBHS-00715` / `sbtread2` 恢复失败时查：`Content-Encoding: aws-chunked` 兼容性定位、抓包及 trace、在恢复专用副本移除相关 metadata 的绕过思路。 | 错误机制已定位；原始备份对象不得修改 |
| [TiDB 临时表报 The table is full](./tidb-temporary-table-full.md) | 批处理写入**真正的 TiDB 临时表**失败时查：`tidb_tmp_table_max_size`、SESSION / GLOBAL 范围、如何与算子临时磁盘限额区别。 | 已确认案例；调整前须核实表类型 |
| [Druid 连接池 CommunicationsException](./druid-connection-failure-network-jitter.md) | Druid 报连接通信失败并 `discard` 时查：`poolingCount`、`idleMillis`、`lastValidIdleMillis` 的含义、连接保活与网络排障证据。 | **阶段性判断**：倾向网络抖动，尚未最终定因 |
| [Commvault 磁带全备：Chunk 提交失败、重连差异、时钟漂移与 10 次重启上限](./commvault-tape-backup-chunk-close-retries-2026.md) | 多日全备失败排查：两 Job 的 TapeMark/Chunk 统计、18 次中长等待失败与 2 次超长成功重连；追踪 11-10 TapeMark 线程长时间无日志、CS/MA Chunk 事件时间差从约 25s 漂至 29s、异常换算到 CS 侧每五分钟 :45～46 相位；解释 DM_MONITOR 与 MediaManager 日志无法证明 TCP 收发方向的边界，以及重试预算缓解方案。 | **阶段性排查**：确定 10 次重启耗尽导致整体失败；CS 周期性机制与旧连接失效/重建均属待证假设，SCSI 等待点、网络断链首发方未知 |

## Kubernetes、容器与存储（3）

| 文章 | 什么时候查 / 核心内容 | 状态 / 边界 |
| --- | --- | --- |
| [Kubernetes：CSI/LVM 导致 TiKV Pod 卡在 Terminating](./kubernetes-csi-lvm-pod-unmount-recovery.md) | Pod 删除过程中 `vol_data.json` 缺失、重复 `umount` 且不能重启 kubelet 时查：卷元数据核验、空目录 **self bind mount** 的有限场景恢复、业务数据保护前提。 | **现场恢复成功**；非常规应急手段，严格核实前置条件 |
| [Kubernetes 控制面超时：etcd WAL 延迟与 CBS 告警（2026-11-01）](./kubernetes-etcd-cbs-control-plane-incident-2026-11-01.md) | 多个 Controller Lease 丢失、API Server 超时等**同时出现**时查：etcd WAL `fdatasync` 约 12.28 秒长尾、Raft / ReadIndex 现象、CBS 单副本故障告警时间线与下一步取证。 | **阶段性定位**；CBS 与底层卷 I/O 因果关联待验证 |
| [Docker 镜像离线导出与导入](./docker-image-offline-transfer.md) | 机器间无仓库搬迁镜像时查：`docker save/load`、gzip 压缩、多镜像打包、与 `export/import` 的区别。 | 可复用操作手册 |

## Nginx、网络与对象存储（3）

| 文章 | 什么时候查 / 核心内容 | 状态 / 边界 |
| --- | --- | --- |
| [Nginx 动态 DNS、源 IP、限流及 502/503](./nginx-dns-and-source-ip-troubleshooting.md) | 两层反向代理出现间歇 403/502/503 时查：运行期 `resolver`、`allow/deny` 看到的真实地址、`limit_req`、upstream TCP 建连超时、`curl --resolve` 对照压测。 | 多项故障现象已确认；少量建连异常仍需继续追链路 |
| [Nginx + hosts 下的 HTTPS / SSO 反向代理](./nginx-hosts-sso-reverse-proxy.md) | 浏览器通过 hosts 指向 Nginx、登录跳转另一 SSO 域名后失败时查：443 入口、HTTP 与 HTTPS 边界、`http` 反代和 `stream` TCP 透传、证书与 `proxy_redirect` 限制。 | 原理说明与方案选择 |
| [S3 / COS Request has expired：系统时钟偏差](./s3-request-expired-clock-skew.md) | S3 兼容存储返回 `403 AccessDenied / Request has expired` 时查：签名时间戳、`ServerTime` 与本机时间比较、Windows / Linux 时钟检查。 | 已确认案例；注意 UTC 与本地时区转换 |

## 桌面系统、WSL 与 Agent 工具（4）

| 文章 | 什么时候查 / 核心内容 | 状态 / 边界 |
| --- | --- | --- |
| [Windows + WSL2 + VS Code + Codex 开发环境](./windows-wsl-vscode-codex.md) | 恢复或比较**VS Code Codex Extension、Native Codex 与 Work** 的使用方式时查：WSL 目录与代理、项目路径、会话历史隔离、备份和双入口分工。 | 当前个人环境记录；产品行为可能随版本变化 |
| [Claude 桌面 App + WSL + 手机 Remote Control](./claude-desktop-wsl-remote-control.md) | 需要在 WSL 内用 Claude Code、手机接续和接收推送时查：桌面 App 自带 CLI、WSL 每发行版设置、Remote Control 两种模式及推送开关。 | 桌面接续与手机推送已实测；界面设置可能变化 |
| [Windows 11：Notepad++ 窗口重绘迟滞](./windows-notepadpp-rendering.md) | 最小化后切回 Notepad++ 出现**编辑区或 Theme 延迟重绘**时查：Scintilla 渲染模式 A/B 测试、`DirectWrite (Draw to GDI DC)` 的有效规避配置。 | 当前环境中 workaround 已验证 |
| [银河麒麟 V10 SP1：xrandr 自定义分辨率](./kylin-desktop-xrandr-custom-resolution.md) | 系统设置缺少 `1600×900` 等显示模式时查：X11 下 `xrandr` 添加 modeline、输出端口确认、常见限制。 | 命令可执行；切换效果和重启持久化未确认 |

## 如何维护本索引

1. **新增 `topics/*.md`**：在最贴切的分类新增一行，使用中文问题导向的可点击标题、**适用症状/目标 + 核心结论** 的简述，并标注已验证、阶段性、待确认等状态。
2. **修改已有专题的结论或状态**：同步检查本页对应摘要；尤其当“疑似根因”升级为“已证实”或被推翻时。
3. **不按文件名硬猜内容**：摘要应先阅读正文；没有证据时注明待核实，不把示例配置写成通用最佳实践。
4. **分类不等于目录迁移**：为保持历史链接稳定，不因分类变化改名或移动正文文件；一个文章优先在一个分类出现，可用关键词全文搜索交叉主题。
5. **核对完整性**：维护后核对索引条目数与 `topics/` 实际专题 Markdown 数量一致（不包含本 `README.md`），更新上方日期及计数。
