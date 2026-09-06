# Oracle 批量 DELETE 无索引导致重复全表扫描：AWR、HWM 与止损判断

## 背景

Oracle 19.20 RAC 环境中，排查 PDB AWR 里一条 CPU/Elapsed Time 都很高的 DELETE。

目标表：

```text
PPSS.BSOP_CHL_ELB
```

关键 SQL：

```sql
delete from ppss.bsop_chl_elb
where cst_id = :1
   or cst_id is null;
```

当前执行计划：

```text
DELETE STATEMENT
  DELETE BSOP_CHL_ELB
    TABLE ACCESS FULL BSOP_CHL_ELB
```

Plan Hash Value：

```text
1485910571
```

`GV$SQL` 中两个 RAC 实例都能看到该 SQL。

## 已确认的表状态

排查时实际数据：

```text
TOTAL        = 190243
NON_NULL     = 190243
NULL_CNT     = 0
DISTINCT_CST = 190243
```

因此当前数据满足：

```text
CST_ID 无 NULL
CST_ID 无重复
```

但表定义中 `CST_ID` 仍允许 NULL，且表上没有可用索引。

统计信息 / segment 情况：

```text
DBA_TABLES.NUM_ROWS    ≈ 528243   # 统计信息采集时的行数
DBA_TABLES.BLOCKS      ≈ 10097
AVG_ROW_LEN            ≈ 62
LAST_ANALYZED          = 2026-09-06 06:10:36

DBA_SEGMENTS.SIZE      ≈ 80 MB
DBA_SEGMENTS.BLOCKS    ≈ 10240
```

结论：这不是一个“巨大 segment / HWM 异常膨胀”的表；当前 HWM 大约就是 1 万个 block、80 MB 左右。

## AWR 现象与根因判断

AWR 中该 DELETE 的典型指标：

```text
Executions             ≈ 527
Elapsed / Exec         ≈ 74.72 s
CPU / Exec             ≈ 73.25 s
Buffer Gets / Exec     ≈ 16,442,333
IO Wait                ≈ 0
```

这说明：

1. 主要瓶颈是 CPU + 逻辑读，不是物理 I/O；
2. 表只有约 1 万 block，但一次 AWR execution 产生约 1644 万 buffer gets；
3. 折算下来，相当于一次 execution 内把这张表完整扫描约 1600 次：

```text
16,442,333 / 10,097 ≈ 1,628
```

程序侧确认“2000 一提交”。结合 AWR 量级，最可能的执行模式是批量/array DML 或同一事务内大量重复 DELETE：每个 `CST_ID` 在没有索引的情况下都触发一次 Full Table Scan。

因此 70 多秒不是“一次扫描 80 MB 表要 70 秒”，而是更接近：

```text
一次 Full Scan 约几十毫秒
× 一千多到两千个 bind
≈ 几十秒
```

如果开发所说的“2000 一提交”只是循环 2000 次 `executeUpdate()` 后再 `commit`，而不是 `addBatch()/executeBatch()`，那么 AWR 的 execution 计数语义会不同；需要结合客户端实现确认。但“不带索引时每个 bind 都要扫表”这一根因不变。

## 为什么普通单列索引未必直接解决当前 SQL

当前谓词是：

```sql
where cst_id = :1
   or cst_id is null
```

Oracle 普通单列 B-tree 索引不保存索引键全为 NULL 的条目，因此：

```sql
create index ... on ppss.bsop_chl_elb(cst_id);
```

虽然非常适合 `cst_id = :1`，但对 `cst_id is null` 这一支没有天然索引入口。

由于列定义目前也不是 `NOT NULL`，优化器不能仅凭当前数据“恰好没有 NULL”就永久认为 `IS NULL` 不可能成立。

因此只加一个普通单列索引后，这条带 `OR cst_id IS NULL` 的 SQL 是否能改走索引，必须看实际执行计划，不能预设一定有效。

## 当前生产约束与止损决策

当前阶段：

- 不能做 DDL；
- 不能修改 DELETE SQL；
- 表只剩约 19 万行待清理；
- 这是正在运行中的批量任务。

因此当前决策是：

> 让本轮清理自然跑完，不为了临时提速在线做高风险结构变更。

不建议优先尝试：

```text
增大 Buffer Cache
调 PGA
调存储
重新收集统计信息
SQL Profile / 换计划
强行并行 Full Scan
```

原因是当前没有更好的物理访问路径，而且 AWR 已证明几乎不是 I/O 等待问题。

## 清空以后，第二天数据量很小时为什么仍可能慢

