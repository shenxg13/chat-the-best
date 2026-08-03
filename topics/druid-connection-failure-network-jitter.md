# Druid 数据库连接失效：现网初步分析

## 背景

现网应用使用 Druid 1.1.21 维护数据库连接池，出现 `CommunicationsException`，Druid 随后将异常连接 `discard`。

当前相关 Druid 配置中可见：

- `initialSize = 5`
- `minIdle = 80`
- `maxActive = 80`
- `validationQuery = SELECT 1`
- `testWhileIdle = true`
- `testOnBorrow = false`
- `testOnReturn = false`
- `timeBetweenEvictionRunsMillis = 1000`
- `minEvictableIdleTimeMillis = 400000`
- `keepAlive = true`

故障日志中可见的关键字段包括：

- `poolingCount = 78`
- `idleMillis = 33`
- `lastValidIdleMillis = 121260`
- `minIdle = 80`
- `CommunicationsException`
- `discard connection`

## 第一版结论

结合现网特点，目前第一版判断为：

> **更可能是现网网络抖动导致已经建立的数据库 TCP 连接失效，Druid 后续在使用或检查该连接时发现通信失败并将其丢弃。**

现阶段不把 Druid 本身认定为连接中断的直接根因。Druid 更像是在已有连接因网络异常失效后，暴露并清理失效连接。

该结论目前属于阶段性判断，后续如果能够拿到网络设备、数据库端连接中断日志或抓包结果，需要继续验证。

## 分析过程

### 1. `poolingCount=78` 的含义

`poolingCount` 表示当前 Druid 连接池中处于 pooling 状态、可以被业务借用的**空闲连接数**，不是数据库总连接数。

可以近似理解为：

```text
总物理连接数 ≈ poolingCount + activeCount
```

因此日志中的：

```text
poolingCount = 78
minIdle      = 80
maxActive    = 80
```

说明当时池内有 78 个空闲连接；如果总连接数接近 80，则另外约 1～2 个连接可能正在被业务线程持有。

### 2. `timeBetweenEvictionRunsMillis=1000` 不等于每个连接每秒执行一次 `SELECT 1`

该配置容易误解。

`timeBetweenEvictionRunsMillis=1000` 表示 Druid 后台相关检查任务的运行周期，同时会参与连接空闲检查逻辑；它并不意味着所有 idle connection 每隔 1 秒都会执行一次 `validationQuery`。

因此，即使配置了：

```text
testWhileIdle = true
validationQuery = SELECT 1
timeBetweenEvictionRunsMillis = 1000
```

也不能据此推导出“一个连接两分钟没有成功探测，就说明两分钟内 Druid 每秒探测都失败”。

### 3. `idleMillis=33` 是一个重要现场信息

日志中的：

```text
idleMillis = 33
```

说明打印该日志时，这个 connection 刚刚处于 idle 状态约 33 ms。

这意味着它并不是已经连续 2 分钟待在 idle pool 中。

一个可能的过程是：

```text
connection 被业务 borrow
        ↓
在业务侧持有一段时间
        ↓
connection 被归还到连接池
        ↓
约 33 ms
        ↓
Druid 后续检查/操作该连接
        ↓
CommunicationsException
        ↓
discard connection
```

### 4. `lastValidIdleMillis=121260` 提供约 2 分钟的线索，但不能单独证明连接连续 active 了 2 分钟

日志中的：

```text
lastValidIdleMillis = 121260 ms
```

约等于 121 秒。

这个时间与 `idleMillis=33` 差距很大，因此可以判断：这个连接当前刚进入 idle 状态，而距离上次被确认有效已经过去较长时间。

但仅凭 `lastValidIdleMillis` 不能严格证明：

```text
这 121 秒连接始终处于 active / borrowed 状态
```

期间可能存在多次 borrow/return，具体需要结合 Druid 内部状态、监控或业务调用栈进一步确认。

### 5. Druid 的 active 与网络设备的 TCP idle 是两个不同概念

即使一个 connection 已经被业务 borrow，对 Druid 来说属于 active，也不意味着这条 TCP 连接上持续有报文。

例如：

```text
应用 getConnection()
        ↓
connection 离开 Druid idle pool
        ↓
业务执行其他逻辑 / 等待远端调用 / 锁等待
        ↓
较长时间没有数据库网络流量
```

此时：

- 对 Druid：connection 是 active；
- 对防火墙、LB、NAT 或数据库网络链路：TCP 连接可能实际上处于 idle。

如果链路有 idle timeout，或者现场发生网络抖动，已经建立的连接仍然可能失效。

### 6. 为什么当前更倾向于网络抖动

现网存在网络抖动这一已知特点；而日志表现为已有连接发生 `CommunicationsException` 后被 Druid 丢弃，与网络瞬断、TCP session 被中间链路清理、连接半开等场景是一致的。

Druid 配置可能影响问题的暴露方式和恢复速度，但目前没有证据表明 `timeBetweenEvictionRunsMillis=1000`、`testWhileIdle=true` 本身会主动制造这类通信异常。

因此当前优先级排序是：

1. **网络抖动导致已有数据库连接失效** —— 当前第一版结论；
2. 链路中的防火墙 / LB / NAT idle timeout 清理连接；
3. 业务较长时间持有数据库连接且期间无流量，使连接更容易受到网络异常或 idle timeout 影响；
4. Druid 参数设置不合理放大连接数量或失效连接暴露概率，但不是首要根因。

## Druid 配置中值得后续单独评估的点

### `minIdle = maxActive = 80`

该组合意味着连接池会倾向于长期维持接近 80 个数据库连接。若实际并发不需要这么多连接，会增加长期空闲连接数量，也会增加网络抖动时需要恢复的连接数量。

该配置值得后续根据真实并发、连接创建代价和数据库承载能力重新评估，但当前不要把它和本次网络类 `CommunicationsException` 直接画等号。

### `timeBetweenEvictionRunsMillis = 1000`

1 秒的后台检查周期较激进，也值得后续评估是否有必要。

但要注意，它不等同于“每个连接每秒执行 `SELECT 1`”，因此不能简单用该参数否定网络导致连接失效的可能性。

## 后续验证方向

若继续推进根因确认，优先收集：

1. 故障时间点对应的网络设备、防火墙、LB、NAT 日志；
2. 数据库端是否记录连接 reset / broken pipe / EOF 等异常；
3. 应用侧 `CommunicationsException` 是否集中出现在同一秒或同一小时间窗；
4. 同一时刻是否有多个连接同时被 Druid discard；
5. 是否存在固定约 120s、300s、600s 等周期特征，以排查 idle timeout；
6. Druid 的 `activeCount`、`poolingCount` 和 discard/create 连接数趋势；
7. 必要时通过抓包确认 TCP RST、FIN、重传和连接半开情况；
8. 若怀疑连接被业务长时间持有，再针对 connection borrow 时长做单独诊断。

## 当前状态

当前先采用“**网络抖动造成 Druid 中已有数据库连接失效**”作为第一版现场结论；Druid 负责检测和清理异常连接。后续如取得更强证据，再更新本文结论。
