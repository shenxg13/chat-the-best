# SQL Baseline 系统方法论

## 目标与边界

SQL Baseline 系统用于回答两个不同问题：

1. **历史上某类 SQL 正常需要运行多久？**
2. **当前仍在运行的 SQL，是否已经明显超过其历史正常完成时间？**

当前设计明确不依赖 AI，重点放在数据采集、SQL 指纹、历史聚合、统计模型、运行中异常检测和诊断数据输出。

系统不追求随每次 SQL 执行实时更新 Baseline。推荐将“历史基线构建”和“在线异常检测”拆成两个平面：

```text
历史已完成 SQL
  -> 结构化历史数据源
  -> SQL 结构指纹
  -> 历史耗时分布 / Baseline

当前执行中 SQL
  -> activity/session 视图
  -> 同一套结构指纹
  -> 与已发布 Baseline 比较
  -> 异常事件
```

当前主要目标不是发现“已经结束后才知道它慢”，而是尽早发现：

> 一条 SQL 历史上通常早已完成，但当前仍然运行。

---

## 核心架构决策

### 1. Baseline 离线构建，按天更新

Baseline 本质上是统计模型，不需要随每次 SQL 执行实时更新。

建议训练窗口初始采用最近 14～30 天，Baseline 每天生成一个新版本。

推荐流程：

```text
生产 Greenplum / HashData
  ↓
已完成 SQL 的结构化历史数据
  ↓ 成熟采集组件持续或批量抽取
离线 / 独立分析库
  ↓
SQL 结构指纹
  ↓
执行事实表
  ↓
训练数据清洗
  ↓
Baseline 聚合
  ↓
版本化发布
```

生产数据库本身不承担 SQL 指纹解析、分位数计算、Baseline 构建和长期历史统计等工作。

### 2. 在线异常检测只关注“还没跑完”的 SQL

实时检测与 Baseline 构建分离。

```text
activity/session 视图
  ↓ 每 10～30 秒采样
当前仍运行 SQL
  ↓
使用和离线完全相同的 SQL 指纹算法
  ↓
匹配 ACTIVE Baseline
  ↓
比较 elapsed time 与历史完成时长分布
```

在线检测优先关注：

- LONG_RUNNING_EXECUTION
- LOCK_WAIT
- RESOURCE_QUEUE / RESOURCE_GROUP WAIT
- 后续可扩展 Motion / Segment / Interconnect / Spill 等诊断维度

### 3. 离线和在线必须使用同一套 SQL 指纹算法

这是系统正确性的核心约束。

如果离线历史和实时 activity 对同一条 SQL 生成不同 fingerprint，则实时检测无法可靠匹配 Baseline。

SQL 指纹组件应：

- 作为公共库或公共服务复用；
- 具有 `normalizer_version` / `fingerprint_version`；
- 支持 PostgreSQL / Greenplum / HashData 语法适配；
- 对无法解析的 SQL 提供文本归一化降级路径；
- 保留原始 SQL，使算法升级后可以重新计算历史指纹。

---

## Greenplum / HashData 原生数据源结论

### 当前执行 SQL

优先使用 `pg_stat_activity` 或当前版本等价 activity/session 视图获取当前 active SQL、`query_start` 和等待状态，实时计算已运行时长。

如果启用了 gpperfmon，可将 `queries_now` 作为补充，但它受采集周期影响，不应默认比 activity 视图更实时。

### 历史执行明细

历史 Baseline 的数据源优先级调整为：

1. **gpperfmon / Greenplum Command Center 的结构化历史查询数据**（如实际版本提供 `queries_history`）；
2. Coordinator 已轮转 CSV 日志；
3. `gp_log_database` 等日志外部表仅作为调查、补充或一次性验证手段。

结构化历史表比 CSV 更适合作为主数据源，因为已经包含查询文本、状态和时间字段，不需要自行处理 CSV 中 SQL 的逗号、引号、多行和日志事件拼装。