如果今天用普通 `DELETE` 把表清到 0 行：

```text
DELETE 不会自动降低 High Water Mark
```

所以明天即使只插入几千条，表可能仍然是：

```text
实际行数：几千
HWM：约 10000 blocks
segment：约 80 MB
```

只要 SQL 和索引情况不变，每个 `CST_ID` 的删除仍可能扫描接近整个 HWM。

因此明天如果仍然 2000 一批/一事务：

```text
2000 × Full Table Scan
```

单批仍然可能是几十秒级；只是因为总数据量只有几千，总 batch 数少，所以整批任务总耗时会明显缩短。

注意：空块变多后，真实行判断、undo/redo 等工作可能减少，因此单批有可能比当前快一些，但不会因为 `COUNT(*)` 只有几千就自动变成毫秒级点查。

## 长期正确整改方向

如果业务规则最终确认是：

```text
CST_ID 必须有值
CST_ID 全表唯一
```

建议最终把数据库约束与业务规则对齐：

```sql
alter table ppss.bsop_chl_elb
modify cst_id not null;

alter table ppss.bsop_chl_elb
add constraint uk_bsop_chl_elb_cst_id
unique (cst_id);
```

同时把 SQL 收敛为：

```sql
delete from ppss.bsop_chl_elb
where cst_id = :1;
```

这样才能从根本上把访问路径从：

```text
每个 bind -> Full Table Scan
```

变成：

```text
每个 bind -> INDEX UNIQUE SCAN / ROWID 定位
```

如果该表本质上是“每天装载、每天清空”的工作表，还应评估是否允许用 `TRUNCATE` 取代“全量 DELETE 清空”；`TRUNCATE` 可以重置 HWM，而普通 DELETE 不会。但是否可用必须结合事务语义、外键、权限和业务行为确认。

## 常用排查 SQL

### 看当前/历史执行计划

```sql
select *
from table(
    dbms_xplan.display_cursor(
        'SQL_ID',
        0,
        'ALLSTATS LAST +PEEKED_BINDS'
    )
);
```

```sql
select *
from table(
    dbms_xplan.display_awr('SQL_ID')
);
```

### 看 RAC 各实例 SQL 统计

```sql
select inst_id,
       executions,
       rows_processed,
       buffer_gets,
       round(buffer_gets / nullif(executions,0)) gets_per_exec,
       round(rows_processed / nullif(executions,0),2) rows_per_exec,
       round(cpu_time / 1e6 / nullif(executions,0),2) cpu_s_per_exec,
       round(elapsed_time / 1e6 / nullif(executions,0),2) ela_s_per_exec
from gv$sql
where sql_id = 'SQL_ID';
```

### 看索引与索引列

```sql
select owner,
       index_name,
       index_type,
       uniqueness,
       status,
       partitioned,
       visibility
from dba_indexes
where table_owner = 'PPSS'
  and table_name  = 'BSOP_CHL_ELB'
order by index_name;
```

```sql
select index_owner,
       index_name,
       column_position,
       column_name,
       descend
from dba_ind_columns
where table_owner = 'PPSS'
  and table_name  = 'BSOP_CHL_ELB'
order by index_name, column_position;
```

### 看行数、NULL、唯一性

```sql
select count(*) total,
       count(cst_id) non_null,
       count(*) - count(cst_id) null_cnt,
       count(distinct cst_id) distinct_cst
from ppss.bsop_chl_elb;
```

### 看 HWM / segment 量级

```sql
select num_rows,
       blocks,
       empty_blocks,
       avg_row_len,
       last_analyzed
from dba_tables
where owner = 'PPSS'
  and table_name = 'BSOP_CHL_ELB';
```

```sql
select bytes/1024/1024 mb,
       blocks
from dba_segments
where owner = 'PPSS'
  and segment_name = 'BSOP_CHL_ELB'
  and segment_type = 'TABLE';
```

## 后续再次遇到同类问题时的判断顺序

1. AWR 先看 `Elapsed / Exec`、`CPU / Exec`、`Buffer Gets / Exec`、`IO Wait`；
2. `DBMS_XPLAN.DISPLAY_AWR` / `DISPLAY_CURSOR` 确认访问路径；
3. 看表 block / segment 大小，判断一次 Full Scan 本身是否异常；
4. 如果 `Gets / Exec` 远大于表 blocks，优先考虑一次业务调用里发生了大量重复扫描，而不是只盯着一次 Full Scan；
5. 确认客户端 batch / array DML / commit 边界；
6. 再决定是索引、SQL 改写、约束收口、降低并发，还是只做临时止损。
