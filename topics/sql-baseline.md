# SQL Baseline 系统方法论

## 目标与边界

SQL Baseline 系统用于回答两个不同问题：

1. **历史上某类 SQL 正常需要运行多久？**
2. **当前仍在运行的 SQL，是否已经明显超过其历史正常完成时间？**

当前设计明确不引入 AI，重点放在数据模型、采集流程、统计模型和异常检测。

系统不追求实时更新 Baseline，也不要求实时处理所有已完成 SQL。推荐将“历史基线构建”和“在线异常检测”拆成两个平面。

---

## 核心架构决策

### 1. Baseline 离线构建，按天更新

Baseline 本质上是统计模型，不需要随每次 SQL 执行实时更新。

推荐流程：

```text
生产数据库
  ↓
已轮转、已关闭的历史日志
  ↓ 每日复制
离线分析环境
  ↓
解析已完成 SQL
  ↓
SQL 指纹归一化
  ↓
执行事实表
  ↓
训练数据清洗
  ↓
构建新版 Baseline
  ↓
版本化发布
```

建议训练窗口初始采用最近 14～30 天，Baseline 每天生成一个新版本。

生产数据库本身不承担 SQL 指纹解析、分位数计算、Baseline 构建和历史统计等工作。

### 2. 在线异常检测只关注“还没跑完”的 SQL

实时异常检测与 Baseline 构建分离。

推荐通过数据库的 activity/session 类视图低频采样当前运行 SQL：

```text
activity 视图
  ↓ 每 10～30 秒采样
当前仍运行 SQL
  ↓
使用和离线完全相同的 SQL 指纹算法
  ↓
匹配已发布 Baseline
  ↓
比较 elapsed time 与历史完成时长分布
```

当前主要检测目标不是“已经完成但执行偏慢”，而是：

> 一条 SQL 历史上通常早已完成，但当前仍然运行。

因此在线检测更关注：

- LONG_RUNNING_EXECUTION
- LOCK_WAIT
- RESOURCE_QUEUE_WAIT
- 后续可扩展 Motion / Segment / Interconnect 等等待类型

### 3. 离线和在线必须使用同一套 SQL 指纹算法

这是系统正确性的核心约束。

如果离线日志和实时 activity 对同一条 SQL 生成不同 fingerprint，则实时检测无法可靠匹配 Baseline。

SQL 指纹组件应：

- 作为公共库或公共服务复用；
- 具有 `normalizer_version` / `fingerprint_version`；
- 支持 PostgreSQL/Greenplum/HashData 语法适配；
- 对无法解析的 SQL 提供文本归一化降级路径。

---

## Baseline 推荐统计模型

SQL 执行时长通常具有偏态、长尾和参数敏感特征，因此不能只依赖平均值和标准差。

第一版建议保存：

```text
sample_count
active_days
p50
p75
p90
p95
p99
median_log_duration
mad_log_duration
```

实时运行 SQL 的阈值可以采用“分位数 + 倍数 + 绝对时间”联合规则，例如：

```text
warning_threshold = max(P95, P50 × 3, absolute_min)
high_threshold    = max(P99, P50 × 5, absolute_min_high)
```

避免毫秒级 SQL 因相对倍数过大产生无意义告警。

---

## Baseline 防污染

异常执行如果直接进入训练集，会逐步把性能退化学习成“新正常”。

因此 Baseline 训练必须区分训练状态，不能简单地把所有 SUCCESS 执行纳入。

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

最低限度需要排除：

- FAILED
- CANCELLED
- STATEMENT_TIMEOUT
- 在线检测已经标记为异常的执行
- 集群故障或维护窗口
- 明显缺失或损坏的执行记录

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

推荐：

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
baseline_algorithm_version
```

生成完成并通过完整性检查后，原子切换 ACTIVE 版本。

这样可以：

- 追踪性能基线漂移；
- 回答某次实时检测引用了哪一版 Baseline；
- 算法升级后重建历史；
- 必要时回滚上一版 Baseline。

---

## 数据分层

推荐最小数据层：

```text
raw_log
    原始历史日志文件，保持不可变

ingest_file
    文件登记、校验、处理状态

sql_fingerprint
    normalized SQL 和 fingerprint 元数据

sql_execution
    已完成 SQL 执行事实

sql_daily_stats
    每日统计聚合

sql_baseline_version
    Baseline 版本

sql_baseline
    每个 fingerprint 的统计模型

baseline_training_rejection
    被排除训练的执行及原因
