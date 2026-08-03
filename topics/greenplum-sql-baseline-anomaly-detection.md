# Greenplum SQL Baseline 与运行中异常 SQL 检测

## 目标

建设一套面向 Greenplum 的 SQL 性能基线与异常检测能力，重点不是事后统计已经完成的慢 SQL，而是发现：

- SQL 仍在执行中；
- 当前运行时长已经明显超过同类 SQL 的历史正常范围；
- 能基于历史分布量化“异常到什么程度”；
- 能进一步提供偏斜、等待、spill 等诊断上下文。

现阶段不依赖现网 AI Agent，系统本身应能完成确定性的发现、量化、分类、告警和诊断数据输出。

## 已确认的总体思路

采用“两路数据 + 分层分析”：

```text
历史已完成 SQL
  -> 结构化历史数据源
  -> SQL 结构指纹
  -> 历史耗时分布/基线

当前执行中 SQL
  -> pg_stat_activity（实时快照）
  -> 同一套结构指纹
  -> 与历史基线比较
  -> 异常事件
```

历史数据用于学习正常分布；`pg_stat_activity` 用于发现尚未结束的异常 SQL。单纯解析 `gpdb-*.csv` 的 duration 日志不适合作为“运行中异常 SQL”的主方案，因为 SQL 未结束时并没有最终 duration。

## Greenplum 原生能力边界

### 当前执行

优先使用 `pg_stat_activity` 获取当前 active SQL 及 `query_start`，实时计算已运行时长。

如果启用了 gpperfmon，也可以将 `queries_now` 作为补充，但它受采集周期影响，不如 `pg_stat_activity` 实时。

### 历史执行明细

可优先考虑 gpperfmon / Greenplum Command Center 的结构化历史查询数据（如 `queries_history`），比解析 CSV 日志更适合作为历史基线数据源。

CSV 日志仍可作为错误信息、底层诊断和补充数据源，但不应成为运行中 SQL 检测的唯一来源。

### SQL 指纹

GP 能标识一次具体执行（例如 gpperfmon 中常见的 `tmid + ssid + ccnt`），但这不是“同类 SQL”的结构指纹。

GP6 文档中的 `query_hash` 曾存在但未实现，不能假定其可直接作为稳定的 SQL 结构指纹。当前实际 GP 版本尚未确认，后续必须先执行 `SELECT version();` 再验证该版本的原生能力。

## 采集层原则：尽量不自研

希望把自研集中在 SQL 指纹、历史聚合和异常分析，不把精力放在日志 tail、断点续传、轮询重试等外围采集问题上。

推荐：

- 历史结构化数据：Logstash JDBC Input 持续/增量读取；
- 当前执行快照：Logstash JDBC Input 周期查询 `pg_stat_activity`；
- 如果必须采集 CSV：Filebeat 负责读取和续传，Logstash 负责结构化解析；
- 不建议自写脚本完成文件 tail 和采集链路。

Logstash JDBC Input 是合适的“无自研轮询组件”；Logstash 直接写 PostgreSQL/MySQL 的官方能力较弱，社区 JDBC output 已长期停更。若现场已有 Kafka，可考虑 Kafka Connect JDBC Sink；否则第一版可单独评估更简洁的落库链路。

## 数据存储与监控分工

不把 SQL 明细直接塞进 Prometheus。

推荐职责：

```text
PostgreSQL / TimescaleDB：
  SQL 执行明细
  SQL 指纹字典
  历史基线
  异常事件
  标准差/P95/P99 等统计分析

Prometheus：
  当前异常 SQL 数量
  最长运行时间
  锁等待数量
  分析服务延迟/健康状态

Grafana：
  统一展示 Prometheus 汇总指标 + PostgreSQL 明细

Alertmanager：
  告警
```

Prometheus 自带的是 Prometheus TSDB，不是 TimescaleDB。Prometheus 适合低基数指标，不适合以 `sql_md5`、`execution_id`、完整 SQL 文本等高基数内容作为主要存储模型。

第一版普通 PostgreSQL 即可，数据规模和保留周期明显增大后再评估 TimescaleDB 或 ClickHouse。

## 自研核心：SQL Analysis Service

第一版建议单服务实现，不必一开始拆很多微服务：

```text
sql-analysis-service
├── fingerprint-worker
├── baseline-worker
├── running-query-detector
├── anomaly-event-writer
└── prometheus-exporter
```

核心自研集中在：

1. SQL AST 解析与结构指纹；
2. 历史耗时分布与基线聚合；
3. 当前执行 SQL 与历史基线匹配；
4. 异常判定规则；
5. Prometheus 低基数指标输出。

## AST 与 SQL 指纹

AST（Abstract Syntax Tree，抽象语法树）用于把 SQL 从文本转换为结构化语法对象。

例如：

```sql
SELECT * FROM users WHERE id = 100;
SELECT * FROM users WHERE id = 200;
```

常量不同，但结构相同。通过 AST 可以忽略文本格式和常量差异，形成结构指纹。

Greenplum/PostgreSQL 方言优先考虑 `libpg_query` / `pg_query_go`，因为它复用 PostgreSQL 解析器，适合做：

- SQL Parse / AST；
- Normalize；
- Fingerprint；
- 后续基于 AST 提取表、JOIN、子查询、谓词等结构特征。

指纹结果建议保存版本信息，而不是只保存一个 MD5：