CSV 日志仍有价值：

- 补充错误信息；
- 底层故障诊断；
- 在 gpperfmon / GPCC 不可用时作为历史来源；
- 保留可重放的原始证据。

### `gp_log_database` 不适合作为长期高频采集入口

经典 Greenplum 的 `gp_toolkit.gp_log_database` 基于日志外部表，底层会读取 Coordinator / Segment 日志目录中的 CSV 文件。时间条件通常只能减少返回和上层处理的数据，并不等价于像索引那样直接定位日志偏移。

因此不建议把它作为长期、高频、准实时的 SQL 明细采集入口。

### SQL 指纹原生能力

Greenplum 能标识一次具体执行，但这和“同类 SQL 的结构指纹”是两个概念。

例如 gpperfmon 常见的：

```text
tmid + ssid + ccnt
```

可以用于标识一次具体查询执行，但不能用于把“结构相同、参数不同”的 SQL 聚为同一类。

GP6 文档中的 `query_hash` 曾存在但未实现，因此不能假定现网可以直接使用原生 SQL 结构指纹。

**当前现网 Greenplum 的确切版本尚未确认。后续不能继续默认 GP6，必须先执行 `SELECT version();`，再针对实际版本验证 `query_hash`、`pg_stat_statements`、gpperfmon / GPCC 和 activity 视图能力。**

---

## 采集层原则：尽量不自研

希望把自研集中在真正有业务价值的部分：

- SQL 指纹；
- 历史数据聚合；
- Baseline 构建；
- 标准差 / 分位数分析；
- 当前运行 SQL 与历史基线匹配；
- 异常判定和诊断上下文。

不把精力放在：

- 文件 tail；
- 读取位置管理；
- 断点续传；
- JDBC 轮询重试；
- 日志轮转跟踪；
- 通用消息传输。

推荐使用成熟组件完成采集：

```text
历史结构化数据：
  Logstash JDBC Input / Kafka Connect JDBC Source 等

当前执行快照：
  Logstash JDBC Input 周期查询 pg_stat_activity

如果必须读取 CSV：
  Filebeat 负责文件读取与续传
  Logstash 负责结构化解析
```

### 关于 Filebeat + Greenplum CSV

Greenplum CSV 的 SQL 字段可能包含逗号、双引号和换行，所以不能把 Filebeat 当作 CSV 解析器。Filebeat 只负责可靠读取和转发，CSV 解析和事件组装必须在后续组件中完成，并需要先用真实日志验证跨物理行记录是否会被错误拆分。

### 关于 Logstash 落 PostgreSQL / MySQL

Logstash 官方有成熟 JDBC Input，但没有同等级官方 JDBC Output。长期生产链路不宜依赖长期停更的社区 JDBC output。

如果现场已有 Kafka，可考虑：

```text
Logstash / Filebeat -> Kafka -> Kafka Connect JDBC Sink -> PostgreSQL
```

如果没有 Kafka，不应为了第一版强行引入整套 Kafka；应单独选择更简单、可维护的落库链路。

---

## 数据存储与监控分工

### PostgreSQL / TimescaleDB

用于存储和分析：

- SQL 执行明细；
- SQL 指纹字典；
- 历史 Baseline；
- 异常事件；
- P50 / P95 / P99 / 标准差 / CV 等统计数据。

第一版普通 PostgreSQL 即可。数据量和保留周期明显增大后，再评估 TimescaleDB 或 ClickHouse。

### Prometheus

Prometheus 自带的是自己的 TSDB，不是 TimescaleDB。

Prometheus 只保存低基数汇总指标，例如：

```text
gp_sql_anomaly_current{cluster,database,severity}
gp_sql_longest_running_seconds{cluster,database}
gp_sql_lock_wait_total{cluster,database}
gp_sql_analysis_lag_seconds{cluster}
```