```

原始日志建议保留足够长时间，使 SQL 指纹或解析算法升级后可以重新 Replay，而不需要重新访问生产环境。

---

## 组件选型与自研边界

### 可以直接复用的现成组件

| 能力 | 推荐组件 |
|---|---|
| 已轮转日志复制 | rsync / rclone / scp / SFTP |
| 日志压缩 | gzip / zstd |
| 文件完整性校验 | SHA-256 |
| CSV 解析 | Python `csv` / Go `encoding/csv` |
| PostgreSQL SQL parser | libpg_query 生态 / pglast / pg_query_go |
| 数据存储 | PostgreSQL |
| 分位数计算 | PostgreSQL `percentile_cont` 等 |
| 定时任务 | systemd timer / cron |
| 展示 | Grafana |

第一版不需要 Kafka、Filebeat、Logstash、Airflow、Spark、Flink、ClickHouse、TimescaleDB。

如果未来规模和任务复杂度显著增加，再逐步引入。

### 必须自研的领域逻辑

1. **数据库日志适配器**
   - 识别实际日志字段和事件类型。
   - 处理数据库版本/厂商差异。

2. **Execution Assembler**
   - 将 SQL 文本、duration、错误等多条日志关联成一次 SQL 执行事实。

3. **Fingerprint Wrapper**
   - 包装通用 SQL parser。
   - 处理 Greenplum/HashData 特有语法。
   - 实现无法解析时的降级规则。

4. **Training Sample Selector**
   - 决定哪些历史执行可以参与 Baseline。

5. **Baseline Builder**
   - 生成 P50/P95/P99/MAD、可信度和状态。

6. **Baseline Publisher**
   - 版本检查、切换和回滚。

真正决定系统准确性的主要是：

```text
原始日志 → 一次完整 SQL 执行
SQL 文本 → 稳定一致的 fingerprint
历史正常执行 → 可发布 Baseline
```

统计函数和基础设施本身不是难点。

---

## 推荐 MVP 技术栈

```text
开发语言：Python 3.12+
SQL parser：libpg_query 生态 / pglast
数据库：PostgreSQL 17
文件传输：rsync
任务调度：systemd timer
原始日志：本地文件系统 + gzip
展示：Grafana
版本管理：Git
```

第一阶段 Python 更适合快速验证 CSV、日志关联和 Baseline 规则。后续如果实时 activity 检测器需要高频扫描多个集群，可以再评估 Go。

---

## Greenplum / HashData 当前适配结论

### `gp_log_database` 不适合作为长期高频采集入口

经典 Greenplum 的 `gp_toolkit.gp_log_database` 基于日志外部表，底层实现会读取 Coordinator / Segment 日志目录中的 CSV 文件。

典型机制类似：

```text
gp_log_database
  ↓
gp_log_system
  ↓
__gp_log_master_ext + __gp_log_segment_ext
  ↓
cat pg_log/*.csv
```

因此：

```sql
WHERE logtime >= ...
```

可以减少返回和上层处理的数据，但通常不能像索引一样直接定位日志偏移；其“增量查询”并不等于“增量读取”。

如果 Baseline 采用日更方式，则更好的策略是：

> 每天直接复制 Coordinator 已轮转、已关闭的 CSV 日志文件到分析环境，在离线环境完成所有解析和统计。

这样可以最大程度避免对生产 GP 的周期性扫描。

### 在线检测来源

Greenplum/HashData 在线异常检测优先通过 activity 类系统视图获取：

```text
query/session id
query_start
state
wait_event / wait 类型
SQL text
resource group / queue（如可获得）
```

实时检测侧只关心当前仍未完成的 SQL，并与已发布 Baseline 匹配。

---

## 第一阶段实施顺序

1. 获取一批真实 Coordinator 已轮转 CSV 日志。
2. 明确日志字段、SQL 文本、duration、错误和 session/cmd 的关联关系。
3. 实现日志解析与 Execution Assembler。
4. 实现统一 SQL fingerprint。
5. 建立 `sql_execution` 和 `sql_fingerprint`。
6. 生成每个 fingerprint 的 P50/P95/P99。
7. 增加训练样本排除逻辑。
8. 增加 Baseline 版本化构建与发布。
9. 最后接入 activity 实时检测器。

第一版暂不做：

- 参数敏感自动聚类；
- 执行计划 Baseline；
- SQL 对象依赖分析；
- 工作日/周末和小时级模型；
- 自动接受 Baseline 漂移；
- AI 分析。

---

## 当前最重要的下一步

在正式设计全部数据库表和在线检测器之前，先用真实 Greenplum/HashData Coordinator CSV 日志验证：

```text
SQL 文本
+ session/cmd 标识
+ 开始时间
+ 完成时间或 duration
+ 最终状态
```

是否能够稳定还原为“一次 SQL 执行事实”。

这一步一旦稳定，后面的 Baseline 统计和发布基本属于常规数据工程问题。
