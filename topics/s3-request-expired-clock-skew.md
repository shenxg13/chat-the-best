# S3 / COS `Request has expired`：优先检查系统时间

## 适用场景

访问 S3 兼容对象存储（包括 COS 等）时返回：

```text
HTTP 403 Forbidden
<Code>AccessDenied</Code>
<Message>Request has expired</Message>
```

这类错误不要第一时间只盯着 AK/SK、Bucket 权限或网络策略。**请求签名依赖时间戳和有效期，客户端系统时间异常是高优先级排查项。**

## 已确认案例

一次 Oracle SBT/备份组件访问 S3 兼容对象存储时出现：

```text
Status => 403
Reason => Forbidden
<Code>AccessDenied</Code>
<Message>Request has expired</Message>
<ServerTime>2026-08-04T00:00:32Z</ServerTime>
```

而 Oracle trace 中本机记录的时间为：

```text
Fri, 26 Jun 2026 08:00:29
```

对象存储返回的 `ServerTime=2026-08-04T00:00:32Z` 换算成 UTC+8 为 `2026-08-04 08:00:32`，与客户端日志时间相差一个多月。

最终确认：**有人手动修改了服务器系统时间，导致客户端使用错误时间生成 S3 请求签名，对象存储端判断请求已经过期，因此返回 403 `AccessDenied / Request has expired`。**

## 原理

S3 风格的签名请求通常包含签名时间、凭证作用范围以及有效期等信息。大致链路为：

```text
客户端当前时间
    ↓
生成带时间信息的签名
    ↓
发送请求
    ↓
对象存储按服务端当前时间校验
    ↓
时间差过大 / 已超过有效期
    ↓
403 AccessDenied: Request has expired
```

因此：

- 网络完全可以是通的；
- Bucket 和对象也可能确实存在；
- AK/SK 本身也可能没有问题；
- 但只要客户端时钟偏差过大，签名仍会失败。

## 推荐排查顺序

看到明确的 `Request has expired` 时，建议优先执行：

```bash
date
date -u
timedatectl
```

如果使用 chrony：

```bash
chronyc tracking
chronyc sources -v
```

如果使用传统 ntpd：

```bash
ntpq -p
```

重点比较：

1. 本机当前时间；
2. 本机 UTC 时间；
3. 对象存储错误响应中的 `ServerTime`；
4. 应用 / Oracle / SBT trace 中记录的时间；
5. NTP/chrony 是否正常同步。

如果客户端时间和 `ServerTime` 明显不一致，应先修复系统时钟，再重新测试对象存储访问。

## 实际排障经验

当错误同时满足以下特征时，系统时间问题应排在非常靠前的位置：

```text
HTTP 403
AccessDenied
Request has expired
```

尤其是错误响应带有 `ServerTime` 时，可以直接拿它与客户端日志时间进行比较。这个证据通常比先反复检查 AK/SK、IAM、Bucket Policy 更直接。

需要注意，`ServerTime` 往往使用 UTC（`Z` 结尾），与本地时间比较前应先换算时区，避免把正常的 8 小时时区差误判为时钟异常。

## 运维侧建议

生产数据库、备份服务器和对象存储客户端所在主机应保持可靠的时间同步。对这类依赖签名的系统，手工修改系统时间可能同时影响：

- S3/COS API 签名；
- TLS 证书有效期判断；
- Kerberos 等依赖时钟的认证协议；
- 数据库、备份系统的日志时间线；
- 定时任务、监控和故障关联分析。

因此，如果必须调整系统时间，应明确评估依赖时间戳的外部认证和分布式系统影响，并优先通过标准时间同步机制恢复时钟。