不要把这些字段作为 Prometheus 主要标签：

- `sql_fingerprint`
- `execution_id`
- `pid`
- 完整 SQL 文本

否则会产生高基数问题。

### Grafana / Alertmanager

Grafana 同时连接 Prometheus 和 PostgreSQL：

- Prometheus：总览、趋势、告警指标；
- PostgreSQL：异常 SQL 明细、历史分布、指纹详情。

Alertmanager 负责告警，不承载明细分析。

---

## 自研核心：SQL Analysis Service

第一版建议做成单一分析服务，而不是一开始拆成很多微服务：

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
2. 历史耗时分布与 Baseline 聚合；
3. 当前执行 SQL 与历史 Baseline 匹配；
4. 异常判定规则；
5. Prometheus 低基数指标输出；
6. 高离散指纹的进一步特征分析。

---

## AST 与 SQL 指纹

AST（Abstract Syntax Tree，抽象语法树）把 SQL 从文本字符串转换为结构化语法对象。

例如：

```sql
SELECT * FROM users WHERE id = 100;
SELECT * FROM users WHERE id = 200;
```

常量不同，但结构相同。通过 AST 可以忽略文本格式和常量差异，形成结构指纹。

Greenplum / PostgreSQL 方言优先考虑 `libpg_query` / `pg_query_go` / pglast 等 PostgreSQL parser 生态，用于：

- Parse / AST；
- Normalize；
- Fingerprint；
- 后续提取表、JOIN、子查询、谓词等结构特征。

指纹结果不要只保存一个 MD5，建议保存：

```text
fingerprint
fingerprint_version
normalized_sql
parser_name
parser_version
```

这样以后升级指纹算法，可以从原始 SQL 重新计算，而不破坏历史可追溯性。

---

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

因此结构指纹只负责**归类**，不能被当成完美的性能画像。

这也是为什么 Baseline 不能只保存平均值和标准差，而要保留完整分布特征，并识别“同一指纹内部离散度很高”的情况。

---

## 第一版不要全量获取执行计划

每天几十万条 SQL 时，绝不能对每条执行反查 `EXPLAIN`。

即使只是规划，也会增加 Coordinator 解析和优化开销，并存在以下重放问题：

- 临时表；
- session 参数；
- 搜索路径；
- 权限；
- 事务上下文；
- 参数类型和值缺失；
- 统计信息已经变化；
- 原 SQL 为写操作。

因此 `plan_fingerprint` 和 `estimated_rows` 不进入全量第一层采集。

采用三级分析策略。

### 第一层：全量低成本特征

所有 SQL 都只做：

```text
structural_fingerprint
execution_duration
status
database / application
query_start
skew_cpu / skew_rows（若原数据源已有）
```

按结构指纹统计：

- sample_count；
- avg；
- stddev；
- P50；
- P75；
- P90；
- P95；
- P99；
- max；
- CV = stddev / avg；
- active_days。

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
- 等值参数的类型 / 热度；
- 其他可从 AST 和原始 SQL 直接提取的特征。

这些分析优先在离线分析侧完成，不需要逐条回生产库。

### 第三层：按需执行计划分析

只针对以下对象再考虑获取计划或 estimated rows：

- 当前正在异常运行的 SQL；
- 高频高离散指纹；
- 同指纹近期突然整体变慢；
- DBA 明确标记的重点 SQL。

并应做：

- 缓存；
- 限流；
- 低峰执行；
- 只读限制；
- 超时保护；
- 同一指纹 / 参数画像避免重复分析。

执行计划属于**异常后的诊断信息**，不是全量 Baseline 字段。

---

## Baseline 推荐统计模型

SQL 执行时长通常具有偏态、长尾和参数敏感特征，因此不能只依赖平均值和标准差。

第一版建议保存：

```text
sample_count
active_days
avg_duration
stddev_duration
cv
p50
p75
p90
p95
p99
max_duration
median_log_duration
mad_log_duration
```

