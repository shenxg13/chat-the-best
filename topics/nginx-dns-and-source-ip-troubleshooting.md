# Nginx 动态 DNS 与源 IP 访问控制排查

## 背景

2026-09-01 排查一台 Nginx 反向代理的间歇性访问异常。

相关配置模式：

```nginx
resolver 10.214.100.13 valid=20s ipv6=off;
resolver_timeout 10s;

set $backend "typay02.swiftpass.top";

allow 10.212.1.209;
deny all;
```

后端域名通过变量用于 `proxy_pass`，因此 Nginx 会在运行期使用 `resolver` 指定的 DNS 做动态解析。

## 已确认问题 1：动态 DNS 解析不稳定

错误日志曾频繁出现：

```text
typay02.swiftpass.top could not be resolved (3: Host not found)
unexpected DNS response for typay02.swiftpass.top
```

同时并非 100% 失败，部分请求可以正常解析并访问。

排查结论：问题位于 DNS 解析链路，而不是 `proxy_pass` 本身。因为使用变量后，Nginx 会按 `resolver` 配置反复解析后端域名；DNS 返回不一致时会表现为一段时间成功、一段时间失败。

后续 DNS 问题已解决。

### 同类问题排查方法

可直接在 Nginx 节点持续查询指定 DNS，确认是否出现 `NOERROR` / `NXDOMAIN` / 超时等结果交替：

```bash
for i in $(seq 1 100); do
    echo "===== $(date '+%F %T') ====="
    dig @10.214.100.13 typay02.swiftpass.top A \
        +time=2 +tries=1 +noall +comments +answer
    sleep 1
done
```

如果指定 DNS 后面是 VIP、DNS 集群或转发器，还需要继续确认不同后端节点/上游 DNS 的记录是否一致。

## 已确认问题 2：Nginx 实际看到的源 IP 与 allow 白名单不一致

DNS 修复后，access log 中出现大量 403/503。日志里的客户端源地址实际为：

```text
10.212.2.173
10.212.2.186
```

但配置只允许：

```nginx
allow 10.212.1.209;
deny all;
```

因此 Nginx 的访问控制与实际看到的源 IP 不匹配。

将白名单调整为实际源 IP 后，访问恢复正常：

```nginx
allow 10.212.2.173;
allow 10.212.2.186;
deny all;
```

多个 `allow` 地址必须分别写多条指令，不能用逗号拼在一条 `allow` 中。

修改后先检查并 reload：

```bash
nginx -t
nginx -s reload
```

## 可复用结论

1. Nginx 使用变量形式的 `proxy_pass` 时，后端域名依赖运行期 `resolver`；间歇性的 `Host not found` 应优先检查指定 DNS，而不是只看系统 `/etc/resolv.conf`。
2. `allow/deny` 默认依据 Nginx 实际看到的客户端地址判断。如果链路中存在 LB、反向代理或 NAT，Nginx 看到的可能是中间设备 IP，而不是真实终端 IP。
3. 配置 IP 白名单时，应先从 access/error log 确认 `$remote_addr` 实际值；如果业务要求按真实客户端 IP 做访问控制或限流，则应进一步确认前置设备是否传递可信的真实 IP Header，并正确配置 Nginx `real_ip` 相关规则。
4. 本次故障中，DNS 问题修复后，最终通过修正 `allow` 白名单使访问恢复；此前对 503 是否由 `limit_req` 引起并未做最终确认，因此不应将其作为本次已证实根因。
