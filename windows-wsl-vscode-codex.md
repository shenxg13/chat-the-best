# Windows / WSL / VS Code / Codex

## 当前环境

- Windows 11 上使用 WSL 2。
- 当前发行版名称：`AlmaLinux-9`。
- `wsl -l -v` 显示版本为 WSL 2。
- AlmaLinux 作为本地开发/POC 环境使用，后续可能承载 PostgreSQL、项目代码、VS Code Remote WSL / Codex 等工具。

## WSL 配置管理

新版 WSL 已提供 **WSL Settings** 图形界面。CPU、内存、Swap、网络模式、自动代理、VHD 相关参数优先在图形界面中维护，不需要再额外创建 `.wslconfig`；避免同时维护两套全局配置。

需要区分：

- WSL Settings / `.wslconfig`：控制整个 WSL 2 虚拟化环境的资源、网络、VHD 等全局参数。
- `/etc/wsl.conf`：控制某个 Linux 发行版内部的默认用户、systemd、挂载、interop 等行为。

WSL 2 底层可近似理解为由 Windows 自动管理的轻量 Hyper-V 虚拟机环境；发行版不是需要在 Hyper-V Manager 中手工维护的普通 VM。多个 WSL 2 发行版共享 WSL 管理的底层虚拟化环境和 Linux 内核，但各自有独立的 Linux 用户空间和文件系统。

## AlmaLinux 默认用户与 root

正常通过 `wsl --install` / Store 安装发行版后，第一次启动通常会要求创建 Linux 用户名和密码，该用户会成为发行版默认登录用户。

当前 AlmaLinux 中存在 `sfmon` 用户；如果它是首次启动时创建的，那么它就是默认用户。可通过以下命令确认：

```bash
whoami
getent passwd sfmon
cat /etc/wsl.conf
```

进入 root 推荐使用：

```bash
sudo -i
```

也可以从 Windows 直接以 root 启动：

```powershell
wsl -d AlmaLinux-9 -u root
```

`su -` 需要 root 自己的密码；WSL 中 root 常常未设置独立密码，因此日常更适合 `sfmon + sudo`。

## WSL 文件系统

典型 `df -h` 中最重要的两类文件系统：

```text
/dev/sdX  -> AlmaLinux 自己的 Linux 根文件系统 /
/mnt/c    -> Windows C: 盘
```

例如：

```text
/dev/sdd  ~1TB  /
C:\       ...   /mnt/c
```

`/dev/sdd` 对应 WSL 管理的 ext4 VHDX。`/etc`、`/home`、`/var`、`/opt`、PostgreSQL 数据目录等都位于这里。

`/mnt/c` 是 Windows 文件系统映射。适合放原始数据、安装包、备份和 Windows/WSL 共享文件，但 PostgreSQL 数据目录、频繁编译的代码仓库等应优先放在 WSL 自己的 ext4 文件系统中，以获得更好的性能和 Linux 权限语义。

`/run`、`/dev`、`/mnt/wsl`、`/mnt/wslg`、`/run/user/*` 等多数属于内存文件系统、运行时挂载或 WSL 注入的虚拟文件系统，不是独立磁盘分区。

## VHD 容量与 Sparse

WSL 根文件系统中看到约 1TB，例如：

```text
/dev/sdd  1007G ... /
```

这里的 1TB 是 **逻辑最大容量**，不是 Windows 已经预占了 1TB 空间。WSL 的 VHDX 会随实际写入动态增长。

如果要改变新发行版的默认文件系统最大容量，可在 WSL Settings 中修改 File System / VHD 相关设置后再创建发行版。已有发行版不会因为修改默认值自动缩小；已有 VHD 真要缩容，最稳妥的方式通常是导出/重建，而不是直接在线缩小 ext4。

### Sparse VHD

`sparse` 可以近似理解为存储中的 thin provisioning：

- 逻辑容量可以很大；
- 物理空间按实际写入分配；
- Linux 删除数据后，可以借助 discard/TRIM 等机制更容易把空闲块归还给 Windows。

普通动态 VHDX 本身已经具有“按需增长”的特点；Sparse VHD 更重要的价值是减少“VHD 只增不减”的情况，因此适合数据库 POC 中频繁导入、删除、重建数据和索引的场景。

## 删除并重建 AlmaLinux

如果发行版基本没有需要保留的配置，可直接注销，不必先 export：

```powershell
wsl --shutdown
wsl --unregister AlmaLinux-9
wsl --list --online
wsl --install -d AlmaLinux-9
```

`wsl --unregister` 会永久删除该发行版中的用户、软件、配置、数据以及对应 VHDX。

## WSL 快照 / 备份

WSL 没有 VMware/Hyper-V 普通 VM 那种增量在线快照树，但可以通过 `wsl --export` 做离线恢复点。

关键环境节点可以：

```powershell
wsl --terminate AlmaLinux-9
wsl --export AlmaLinux-9 D:\WSL-Backup\AlmaLinux-9-base.vhdx --format vhd
```

