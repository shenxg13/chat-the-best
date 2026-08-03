# Greenplum / HashData SQL Baseline 先导研究

## 目的

本文记录在正式设计 SQL Baseline 系统之前，对现网 Greenplum / HashData 原生监控能力、日志能力和存储相关机制的确认结果。

目标不是设计 AI Agent，而是回答两个基础问题：

1. 建立 SQL Baseline 所需的基础事实数据，Greenplum / HashData 能原生提供到什么程度？
2. Baseline 系统后续应以什么数据源和工程边界为前提？

## 现网环境

已确认版本：

- PostgreSQL 9.4.26
- Greenplum Database 6.20.3
- HashData Warehouse 3.13.13

架构特征：

- HashData 为基于 Greenplum 6 的定制版本。
- 存在共享 Catalog / 计算存储分离相关增强。
- 业务数据存储在 S3。
- 同一业务固定路由到同一个 GP 计算集群。
- 大部分数据库监控接口仍沿用 Greenplum 6 的 `pg_catalog` 和 `gp_toolkit` 体系。

## 核心结论一：系统视图不能直接构建 SQL Baseline 历史事实表

已经确认，仅依赖 MPP / GP 的系统视图，无法获得“每次 SQL 执行一条记录”的完整历史流水。

典型能力边界：

- `pg_stat_activity`：当前会话 / 当前或最近 SQL 的运行态信息；SQL 完成后不形成可长期使用的执行历史。
- `gp_resq_activity`、`gp_resqueue_status`：队列和排队运行态信息，不是执行历史。
- `gp_workfile_usage_per_query` 等：适合捕获执行中的 spill / workfile 状态，但 SQL 完成后不能作为历史事实源。
- `pg_stat_*`：主要是数据库、表、函数等对象的累计统计，不是单次 SQL 明细。

因此 Baseline 的基础事实表不能只靠系统视图构建。

建议的数据源定位：

- **日志：历史执行事实的主数据源。**
- **系统视图：运行态补充源。** 用于长时间未结束 SQL、排队、锁等待、workfile、资源状态等实时或瞬时信息。
- **Catalog / 统计视图：上下文源。** 用于补充表类型、分布策略、AO/Heap、统计信息、对象大小等环境特征。

## 核心结论二：现网日志与 Greenplum 6 标准 CSV 日志格式匹配

现场日志片段与 Greenplum 6 的专有 CSV server log 高度匹配。

可见典型字段包括：

- `event_time`
- `user_name`
- `database_name`
- `process_id`（如 `p3111686`）
- `thread_id`
- `remote_host`
- `remote_port`
- `session_start_time`
- `transaction_id`
- `gp_session_id`（如 `con3868347`）
- `gp_command_count`（如 `cmd7`）
- `gp_segment`（如 `seg-1`）
- `event_severity`
- `sql_state_code`
- `event_message`（例如 `duration: 1937.324 ms`）
- 多行 SQL 文本
- `file_name`（例如 `autostats.c` / `postgres.c`）
- `file_line`

Greenplum 标准日志的一条逻辑 CSV 记录可以跨多个物理文本行，因为 SQL 本身可能包含换行、逗号和引号。

### 对日志解析器的约束

不能采用“逐物理行 + 正则切割”方式，也不能简单把 `duration:` 前后的文本拼接成一条 SQL。

正确方式应是：

- 按 CSV 引号规则读取**逻辑 record**；
- 使用成熟 CSV parser 处理 SQL 内部的换行、逗号和双引号；
- 解析后输出标准化的 `ExecutionRecord`。

### SQL 执行关联键

Greenplum 日志中的：

```text
gp_session_id + gp_command_count
```

可用于关联同一会话中的一次 command / SQL。

为了跨集群、长期保存和避免重启后潜在重复，工程上建议进一步带上：

```text
cluster_id
+ session_start_time
+ gp_session_id
+ gp_command_count
```

作为执行事件的稳定关联维度。

## 核心结论三：第一版 Baseline 已有足够基础字段

从现网日志中至少可以稳定获得构建第一版执行基线所需的核心字段：

- SQL 原文
- 执行时间戳
- duration
- database
- user / role
- client host / port
- process / thread
- session id
- command id
- segment
- SQL state / severity
- 错误或日志消息

这些字段已经足以支持：

- SQL 归一化和指纹生成
- SQL 执行次数统计
- 平均值、P50/P90/P95/P99 等耗时基线
- 按小时 / 天 / 时间窗口聚合
- 按数据库、用户、集群分类
- Top SQL
- 执行时间异常偏离检测

后续是否还能稳定得到行数、错误上下文、bind 参数、计划信息等，应单独验证，不应在第一版设计里假设一定存在。

## Greenplum 原生监控能力的成本分层

不能只记录“有无某个视图”，还需要记录查询成本和是否适合实时采集。

建议长期采用类似以下分类：

| 能力 | 典型接口 | 成本 | 是否适合实时 |
|---|---|---:|---|
| 当前 SQL | `pg_stat_activity` | 低 | 是 |
| 资源队列 | `gp_resqueue_status` / `gp_resq_activity` | 低~中 | 是 |
| Workfile / Spill | `gp_workfile_*` | 中 | 是，需控制频率 |
| 表 / Catalog 统计 | `pg_stat_*` / catalog | 低~中 | 可定期 |
| 数据倾斜 | `gp_skew_coefficients` | 高 | 否 |
| 数据倾斜 | `gp_skew_idle_fractions` | 高 | 否 |

