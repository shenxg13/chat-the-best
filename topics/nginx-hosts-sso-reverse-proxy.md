# Nginx + hosts 场景下的 HTTPS / SSO 反向代理

## 背景

某终端通过本地 `hosts` 将外部业务域名解析到一台 Nginx，再由 Nginx 转发到真实外部站点。

典型链路：

```text
终端浏览器
  -> hosts: xinxipilu 域名 -> Nginx
  -> Nginx:8080 HTTP 反向代理
  -> 真实 xinxipilu HTTPS:443
```

登录时，页面点击“登录”按钮后，浏览器会跳转到另一个 SSO 域名（如 `wsso`），地址形如：

```text
https://wsso.example.com/...
```

终端的 `hosts` 同样把 `wsso.example.com` 指向这台 Nginx。

因此浏览器跳转后实际访问的是：

```text
Nginx_IP:443
```

而不是原来的 `8080`。

---

## 多个 `server` 监听同一个端口

Nginx 的 `http` 模块允许多个 `server` 同时监听同一个端口，例如：

```nginx
server {
    listen 8080;
    server_name a.example.com;
}

server {
    listen 8080;
    server_name b.example.com;
}
```

Nginx 会根据 HTTP `Host` 头匹配 `server_name`。

真正有问题的是同一个 `listen` + 同一个 `server_name` 被重复定义，例如：

```nginx
server {
    listen 8080;
    server_name a.example.com;
}

server {
    listen 8080;
    server_name a.example.com;
}
```

这不是负载均衡方式。需要多后端时应使用 `upstream`。

### 直接使用 IP 访问时

例如：

```text
http://Nginx_IP:8080/
```

请求的 Host 是 IP，通常不会匹配业务域名，因此会落到该 `IP:port` 的默认 server：

- 有 `default_server` 时使用它；
- 没有时，一般使用该监听地址/端口的第一个 server。

建议显式配置兜底：

```nginx
server {
    listen 8080 default_server;
    server_name _;
    return 444;
}
```

---

## `http server` 与 `stream server` 的区别

是否属于 TCP/四层代理，不取决于配置文件目录名，而取决于配置所在上下文。

HTTP 七层代理：

```nginx
http {
    server {
        listen 8080;
        server_name a.example.com;

        location / {
            proxy_pass https://backend:443;
            proxy_set_header Host a.example.com;
        }
    }
}
```

具有 `server_name`、`location`、`proxy_set_header` 等指令的 `server` 是 HTTP server。

TCP 四层代理：

```nginx
stream {
    server {
        listen 443;
        proxy_pass backend:443;
    }
}
```

`stream` 不理解 HTTP URL、Header，不能使用 `location`、`proxy_set_header`、`proxy_redirect` 等 HTTP 指令。

即使目录叫 `tcp.d`，如果它是从 `http {}` 中 `include` 的，里面的配置仍然属于 HTTP 上下文。

---

## 后端 HTTPS 不代表入口 HTTPS

如下配置：

```nginx
server {
    listen 8080;

    location / {
        proxy_pass https://backend:443;
    }
}
```

链路是：

```text
浏览器 --HTTP:8080--> Nginx --HTTPS:443--> 后端
```

Nginx 的入口仍然是普通 HTTP，所以不需要服务端证书。

只有当 Nginx 自己作为 HTTPS 服务端，例如：

```nginx
listen 443 ssl;
```

或：

```nginx
listen 8080 ssl;
```

才需要给 Nginx 配置服务端证书和私钥：

```nginx
ssl_certificate     /path/server.crt;
ssl_certificate_key /path/server.key;
```

---

## SSO 跳转到 HTTPS/443 时的核心问题

如果登录按钮最终让浏览器访问：

```text
https://wsso.example.com/...
```

且 `hosts` 中：

```text
Nginx_IP wsso.example.com
```

那么浏览器一定会连接：

```text
Nginx_IP:443
```

因此 Nginx 必须能够接住 443。

### 方案 1：Nginx 终止 TLS，再做 HTTPS 反向代理

```nginx
server {
    listen 443 ssl;
    server_name wsso.example.com;

    ssl_certificate     /usr/local/nginx/cert/multi.crt;
    ssl_certificate_key /usr/local/nginx/cert/multi.key;

    location / {
        proxy_pass https://REAL_WSSO_IP:443;

        proxy_set_header Host wsso.example.com;
        proxy_set_header X-Forwarded-Proto https;

        proxy_ssl_server_name on;
        proxy_ssl_name wsso.example.com;
        proxy_ssl_verify off;
    }
}
```

