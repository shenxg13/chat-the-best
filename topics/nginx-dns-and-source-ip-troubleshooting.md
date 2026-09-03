# Nginx 动态 DNS、访问控制、限流与 upstream 超时排查

## 背景

2026-09-01 至 2026-09-03 排查一套两层 Nginx 反向代理链路的间歇性访问异常。

外层 Nginx 典型配置包括：

```nginx
resolver 10.214.100.13 valid=20s ipv6=off;
resolver_timeout 10s;
set $backend "typay02.swiftpass.top";

limit_req_zone $binary_remote_addr zone=req_limit:100m rate=2000r/s;
limit_conn_zone $binary_remote_addr zone=conn_limit:100m;
limit_req zone=req_limit burst=2000 nodelay;
limit_conn conn_limit 600;

allow 10.212.2.173;
allow 10.212.2.186;
deny all;

proxy_pass https://$backend:443;
proxy_http_version 1.1;
proxy_set_header Connection "";
proxy_connect_timeout 1200s;
proxy_send_timeout 1200s;
proxy_read_timeout 1200s;
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

但原配置只允许：

```nginx
allow 10.212.1.209;
deny all;
```

因此 Nginx 的访问控制与实际看到的源 IP 不匹配。

将白名单调整为实际源 IP 后，403 问题恢复正常：

```nginx
allow 10.212.2.173;
allow 10.212.2.186;
deny all;
```

多个 `allow` 地址必须分别写多条指令，不能用逗号拼在一条 `allow` 中。

本机压测外层 Nginx 时，为避免本机请求被 `deny all` 拒绝，需要临时加入：

```nginx
allow 127.0.0.1;
```

修改后先检查并 reload：

```bash
nginx -t
nginx -s reload
```

## 已确认问题 3：503 来自 `limit_req` 主动限流

后续 error.log 明确出现：

```text
limiting requests, excess: 100.xxx by zone "req_limit"
```

原限流曾配置为：

```nginx
limit_req_zone $binary_remote_addr zone=req_limit:100m rate=100r/s;
limit_req zone=req_limit burst=100 nodelay;
```

含义：同一个 `$remote_addr` 持续请求速率限制为 100r/s，允许额外突发 100 个请求；超过 burst 后 Nginx 默认返回 503。

因此：

- `zone=req_limit:100m` 中的 `100m` 是共享内存区大小，不代表请求资源只剩 100M；
- `excess: 100.xxx` 是限流算法里的超额请求量，不是 CPU、内存或连接资源不足；
- `limit_conn` 触发时日志会是 `limiting connections by zone ...`，与本次 `limit_req` 日志不同。

确认压测来源是单一真实终端后，将限流阈值调高后，原来的 503 限流错误消失。当前示例值为：

```nginx
limit_req_zone $binary_remote_addr zone=req_limit:100m rate=2000r/s;
limit_req zone=req_limit burst=2000 nodelay;
limit_conn conn_limit 600;
```

如果客户端 IP 实际是前置 LB / 网关 / NAT 地址，则仍需警惕 `$binary_remote_addr` 把多个真实客户端聚合进同一个限流桶；这种场景应优先正确恢复真实客户端 IP，而不是只调大限流。

## 已确认问题 4：少量 502 与 upstream TCP 建连超时直接对应

限流调整后，仍偶发少量 502。access.log 中失败请求耗时约 129～131 秒，例如：

```text
502 ... request_time≈129~131s
```

同期外层 Nginx error.log 出现：

```text
connect() failed (110: Connection timed out) while connecting to upstream
upstream: "https://123.207.100.60:443/..."
```

这说明失败发生在：

```text
外层 Nginx -> 123.207.100.60:443
```

的 TCP 建连阶段，而不是后端应用处理阶段。

`while connecting to upstream` 表示连接尚未建立完成，因此应优先排查 SYN/SYN-ACK、网络链路、中间防火墙/NAT、对端 443 接入能力，而不是 `proxy_read_timeout` 或应用响应慢。

## 约 130 秒超时不是 Nginx 配置的 1200 秒

外层 Nginx 明确配置了：

```nginx
proxy_connect_timeout 1200s;
proxy_send_timeout 1200s;
proxy_read_timeout 1200s;
```

但实际 connect timeout 约 129～131 秒，因此 130 秒不是这里直接配置的。

更符合 Linux 主动 TCP 建连时 SYN 重传最终超时的行为。可检查：

```bash
sysctl net.ipv4.tcp_syn_retries
```

若为：

```text
net.ipv4.tcp_syn_retries = 6
```

含义是：初始 SYN 发出后，Linux 最多再重传 SYN 6 次；一直收不到 SYN-ACK 时，内核最终让 `connect()` 返回 `ETIMEDOUT`。Nginx 随后记录：

```text
connect() failed (110: Connection timed out)
```

并向客户端返回 502。

因此即使 `proxy_connect_timeout` 配成 1200 秒，Linux TCP 栈也可能先在约两分钟左右结束底层 connect。

## 已完成的对照压测方法

### 1. 在外层 Nginx 本机走外层 Nginx 自己

为绕开前面的 CLB，但保留当前外层 Nginx -> upstream 链路，可强制域名解析到本机：

```bash
curl -sS \
  --resolve typay02.swiftpass.top:80:127.0.0.1 \
  -o /dev/null \
  -w 'code=%{http_code} connect=%{time_connect} total=%{time_total}\n' \
  http://typay02.swiftpass.top/css/default_login.css
