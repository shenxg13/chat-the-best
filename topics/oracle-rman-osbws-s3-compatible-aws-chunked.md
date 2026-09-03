# Oracle RMAN + OSBWS + S3-compatible 存储：`aws-chunked` 恢复兼容性问题

## 背景

在 Oracle 11g RMAN 通过 Oracle Secure Backup Web Services Library（OSBWS）向第三方 S3-compatible COS 备份后，恢复控制文件时出现：

```text
ORA-19870: error while restoring backup piece ...
ORA-27190: skgfrd: sbtread2 returned error
ORA-19511: Error received from media manager layer, error text:
KBHS-00715: HTTP error occurred 'unsupported-algorithm'
```

该问题已通过抓包、RMAN debug、OSBWS `sbtio` trace 和对象 metadata 对比定位。

## 已确认结论

### 1. 问题发生在 OSBWS 的 `sbtread2` / HTTP 响应处理层

RMAN debug 显示：

```text
SYS.DBMS_BACKUP_RESTORE.RESTOREBACKUPPIECE
  -> sbtread2
  -> KBHS-00715: HTTP error occurred 'unsupported-algorithm'
```

SBT channel 本身能够正常分配，OSBWS library 能正常加载；数据库处于 `NOMOUNT` 也不是原因。

抓包和 `sbtio` trace 显示：

1. COS 对 chunk 的 GET 返回 `HTTP/1.1 200 OK`；
2. OSBWS 开始读取后主动中止下载；
3. 随后执行 `AbortDownload`；
4. OSBWS 内部 `nhpRespGet()` 返回 `unsupported-algorithm`；
5. 最终由 `sbtread2` 把错误传给 RMAN。

因此这不是 DNS、端口、防火墙、对象不存在、S3 鉴权失败或 RMAN 解析 controlfile 本身时报错。

### 2. 直接触发条件是对象保留了 `Content-Encoding: aws-chunked`

对比两类能够/不能恢复的 S3 chunk：

#### Windows + OSBWS 3.17.4.21 写入的对象

```text
Content-Encoding: aws-chunked
Content-Type: Application/Octet-Stream
```

恢复失败：

```text
KBHS-00715: HTTP error occurred 'unsupported-algorithm'
```

#### Linux + OSBWS 19.0.0.1 写入的对象

没有 `Content-Encoding: aws-chunked`，可正常通过 OSBWS 恢复。

关键验证：

- 将失败对象复制到测试位置；
- 仅删除复制品的 `Content-Encoding: aws-chunked`；
- 不修改 chunk 数据内容；
- 再次通过 RMAN + OSBWS 恢复控制文件；
- **恢复成功**。

因此当前环境中的直接故障触发条件已经可以确认：

> S3-compatible COS 持久化并在 GET/HEAD 时返回了 `Content-Encoding: aws-chunked`，OSBWS restore 端无法处理该 encoding，最终报 `unsupported-algorithm`。

### 3. 更准确的根因表述

不宜简单写成“OSBWS 3.x bug”。

`aws-chunked` 本身是 AWS SigV4 streaming upload 的正规机制。问题更准确地描述为：

> **OSBWS 3.17.4.21 与当前第三方 S3-compatible COS 在 SigV4 streaming / `aws-chunked` 处理上存在兼容性问题。**

标准 AWS S3 在完成 streaming upload 后，应正确处理 `aws-chunked`，不应把它作为最终对象的普通 `Content-Encoding` 持久化并在后续 GET 时继续返回。

当前 COS 保存了这一 header，导致 OSBWS 在恢复读取对象时进入不支持的 decoding/algorithm 分支。

### 4. 与 Oracle 11.2.0.3 / 11.2.0.4 的关系

恢复测试过程中曾怀疑源库 11.2.0.4、目标库 11.2.0.3 的版本差异。

最终证据表明：

- `unsupported-algorithm` 的直接原因不是 RMAN/数据库版本检查；
- 同一恢复端对 Linux 新版 OSBWS 写出的对象可以正常读；
- 对带 `Content-Encoding: aws-chunked` 的旧对象则失败；
- 删除该 header 后恢复成功。

因此对该错误本身，Oracle patchset 差异不是主因。

但独立于本问题，生产为 11.2.0.4 时，脱敏/恢复环境仍建议使用 11.2.0.4 Oracle Home，不建议把 11.2.0.4 的 RMAN 物理备份作为常规方案恢复到 11.2.0.3。

## OSBWS 对象结构观察

旧版 OSBWS 在 S3 中将一个逻辑 RMAN backup piece 保存为目录/前缀结构，例如：

```text
<backup-piece>/
  <incarnation>/
    0000000001
    0000000002
    ...
    metadata.xml
```

典型 metadata：

```text
ChunkSize = 104857600   # 100 MiB
ChunkIdFormat = %010d
ChunkType = FileChunk
```