其中 `proxy_ssl_server_name on` 和 `proxy_ssl_name` 对真实后端依赖 SNI 的场景很重要。

链路：

```text
浏览器 --HTTPS--> Nginx --HTTPS--> 真实 wsso
```

此方案可以继续使用 HTTP 层能力，例如修改 Header、路径、响应等。

### 方案 2：`stream` 做 TLS/TCP 透传

```nginx
stream {
    server {
        listen 443;
        proxy_pass REAL_WSSO_IP:443;
    }
}
```

链路：

```text
浏览器 --TLS--> Nginx(TCP透传) --> 真实 wsso
```

TLS 握手实际发生在浏览器和真实 wsso 之间，Nginx 不解密，因此 Nginx 不需要该域名的 cert/key。

限制：不能在 Nginx 上使用 `location`、`proxy_set_header`、`proxy_redirect` 等七层能力。

如果 443 需要按多个 HTTPS 域名分流，可进一步使用 `ssl_preread` + SNI 分流。

---

## 为什么 `proxy_redirect` 不一定解决登录按钮跳转

`proxy_redirect` 只修改上游 HTTP 响应中的 `Location` / `Refresh` 等重定向头，例如：

```http
Location: https://wsso.example.com/login
```

可改写为：

```nginx
proxy_redirect https://wsso.example.com/ http://wsso.example.com:8080/;
```

但是如果“点击登录”是页面里的链接、`form action` 或 JavaScript：

```html
<a href="https://wsso.example.com/login">...</a>
```

```html
<form action="https://wsso.example.com/login">
```

```javascript
window.location.href = "https://wsso.example.com/login";
```

则它不是 HTTP 302/Location 响应，`proxy_redirect` 不会生效。

虽然可以考虑 `sub_filter` 修改 HTML/JS 内容，但对 SSO 通常不推荐，因为 HTTP 化后可能继续遇到：

- `Secure` Cookie 不发送；
- `SameSite`；
- Origin / Referer 校验；
- 回调 URL 白名单；
- 登录循环。

因此，当外部页面固定跳 `https://wsso...` 时，更稳妥的方式是让 Nginx 正常承接 HTTPS/443，或直接四层透传。

---

## 自签名证书：一张证书覆盖多个域名

测试环境可生成一张带多个 SAN 的自签名证书。

示例 `openssl.cnf`：

```ini
[req]
default_bits       = 2048
prompt             = no
default_md         = sha256
distinguished_name = dn
x509_extensions    = v3_req

[dn]
CN = xinxipilu.example.com

[v3_req]
subjectAltName = @alt_names
keyUsage = critical, digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth

[alt_names]
DNS.1 = xinxipilu.example.com
DNS.2 = client-xinxipilu.example.com
DNS.3 = wsso.example.com
```

和单域名证书相比，真正决定多域名支持的是 SAN：

```ini
[alt_names]
DNS.1 = ...
DNS.2 = ...
DNS.3 = ...
```

`CN` 仍然只有一个值；现代浏览器主要依据 SAN 判断域名是否匹配。

生成：

```bash
openssl req -x509 \
  -nodes \
  -newkey rsa:2048 \
  -sha256 \
  -days 3650 \
  -keyout multi.key \
  -out multi.crt \
  -config openssl.cnf
```

检查 SAN：

```bash
openssl x509 -in multi.crt -noout -ext subjectAltName
```

同一张多 SAN 证书可以被多个 Nginx HTTPS `server` 共用。

自签名证书默认不被终端浏览器信任；测试环境需要在终端导入信任，或接受证书告警。不要把私钥提交到长期上下文或代码仓库。

---

## 当前场景的建议

现有访问方式中：

1. `xinxipilu` 继续通过 `HTTP:8080 -> Nginx -> 外部 HTTPS:443` 可以正常工作；
2. `wsso` 因页面主动跳转到 `https://wsso...`，终端会通过 hosts 连接 Nginx 的 443；
3. 因此需要让 Nginx 承接 `wsso:443`：
   - 需要七层改写能力时，用 `listen 443 ssl` + 证书；
   - 只需要透明通过时，可考虑 `stream` TCP 透传；
4. 对 SSO 不建议为了复用 8080 而强制把 HTTPS 改成 HTTP，容易受到 Cookie 和认证安全策略影响。