实时运行 SQL 的阈值可以采用“分位数 + 倍数 + 绝对时间”联合规则，例如：

```text
warning_threshold = max(P95, P50 × 3, absolute_min)
high_threshold    = max(P99, P50 × 5, absolute_min_high)
```

或第一版采用：

```text
minimum_runtime
AND sample_count >= N
AND (
  current_runtime > historical_p99
  OR current_runtime > historical_p95 * K
  OR current_runtime > avg + 3 * stddev
)
```

对于没有历史 Baseline 或样本不足的新 SQL，回退到固定时长阈值。

同一结构指纹内部如果 `P95/P50`、`P99/P50` 或 CV 很高，本身就是一个重要信号：可能存在冷热参数、数据偏斜、不同计划或不同资源环境。第一版不必立即把这些群体拆开，只需要先识别“高离散指纹”。

---

## Baseline 防污染

异常执行如果直接进入训练集，会逐步把性能退化学习成“新正常”。

因此 Baseline 训练必须区分训练状态，不能简单把所有 SUCCESS 执行纳入。

推荐训练状态：

```text
ELIGIBLE
EXCLUDED_FAILED
EXCLUDED_CANCELLED
EXCLUDED_RUNTIME_ANOMALY
EXCLUDED_CLUSTER_INCIDENT
EXCLUDED_STATISTICAL_OUTLIER
PENDING_DRIFT_REVIEW
ACCEPTED_NEW_NORMAL
```

最低限度排除：

- FAILED；
- CANCELLED；
- STATEMENT_TIMEOUT；
- 在线检测已经标记为异常的执行；
- 集群故障或维护窗口；
- 明显缺失或损坏的执行记录。

旧 Baseline 应用于判断新数据能否进入训练集：

```text
Baseline Vn
  ↓
筛选新一天的正常训练样本
  ↓
生成 Baseline Vn+1
```

不能先用当天全部数据更新 Baseline，再用更新后的 Baseline 判断当天是否正常。

偶发异常应隔离而不是删除。若慢执行连续多天稳定存在，并且不是故障或排队导致，则可进入 `PENDING_DRIFT_REVIEW`，作为新正常候选。

---

## Baseline 版本管理

不建议直接 UPDATE 覆盖当前 Baseline。

推荐状态：

```text
BUILDING
READY
ACTIVE
RETIRED
FAILED
```

每个版本记录：

```text
version_id
training_start
training_end
created_at
activated_at
normalizer_version
fingerprint_version
baseline_algorithm_version
```

生成完成并通过完整性检查后，原子切换 ACTIVE 版本。

这样可以：

- 追踪性能 Baseline 漂移；
- 回答某次实时检测引用了哪一版 Baseline；
- 算法升级后重建历史；
- 必要时回滚上一版 Baseline。

---

## 运行中 SQL 与历史 Baseline

当前执行 SQL 每隔约 10～30 秒采样即可：

```text
pg_stat_activity / 等价 activity 视图
  -> structural_fingerprint
  -> 查询 ACTIVE Baseline
  -> current_runtime / P95 / P99 / stddev
  -> anomaly event
```

一次具体执行的 `execution_id` 与结构指纹必须分开：

- `structural_fingerprint`：表示同一类 SQL；
- `execution_id`：表示当前这一具体执行实例，可由 session / pid / query_start 等组合生成。

同一条长 SQL 会被连续采样多次，异常事件必须基于 `execution_id` 去重并更新 `last_detected_at`，不能每次采样都生成新事件。

---

## 数据分层

推荐数据层：