## `gp_skew_coefficients` 的实现与成本确认

已沿 `gp_toolkit` 源码定义逐层下钻：

```text
gp_skew_coefficients
  -> __gp_skew_coefficients()
  -> gp_skew_coefficient(oid)
  -> gp_skew_details(oid)
```

### 顶层行为

`__gp_skew_coefficients()` 会枚举 `gp_toolkit.__gp_user_data_tables_readable` 中的用户数据表，并对每张表调用 `gp_skew_coefficient()`。

因此它天然是全库级循环，不适合作为高频实时监控查询。

### AO 表

`gp_skew_details()` 对 AO 表主要通过：

```text
pg_catalog.pg_aoseg_distribution(oid)
```

读取各 Segment 的 tuple count 元数据，不需要直接扫描业务表。

### Heap 表

对 Heap 表则动态执行等价于：

```sql
SELECT gp_segment_id, COUNT(*)
FROM <table>
GROUP BY gp_segment_id;
```

同时结合 segment 数生成空 segment 行。

因此在存在大量或超大 Heap 表时，查询 `gp_skew_coefficients` 实际上可能触发大量全表扫描。

结论：

> `gp_skew_coefficients` 是诊断工具，不是实时监控指标源。

## AO / Heap 相关确认

AO = **Append Only**，不是 Ordered。

它不是“有序表”的概念，也不意味着 SQL 查询顺序有任何保证。

存储类型应区分：

- Heap
- AO Row
- AO Column

Greenplum 的 AO 是 GP 面向数据仓库场景提供的存储机制，并非 PostgreSQL 原生 Heap 概念的简单别名。

AO 表的 skew 统计可以利用 AO 元数据，而 Heap 表通常无法直接从同类元数据得到精确 Segment tuple count，因此 Toolkit 会退化为实际 `COUNT(*) GROUP BY gp_segment_id`。

## Baseline 数据库的技术选型边界

SQL Baseline 的核心模型**不依赖 PostgreSQL 独有特性**。

核心逻辑本质是：

```text
SQL Execution History
  -> Normalize / Fingerprint
  -> Window Aggregation
  -> Baseline Statistics
  -> Anomaly Detection
```

只要数据库支持常规表、索引、聚合、唯一约束和时间分区等基本能力，就可以实现。

### PostgreSQL 的优势但非硬依赖

若第一版使用 PostgreSQL，可便利地利用：

- JSONB
- 分区表
- `INSERT ... ON CONFLICT`
- `avg/stddev/percentile_cont`
- BRIN

但核心业务逻辑不应绑定到：

- TimescaleDB
- pgvector
- 复杂 PL/pgSQL
- 大量数据库触发器
- PostgreSQL 专属调度机制

推荐把以下能力放在应用层，保持数据库可替换：

- 日志解析
- SQL 归一化
- SQL 指纹算法
- 时间窗口定义
- Baseline 算法
- 异常检测规则
- 告警生命周期

### 其他数据库

- MySQL：可以实现核心模型，但复杂统计和分析能力通常不如 PostgreSQL 顺手。
- TiDB：适合更大规模和横向扩展，但不是第一版必须引入。
- ClickHouse：非常适合未来大规模 SQL 执行明细、长期保留和大量百分位聚合；可在规模上升后将事实明细与 PostgreSQL 控制元数据拆分。
- OpenSearch：适合作为 SQL / 错误日志全文检索补充，不建议作为 Baseline 主数据库。

当前倾向：

> 第一版使用 PostgreSQL，但保持核心数据模型和算法数据库无关；未来如明细规模显著增长，再考虑将执行明细和分析查询拆到 ClickHouse。

## 建议的系统分层前提

在正式设计 Baseline 之前，先固定以下边界：

```text
Greenplum CSV Log
      |
      v
Log Parser / Adapter
      |
      v
ExecutionRecord（数据库无关标准事件）
      |
      +--> SQL Normalize / Fingerprint
      |
      v
Execution History
      |
      v
Baseline Aggregator
      |
      v
Baseline / Anomaly
```

与此同时：

```text
pg_stat_activity / gp_resqueue / gp_workfile / catalog
```

作为运行态和上下文补充数据源，而不是替代日志事实流。

## 下一阶段正式设计应回答的问题

进入 SQL Baseline 正式设计后，优先确定：

1. `ExecutionRecord` 的标准字段定义，以及哪些字段来自日志、哪些来自运行态补充。
2. SQL 归一化和 Fingerprint 的稳定算法。
3. 执行明细的保留周期、分区策略和幂等键。
4. Baseline 聚合维度和时间窗口。
5. P50/P95、Median/MAD、Stddev 等统计指标的选择原则。
6. 异常 SQL 是否进入 Baseline，以及失败 SQL 如何分类处理。
7. 实时异常检测（`pg_stat_activity`）与离线 Baseline 更新如何解耦。
8. 执行计划、表倾斜、workfile 等增强特征在什么时候采集，而不是对每条 SQL 强制采集。

本文属于 SQL Baseline 的先导研究，后续正式方案应以这些已经确认的能力边界为前提，而不再重复假设“可以从 GP 系统视图直接获得完整 SQL 历史”。