因此大型 backup piece 会被拆成多个 100 MiB `file_chunk` 对象。

注意：单独下载 `0000000001` 后直接使用 RMAN `DEVICE TYPE DISK` 测试时曾出现 `invalid fileheader`，因此不能在未进一步验证前假设“多个 chunk 直接 `cat` 即可得到可用 RMAN backup piece”。当前首选恢复方式是保留 OSBWS 目录结构，只修正对象 header，然后继续通过 SBT/OSBWS restore。

## 当前可用恢复绕过方案

### 原则

**不要修改原始备份对象。**

推荐：

1. 把需要恢复的 backup piece 相关对象复制到恢复专用 bucket/prefix；
2. 保持原有 key/目录结构和 `metadata.xml`；
3. 对所有实际数据 chunk（`0000000001`、`0000000002`...）移除：

```text
Content-Encoding: aws-chunked
```

4. 检查对象大小没有变化；
5. 让 RMAN/OSBWS 指向测试/恢复 bucket；
6. 正常通过 SBT restore。

控制文件已用此方式验证成功。

### s3cmd 示例思路

先复制对象，再仅处理复制品，例如：

```bash
s3cmd modify \
  --remove-header=content-encoding \
  s3://<restore-bucket>/<file_chunk_path>/0000000001
```

执行后用：

```bash
s3cmd info s3://<restore-bucket>/<file_chunk_path>/0000000001
```

确认：

- `Content-Encoding: aws-chunked` 已消失；
- `Content-Length` 未变化；
- 其他 `x-amz-meta-*` 信息仍保留。

对于 datafile backup piece，需要对该 piece 下的所有 `000000000N` chunk 进行同样处理。

## 长期处理建议

### 1. 停止继续使用 OSBWS 3.17.4.21 向当前 COS 产生新备份

Linux 环境中 OSBWS 19.0.0.1 产生的对象未出现 `Content-Encoding: aws-chunked`，并可正常恢复。

建议验证 Windows 新版 OSBWS：

1. 安装/下载较新的 Windows 64-bit `oraosbws.dll`；
2. 产生一个新的 controlfile backup；
3. 用 `s3cmd info` 检查新 chunk 是否仍有 `Content-Encoding: aws-chunked`；
4. 做一次 restore 验证。

如果新版本写入对象不再带该 header，则可从源头解决新备份问题；历史旧备份继续采用“复制 + 去 header”兼容恢复。

### 2. OSBWS Windows 64-bit 获取方式

Oracle 官方的 Amazon S3 OSB Cloud Module 使用：

```text
osbws_install.jar
```

安装器支持指定目标 library 平台，例如：

```text
-libPlatform windows64
```

因此可以在一台可访问互联网的 Linux/Windows 临时机器上下载 Windows 64-bit library，不要求该机器本身安装 Oracle Database。

如果 installer 没有真正的 download-only 选项，则通常还需要：

- 能访问 Oracle 下载源；
- 一个可用的 S3 服务及有效测试凭据；
- 临时 `walletDir` / `configFile` / `libDir`；
- 完成后只把 `oraosbws.dll` 带回隔离现网；
- 不复用外网生成的 wallet 和 `.ora` 配置。

如需精确获取某一历史 OSBWS 版本（例如 `19.0.0.1`），优先保留/使用对应年代的 `osbws_install.jar`，或通过 My Oracle Support 查找对应 OS-specific Cloud Backup Module patch。

## 调试方法记录

### RMAN debug

RMAN debug 可以确认错误是否来自 `RESTOREBACKUPPIECE -> sbtread2`，但通常看不到 OSBWS 内部具体算法原因。

### OSBWS trace

可在 OSBWS 配置中临时开启：

```text
_OSB_WS_TRACE_LEVEL=100
```

然后检查 `sbtio.log`。

本问题中 trace 看到：

```text
nhpRespGet ... HTTP/1.1 200 OK
... unsupported-algorithm
Exiting sbtread2; retval=-1
```

说明错误发生在 OSBWS HTTP 响应处理阶段，而非 RMAN backup piece 解析阶段。

调试完成后应关闭 trace，避免长期产生大量日志；日志可能包含请求、凭据相关信息，对外分享前必须脱敏。

### 抓包

当 endpoint 使用 HTTP（非 HTTPS）时，可以直接通过 tcpdump/Wireshark 观察：

```text
GET chunk
<- HTTP 200 OK
客户端主动终止
AbortDownload
```

该行为与 OSBWS trace 相互印证。

## 一句话结论

> **旧版 Windows OSBWS 3.17.4.21 写入第三方 S3-compatible COS 时产生的 chunk 被持久化为 `Content-Encoding: aws-chunked`；COS 在读取时继续返回该 header，导致 OSBWS restore 的 HTTP 层报 `unsupported-algorithm`。复制备份对象并移除该 header 后，RMAN 控制文件恢复已验证成功。**