也可导出 tar。

建议用途：

- AlmaLinux 刚安装好：做一次基础恢复点；
- 开发工具、代理、PostgreSQL 安装完成：再做一次恢复点；
- 导入几百 GB 数据后不宜频繁做完整 WSL export，数据库数据应使用自己的备份策略。

## Windows 本地 7890 代理

目标：AlmaLinux 通过 Windows 本机 `7890` 端口访问互联网。

推荐使用新版 WSL 的：

- 网络模式：Mirrored；
- Auto Proxy：开启；
- DNS Tunneling：开启。

镜像网络模式下，WSL 可直接访问 Windows 的 `127.0.0.1` 服务，因此代理地址可使用：

```text
http://127.0.0.1:7890
```

基础验证：

```bash
curl -I -x http://127.0.0.1:7890 https://github.com
```

Shell 可配置：

```bash
export http_proxy="http://127.0.0.1:7890"
export https_proxy="http://127.0.0.1:7890"
export HTTP_PROXY="$http_proxy"
export HTTPS_PROXY="$https_proxy"
export no_proxy="localhost,127.0.0.1,::1,.local"
export NO_PROXY="$no_proxy"
```

即使目标站点是 HTTPS，`https_proxy` 通常仍写 `http://127.0.0.1:7890`，因为这里描述的是到代理服务器的 HTTP CONNECT 连接。

DNF 若无法继承用户代理，可先测试：

```bash
sudo -E dnf makecache
```

必要时再在 `/etc/dnf/dnf.conf` 中配置：

```ini
proxy=http://127.0.0.1:7890
```

## 终端连接方式

本机使用 WSL 不需要 SSH，推荐：

```powershell
wsl -d AlmaLinux-9
```

Windows Terminal、WezTerm、Tabby 等都可以直接把 `wsl.exe -d AlmaLinux-9` 配成 profile。

如果需要模拟真实远程 Linux 服务器，也可以在 AlmaLinux 中安装并启动 `sshd`，然后通过 `ssh sfmon@localhost` 或指定端口连接，但这不是本机日常访问 WSL 的必要条件。

代码开发仍优先考虑 VS Code Remote - WSL，让编辑器和 Codex 的执行环境直接位于 Linux 文件系统中。

## PostgreSQL 17 / PGDG 初始化

使用 PGDG RPM 安装 PostgreSQL 17 时，常见步骤：

```bash
sudo dnf -qy module disable postgresql
sudo /usr/pgsql-17/bin/postgresql-17-setup initdb
```

### `dnf module disable postgresql`

用于禁用 AlmaLinux 自带的 PostgreSQL module stream，避免它与 PGDG 官方仓库提供的 PostgreSQL 包发生版本选择冲突。

如果首次执行时反复看到同一个 PGDG GPG key：

```text
0x08B40D20
PostgreSQL RPM Repository
```

被多次导入，不一定代表命令死循环；PGDG repo 包可能配置多个版本仓库，每个仓库首次刷新时都触发同一 key 的检查。排障时不要加 `-q`，可查看：

```bash
rpm -qi gpg-pubkey-08b40d20
sudo rpm --import /etc/pki/rpm-gpg/PGDG-RPM-GPG-KEY-RHEL
sudo dnf module disable postgresql -y
dnf repolist --all | grep -i pgdg
```

### `postgresql-17-setup initdb`

该命令不会启动 PostgreSQL，而是初始化一个新的 PostgreSQL 17 database cluster。PGDG 默认数据目录通常为：

```text
/var/lib/pgsql/17/data
```

主要完成：

- 创建数据目录并设置 `postgres:postgres` 权限；
- 调用 `initdb` 建立 PostgreSQL 内部目录结构；
- 创建 `postgres`、`template0`、`template1` 三个初始数据库；
- 创建 PostgreSQL 超级用户角色 `postgres`；
- 生成 `postgresql.conf`、`pg_hba.conf`、`pg_ident.conf`；
- 根据系统 locale 初始化 encoding / locale；
- 创建 `PG_VERSION`；
- 初始化系统 catalog、WAL、事务状态等内部结构。

它不会：

- 启动 PostgreSQL；
- 设置开机启动；
- 创建业务数据库/业务用户；
- 设置 `postgres` 密码；
- 开放远程连接；
- 导入业务数据。

初始化后通常再执行：

```bash
sudo systemctl enable --now postgresql-17
sudo systemctl status postgresql-17
```

## 当前建议的管理原则

```text
WSL Settings        -> 管理全局资源、网络和 VHD
AlmaLinux ext4      -> 项目代码、PostgreSQL 数据、Linux 工具
Windows 文件系统    -> 原始数据、安装包、备份、跨系统共享
Git                 -> 管理代码
wsl --export        -> 关键环境节点恢复点
数据库自身备份       -> 管理大规模 PostgreSQL 数据
Mirrored + 7890     -> WSL 出网代理
VS Code Remote WSL  -> 主要开发入口
```