```text
raw_query_history
    原始历史执行输入，保持可重放

raw_running_query_snapshot
    当前运行 SQL 的采样快照

ingest_file / ingest_batch
    文件或批次登记、校验、处理状态

sql_fingerprint_dictionary
    normalized SQL、fingerprint 和 parser 元数据

sql_execution
    已完成 SQL 执行事实

sql_daily_stats / sql_hourly_statistics
    中间统计聚合

sql_baseline_version
    Baseline 版本

sql_baseline
    每个 fingerprint 的统计模型

baseline_training_rejection
    被排除训练的执行及原因

sql_anomaly_event
    当前运行异常事件
```

原始输入建议保留足够长时间，使 SQL 指纹或解析算法升级后可以重新 Replay，而不需要重新访问生产环境。

---

## AI 的位置

现网目前不具备部署 AI Agent 的条件，因此第一阶段不依赖 AI。

现网系统自身应做到：

```text
能发现
能量化
能分类
能回溯
能告警
能导出诊断包
```

后续如果条件允许，可以把异常事件和诊断包脱敏后送到隔离区 / 办公网中的 AI 做解释和排查建议。

原则：

> 统计和规则负责判断异常，AI 负责解释与诊断。

不要让 LLM 直接消费每天几十万条 SQL 流，也不要让 LLM 决定基础告警是否成立。

---

## 推荐 MVP 技术栈

当前建议调整为：

```text
历史采集：优先 Logstash JDBC Input / 其他成熟 JDBC 增量组件
CSV 兜底：Filebeat + Logstash
当前 SQL：Logstash JDBC Input 周期读取 activity 视图
分析服务：Python 或 Go
SQL parser：libpg_query 生态 / pglast / pg_query_go
分析数据库：PostgreSQL 17（第一版）
展示：Grafana
指标与告警：Prometheus + Alertmanager
版本管理：Git
```

第一阶段不必强行引入 Kafka、TimescaleDB、ClickHouse、Spark、Flink、Airflow。只有现场已经有 Kafka 或后续数据规模 / 解耦要求明显增加时再引入。

---

## 第一阶段实施顺序

1. **确认现网 Greenplum 真实版本。**
   ```sql
   SELECT version();
   ```
2. 确认 gpperfmon / GPCC 是否启用，以及 `queries_history` / `queries_now` 的实际结构和数据覆盖情况。
3. 确认 activity 视图当前能提供的 SQL 文本、query_start、等待状态、session 标识等字段。
4. 确定历史主数据源：优先结构化历史表；CSV 作为补充 / 兜底。
5. 用成熟组件搭建历史增量采集和当前 activity 周期采样，不先自研采集器。
6. 建立原始历史表和运行中快照表。
7. 实现统一 AST / SQL fingerprint，并加入 fingerprint 版本管理。
8. 建立 `sql_execution` 和 `sql_fingerprint_dictionary`。
9. 生成每个 fingerprint 的 P50 / P95 / P99 / stddev / CV 等基础统计。
10. 识别高离散指纹，验证参数归一化造成的性能混群程度。
11. 增加训练样本排除逻辑和 Baseline 版本化构建 / 发布。
12. 接入运行中 SQL 异常检测器。
13. 最后再按需增加偏斜、spill、锁等待、执行计划等诊断上下文。

第一版暂不做：

- 全量参数敏感自动聚类；
- 全量执行计划 Baseline；
- 全量 estimated rows；
- SQL 对象依赖分析；
- 工作日 / 周末和小时级模型；
- 自动接受 Baseline 漂移；
- 现网 AI Agent。

---

## 当前待确认项

1. 现网 Greenplum 的确切版本。
2. gpperfmon / GPCC 是否启用，以及历史 SQL 是否覆盖足够完整。
3. `queries_history` / `queries_now` / activity 视图的实际字段。
4. `query_hash` 是否可用，`pg_stat_statements` 是否可用及能力边界。
5. 最终历史落库链路：目标仍是“采集尽量用成熟组件，自研集中在 SQL 指纹与分析”。
6. 高离散 SQL 的实际比例，以及仅使用结构指纹 + 分位数是否已经足以覆盖第一版在线异常检测。