```text
fingerprint
fingerprint_version
normalized_sql
parser_name
parser_version
```

这样以后升级指纹算法，可以从原始 SQL 重新计算而不破坏历史。

## 重要限制：参数归一化会掩盖数据偏斜

不能假设“结构相同 = 性能分布相同”。

例如：

```sql
SELECT * FROM orders WHERE tenant_id = 1;
SELECT * FROM orders WHERE tenant_id = 9999;
```

归一化后都变成：

```sql
SELECT * FROM orders WHERE tenant_id = ?;
```

但如果 `tenant_id=1` 是热点值或数据严重偏斜，两次执行时间可能差几个数量级。

因此结构指纹只负责“归类”，不能被当成完美的性能画像。

## 第一版不要全量获取执行计划

每天几十万条 SQL 时，绝不能对每条执行反查 `EXPLAIN`。即使只是规划，也会给 Coordinator 增加明显负担，并存在临时表、session 参数、权限、统计信息变化等重放问题。

所以执行计划与 estimated rows 不应进入全量第一层采集。

采用分层策略：

### 第一层：全量

所有 SQL 都只做低成本处理：

```text
structural_fingerprint
execution duration
status
database/application
query_start
skew_cpu / skew_rows（若数据源已有）
```

按结构指纹统计：

- count；
- avg；
- stddev；
- P50；
- P95；
- P99；
- max；
- CV = stddev / avg。

### 第二层：只分析高离散指纹

通过以下信号筛选值得进一步分析的 SQL 类别：

- 样本数足够；
- CV 很高；
- P95 / P50 很高；
- P99 / P50 很高；
- 异常长 SQL 高频出现。

只有这些高离散指纹才进一步分析参数特征，例如：

- IN 列表长度；
- 日期范围；
- LIKE 类型；
- 等值条件参数类型/热度（后续可选）。

这类很多特征可直接从 AST / 原始 SQL 获取，不需要回库。

### 第三层：按需计划分析

只有以下对象再考虑获取执行计划或 estimated rows：

- 当前正在异常运行的 SQL；
- 高频高离散指纹；
- 同指纹近期突然整体变慢；
- DBA 明确标记的重点 SQL。

并应做缓存、限流、低峰执行、只读限制等保护。

执行计划属于“异常后的诊断信息”，不是全量基线字段。

## 基线与异常判定

不要只使用“平均值 + 3σ”，因为 SQL 耗时往往是长尾分布。

建议至少同时保留：

- P50；
- P95；
- P99；
- stddev；
- CV；
- max。

第一版异常规则可以采用组合判断：

```text
minimum_runtime
AND sample_count >= N
AND (
  current_runtime > historical_p99
  OR current_runtime > historical_p95 * K
  OR current_runtime > avg + 3 * stddev
)
```

对于没有历史基线或样本不足的新 SQL，回退到固定时长阈值。

同一结构指纹内部如果 P95/P50、P99/P50 或 CV 很高，本身就是一个重要信号：可能存在冷热参数、数据偏斜、不同计划或资源环境差异。第一版不必立刻解决，只需要先识别这些“高离散指纹”。

## 运行中 SQL 与历史基线

当前执行 SQL 每隔约 10～30 秒采样即可：

```text
pg_stat_activity
  -> 结构指纹
  -> 查询基线
  -> 当前 runtime / P99 / stddev
  -> 异常事件
```

一次具体执行的 `execution_id` 与结构指纹要分开：

- `structural_fingerprint`：表示同一类 SQL；
- `execution_id`：表示当前这一具体执行实例，可由 session/pid/query_start 等组合生成。

同一条长 SQL 会被连续采样多次，异常事件必须基于 `execution_id` 去重并更新 `last_detected_at`，不能每次采样都生成新事件。

## 数据模型建议

至少分三层：

```text
raw_query_history
raw_running_query_snapshot

sql_fingerprint_dictionary
sql_execution

sql_baseline
sql_anomaly_event
sql_hourly_statistics
```

原始层保留完整输入，便于将来升级指纹算法后重放；处理层保存指纹及结构化执行记录；结果层保存基线和异常事件。

## AI 的位置

现网目前不具备部署 AI Agent 的条件，因此第一阶段不依赖 AI。

现网系统应做到：

```text
能发现
能量化
能分类
能回溯
能告警
能导出诊断包
```

后续如果允许，可将异常事件和诊断包脱敏后送到隔离区/办公环境中的 AI 做解释和排查建议。

原则是：

> 统计和规则负责判断异常，AI 负责解释与诊断；不要让 LLM 直接消费每天几十万条 SQL 流，也不要让 LLM 决定基础告警是否成立。

## 当前待确认项

1. 现网 Greenplum 的确切版本尚未确认，不能继续默认 GP6；先执行：

```sql
SELECT version();
```

2. 根据实际版本确认：
   - `pg_stat_activity` 具体字段；
   - gpperfmon / GPCC 是否已启用；
   - `queries_history`、`queries_now` 的实际结构；
   - `query_hash` 是否可用；
   - `pg_stat_statements` 是否可用及能力边界。

3. 明确历史数据主采集源优先级：gpperfmon/GPCC 结构化历史优先，CSV 日志作为补充。

4. 选择最终落库链路，目标仍是“采集尽量用成熟组件，自研集中在指纹与分析”。
