# TiDB 临时表 `The table 'xxx' is full`：优先检查 `tidb_tmp_table_max_size`

## 适用场景

TiDB 中业务程序执行类似下面的逻辑：

```sql
INSERT INTO <temporary_table> (...)
SELECT ...
FROM ...
GROUP BY ...;
```

随后报错：

```text
DBD::mysql::st execute failed:
The table '<temporary_table>' is full
```

如果目标表是 TiDB 的真正临时表（而不是仅仅表名以 `TMP_` 开头的普通表），应优先检查临时表大小上限：

```sql
SELECT
    @@SESSION.tidb_tmp_table_max_size AS session_tmp_table_max_size,
    @@GLOBAL.tidb_tmp_table_max_size  AS global_tmp_table_max_size;
```

## 已确认案例

2026-09-30，一次批处理在向临时表写入聚合结果时失败，核心错误为：

```text
The table 'TMP_AML_BIGAMT_SUPP_REPORT_RPT_A' is full
```

出错 SQL 结构为：

```sql
INSERT INTO TMP_AML_BIGAMT_SUPP_REPORT_RPT_A (...)
SELECT ...
FROM ...
LEFT JOIN (...)
...
GROUP BY ...;
```

最终确认：

- 目标表确实是 TiDB 临时表；
- 临时表数据量达到 `tidb_tmp_table_max_size` 上限；
- 通过**当前业务连接的 SESSION 级别**提高 `tidb_tmp_table_max_size` 后，批处理恢复正常。

因此，本案例可以直接归纳为：

> TiDB 报 `The table 'xxx' is full`，且 `xxx` 为真正的临时表时，优先排查 `tidb_tmp_table_max_size`，不要先把问题归因到 TiKV 数据盘或 SQL spill 临时目录。

## 推荐处理方式

如果只需要放宽某个批处理或某个业务连接的限制，优先使用 SESSION 级别：

```sql
SET SESSION tidb_tmp_table_max_size = <bytes>;
```

例如需要将当前 session 上限临时调整到 512 MiB：

```sql
SET SESSION tidb_tmp_table_max_size = 536870912;
```

这样只影响当前连接，风险和影响范围都比直接修改全局参数小。

注意：

- `SET SESSION` 必须与创建、使用该临时表的业务 SQL 位于**同一个数据库连接**；
- 如果应用断开后重新建连，需要重新设置 SESSION 参数；
- 如果通过应用程序设置，应确保连接池不会让后续 SQL 切换到其他 session。

如果确实需要所有新连接都采用新的默认值，才考虑：

```sql
SET GLOBAL tidb_tmp_table_max_size = <bytes>;
```

修改 GLOBAL 后，应确认业务是否重新建立了连接，因为已经存在的 session 不应简单假定会自动获得新的 session 值。

## 排障时先确认“TMP 表”是不是真临时表

表名里包含 `TMP`、`TEMP` 并不能证明它是临时表。

应检查实际 DDL 是否属于临时表，例如：

```sql
CREATE TEMPORARY TABLE ...
```

如果它只是普通表：

```sql
CREATE TABLE TMP_XXX ...
```

那么 `tidb_tmp_table_max_size` 就不是这个报错的优先方向，需要重新检查真正的容量、资源或执行错误来源。

对于 session 级临时表，另开一个 DBA 连接时可能无法像普通表一样直接观察到业务 session 中的对象，所以必要时应从应用日志、SQL trace 或创建临时表的同一连接确认 DDL。

## 不要和 SQL 执行算子的临时磁盘混淆

`tidb_tmp_table_max_size` 管的是 TiDB 临时表本身的容量限制。

它和 Hash Join、HashAgg、Sort 等 SQL 算子内存不足后 spill 到临时磁盘不是同一类问题。

因此看到：

```text
The table 'xxx' is full
```

且目标对象已经确认为临时表时，排查顺序应优先是：

1. 确认对象是否是真正的临时表；
2. 查看当前 session 的 `tidb_tmp_table_max_size`；
3. 评估本次 `INSERT ... SELECT` 结果集大小；
4. 必要时在同一业务连接中适当提高 SESSION 上限；
5. 再观察 TiDB Server 内存占用和业务并发情况。

不应首先把精力放在：

- TiKV 数据盘是否已满；
- `tmp-storage-path`；
- SQL spill 临时目录；
- 单纯扩大磁盘空间。

除非同时还有其他证据指向这些资源。

## 运维侧经验

临时表上限并非越大越好。临时表数据会增加 TiDB Server 的内存压力，如果多个任务并发创建大临时表，过度提高全局上限可能放大内存风险。

生产环境中更推荐：

```text
单个特殊批处理需要更大临时表
        ↓
优先 SESSION 级提高
        ↓
只覆盖该任务实际所需容量
        ↓
观察 TiDB Server 内存和并发
        ↓
确认业务完成后无需永久扩大默认值
```

本案例采用的就是 **SESSION 级修改**，问题已经处理完成。