```

单请求验证曾得到：

```text
code=200 connect=0.000155 total=0.195050
```

100 并发、10000 请求测试：

```bash
seq 1 10000 | xargs -P100 -I{} sh -c '
curl -sS \
  --resolve typay02.swiftpass.top:80:127.0.0.1 \
  -o /dev/null \
  -w "code=%{http_code} connect=%{time_connect} total=%{time_total}\n" \
  http://typay02.swiftpass.top/css/default_login.css
' | tee /tmp/nginx_100c_test.log
```

该轮 10000 个请求仅出现 1 个 502，说明故障更像低概率、间歇性的 upstream 建连异常，而不是一个非常明确的 100 并发容量阈值。

### 2. 从外层 Nginx 主机直接访问外部 upstream

用于绕过外层 Nginx 本身，直接验证当前主机 -> `123.207.100.60:443`：

```bash
seq 1 10000 | xargs -P100 -I{} sh -c '
curl -sS \
  --resolve typay02.swiftpass.top:443:123.207.100.60 \
  -o /dev/null \
  -w "code=%{http_code} connect=%{time_connect} total=%{time_total}\n" \
  https://typay02.swiftpass.top/css/default_login.css
' | tee /tmp/direct_100c_test.log
```

注意 `--resolve ...:443:...` 必须与 `https://` URL 对应；若 URL 写成 `http://`，默认走 80，443 的 `--resolve` 不会生效。

这种测试会保留正确的 Host 和 TLS SNI，但绕过 DNS、CLB 和外层 Nginx。

## 下一步定位建议

若继续复现 502，优先在外层 Nginx 主机同时观察连接状态和抓包：

```bash
watch -n 1 "ss -ant | grep '123.207.100.60:443' | awk '{print \$1}' | sort | uniq -c"
```

以及：

```bash
tcpdump -nn -i any host 123.207.100.60 and port 443
```

重点确认出现 502 的那个源端口是否表现为：

```text
SYN -> 重传 SYN -> 重传 SYN -> ... -> 始终没有 SYN-ACK
```

如果是，则可以进一步把问题锁定在当前 Nginx 主机出口、中间网络设备或对端 `123.207.100.60:443` 接入侧，而不是 Nginx 应用层配置。

## 可复用结论

1. Nginx 使用变量形式的 `proxy_pass` 时，后端域名依赖运行期 `resolver`；间歇性的 `Host not found` 应优先检查指定 DNS，而不是只看系统 `/etc/resolv.conf`。
2. `allow/deny` 默认依据 Nginx 实际看到的客户端地址判断。存在 LB、反向代理或 NAT 时，必须先确认 `$remote_addr` 实际值。
3. `limiting requests ... by zone "req_limit"` 可以直接确认是 `limit_req` 主动限流；`excess` 不是机器资源耗尽指标。
4. 压测时要区分“测试网关限流上限”和“测试后端真实性能”；前者保留限流，后者应避免限流规则成为瓶颈。
5. `connect() failed (110: Connection timed out) while connecting to upstream` 表示 TCP connect 阶段失败，不等价于应用响应超时。
6. 若 Nginx 的 `proxy_connect_timeout` 明显大于实际失败时间，而实际 connect timeout 稳定在约两分钟，应检查 Linux `net.ipv4.tcp_syn_retries` 和 SYN 重传行为。
7. 在 Nginx 本机使用 `curl --resolve 域名:80:127.0.0.1` 可以绕过 CLB、保留 Host，并测试当前 Nginx -> upstream；使用 `--resolve 域名:443:目标IP` + `https://` 可以直接测试当前主机 -> upstream。
8. 本次测试中，100 并发、10000 请求仅复现 1 次 502，更支持“低概率间歇性 TCP 建连异常”，而不是简单的 100 并发容量不足。
