# Windows + WSL2 + VS Code + Codex 开发环境

## 结论

Windows 开发环境优先采用：

```text
Windows 11
├── VS Code（Windows 原生 GUI）
│   ├── WSL 扩展
│   └── Codex 扩展
└── WSL2 AlmaLinux
    ├── VS Code Server
    ├── 项目代码
    ├── Git
    ├── Python / uv
    ├── PostgreSQL
    └── Codex 实际执行环境
```

核心原则：

- VS Code GUI 安装并运行在 Windows。
- 项目代码和开发工具链放在 WSL2 的 Linux 文件系统中。
- VS Code 通过 Remote WSL 连接 AlmaLinux，并在 WSL 中运行 VS Code Server。
- Codex、Python、Pylance、调试器等需要访问项目和执行命令的扩展，要安装到 WSL 侧。
- 不把项目长期放在 `/mnt/c/...` 下，优先使用 `/home/<user>/...`。
- WSL2 是 Linux 运行环境，不等同于 Codex 自身的安全沙箱；仍应控制 Codex 权限和敏感文件访问。

## 当前机器与 WSL 实例

当前主开发机：

- Windows 11
- ROG G635L
- 32GB 内存
- Windows 本地磁盘剩余空间 700GB+
- WSL2 发行版：`AlmaLinux-9`
- 当前 AlmaLinux 默认开发用户：`sfmon`

查看实例：

```powershell
wsl -l -v
```

当前典型状态：

```text
NAME           STATE      VERSION
* AlmaLinux-9  Stopped    2
```

进入发行版：

```powershell
wsl -d AlmaLinux-9
```

以 root 直接进入：

```powershell
wsl -d AlmaLinux-9 -u root
```

关闭单实例：

```powershell
wsl --terminate AlmaLinux-9
```

关闭整个 WSL：

```powershell
wsl --shutdown
```

当前阶段优先本地开发，不立即购买 ECS/CVM。32GB 内存和 700GB+ 剩余空间足以支撑 Python、PostgreSQL、Codex 和 POC；等需要完整约 300GB 商标数据、建立大量索引或做长期性能测试时，再评估云主机或额外存储。

## WSL2 的虚拟化模型

WSL2 可以近似理解为 Windows 自动管理的轻量 Hyper-V 虚拟化环境：

```text
Windows
└── Hyper-V / Virtual Machine Platform
    └── WSL 托管 Linux VM
        └── AlmaLinux-9 用户空间
```

它不是需要在 Hyper-V Manager 中手工维护的普通虚拟机。多个 WSL2 发行版共享 WSL 管理的底层 Linux 内核和虚拟化环境，但各自有独立的用户空间和 Linux 文件系统。

## WSL 配置管理

新版 WSL 已提供 **WSL Settings** 图形界面。CPU、内存、Swap、网络模式、自动代理、VHD 等全局参数优先在图形界面中维护，不需要为了这些设置额外创建 `.wslconfig`，也不建议同时维护两套配置。

需要区分：

- WSL Settings / `.wslconfig`：控制整个 WSL2 环境的 CPU、内存、Swap、网络、VHD 等全局参数。
- `/etc/wsl.conf`：控制某个发行版内部的默认用户、systemd、Windows 磁盘挂载和 interop 等行为。

修改 WSL 全局配置后通常执行：

```powershell
wsl --shutdown
```

再重新启动发行版。

## AlmaLinux 默认用户与 root

通过 `wsl --install` / Store 正常安装发行版时，第一次启动通常会要求创建 Linux 用户名和密码，该用户随后成为默认登录用户。当前 `sfmon` 很可能就是首次初始化 AlmaLinux 时创建的默认用户。

确认方式：

```bash
whoami
getent passwd sfmon
cat /etc/wsl.conf
```

日常提升到 root 推荐：

```bash
sudo -i
```

`su -` 需要 root 自己的密码；WSL 中 root 常常没有设置独立密码，因此日常更适合 `sfmon + sudo`。

## WSL 文件系统与数据放置

典型 `df -h` 中最重要的是：

```text
/dev/sdX  -> AlmaLinux 自己的 Linux 根文件系统 /
/mnt/c    -> Windows C: 盘
```

当前曾看到：

```text
/dev/sdd  1007G  ...  /
C:\       925G   ...  /mnt/c
```

`/dev/sdd` 对应 WSL 管理的 ext4 VHDX。以下目录都位于 Linux 文件系统：

```text
/etc
/home
/var
/opt
/root
```

例如项目代码和 PostgreSQL 数据应优先放在：

```text
/home/sfmon/code/...
/var/lib/pgsql/...
```

`/mnt/c` 是 Windows 文件系统映射，适合放：

- 原始数据文件
- 安装包
- 数据库备份
- Windows / WSL 之间共享的数据

不建议把 PostgreSQL 数据目录或高频编译的源码仓库长期放在 `/mnt/c` 下，因为 WSL 自己的 ext4 文件系统在性能和 Linux 权限语义方面更合适。

`/run`、`/dev`、`/mnt/wsl`、`/mnt/wslg`、`/run/user/*` 等多数属于内存文件系统、运行时挂载或 WSL 注入的虚拟文件系统，不是独立磁盘分区。

## VHD 容量与 Sparse

WSL 根文件系统看到约 1TB：

```text
/dev/sdd  1007G ... /
```

这里的 1TB 是 **逻辑最大容量**，不是 Windows 已经预占了 1TB。VHDX 会随实际写入动态增长。

如果要改变新发行版的默认文件系统最大容量，可以先在 WSL Settings 中修改 File System / VHD 相关设置，再创建发行版。已有发行版不会因为修改默认值自动缩小；现有 ext4 VHD 真要缩容，最稳妥的方式通常是重建发行版，而不是直接在线缩小。

### Sparse VHD

`sparse` 可以近似理解为存储里的 thin provisioning：

- 逻辑容量可以很大；
- 物理空间按实际写入分配；
- Linux 删除数据后，可以借助 discard/TRIM 等机制更容易把空闲块归还给 Windows。

普通动态 VHDX 本身已经具有按需增长的特点；Sparse VHD 更重要的价值是减少“VHD 只增不减”的情况。因此数据库 POC 中如果会频繁导入、删除、重建表和索引，建议启用 Sparse。

## 删除并重建 AlmaLinux

如果发行版基本没有需要保留的配置，可以直接注销，不必先 export：

```powershell
wsl --shutdown
wsl --unregister AlmaLinux-9
wsl --list --online
wsl --install -d AlmaLinux-9
```

`wsl --unregister` 会永久删除该发行版中的用户、软件、配置、数据和对应 VHDX。

如果只是刚初始化、尚未形成重要环境，重建通常比尝试对现有 VHD 做复杂缩容更简单可靠。

## WSL 快照 / 备份

WSL 没有 VMware / 普通 Hyper-V VM 那种增量在线快照树，但可以通过 `wsl --export` 做离线恢复点。

例如导出 VHDX：

```powershell
wsl --terminate AlmaLinux-9
wsl --export AlmaLinux-9 D:\WSL-Backup\AlmaLinux-9-base.vhdx --format vhd
```

也可以导出 tar。

适合做恢复点的阶段：

```text
01-clean      AlmaLinux 刚安装完成
02-dev-base   Git、编译工具、代理等完成
03-pg-base    PostgreSQL 和扩展安装完成
```

导入几百 GB 数据后不宜频繁做完整 WSL export。大型 PostgreSQL 数据应采用数据库自己的备份策略；代码交给 Git；原始数据和备份可放 Windows 数据盘。

## Windows 本地 7890 代理

目标是让 AlmaLinux 通过 Windows 本机 `7890` 端口访问互联网。

推荐新版 WSL 配置：

- 网络模式：Mirrored
- Auto Proxy：开启
- DNS Tunneling：开启

镜像网络模式下，WSL 可以直接访问 Windows 的 `127.0.0.1` 服务，因此代理地址可使用：

```text
http://127.0.0.1:7890
```

基础验证：

```bash
curl -I -x http://127.0.0.1:7890 https://github.com
```

Shell 中可以配置：

```bash
export http_proxy="http://127.0.0.1:7890"
export https_proxy="http://127.0.0.1:7890"
export HTTP_PROXY="$http_proxy"
export HTTPS_PROXY="$https_proxy"
export no_proxy="localhost,127.0.0.1,::1,.local"
export NO_PROXY="$no_proxy"
```

即使目标站点是 HTTPS，`https_proxy` 通常仍写 `http://127.0.0.1:7890`，因为这里描述的是连接 HTTP 代理并通过 CONNECT 建立隧道。

DNF 若无法继承当前用户代理，可先测试：

```bash
sudo -E dnf makecache
```

必要时再在 `/etc/dnf/dnf.conf` 中配置：

```ini
proxy=http://127.0.0.1:7890
```

## 终端连接 WSL

本机使用 WSL 不需要 SSH，推荐直接：

```powershell
wsl -d AlmaLinux-9
```

Windows Terminal、WezTerm、Tabby 等都可以把下面命令配置为一个 profile：

```text
C:\Windows\System32\wsl.exe -d AlmaLinux-9
```

如果需要模拟真实远程 Linux 服务器，也可以在 AlmaLinux 中启动 `sshd`，再用 SSH 客户端连接，但这不是本地日常使用 WSL 的必要条件。

## VS Code Remote WSL 的工作方式

在 WSL 中进入项目目录后：

```bash
cd ~/code/tm-lab
code .
```

实际架构是：

```text
Windows
└── Code.exe
    │
    └── Remote WSL
        │
        ▼
WSL2 AlmaLinux
└── VS Code Server
    ├── 访问 Linux 项目文件
    ├── 运行终端
    ├── 调用 Python / Git / gcc 等工具
    ├── 运行语言服务
    └── 承载需要在远端运行的 VS Code 扩展
```

`VS Code Server` 可以理解为 VS Code 在远程 Linux 环境中的后端。Windows 负责图形界面，WSL 负责真正访问文件和执行开发工具。

Remote SSH 到云主机时也是同样的模型，只是把 WSL 换成远程 Linux 服务器。

## 扩展安装位置

### 只需要安装在 Windows 的扩展

主要是纯界面类扩展，例如：

- WSL / Remote WSL 连接能力
- 主题
- 图标主题
- 语言包
- 纯 UI 类插件

### 需要安装到 WSL 的扩展

凡是需要读取代码、调用 Linux 工具或参与调试的扩展，应安装在 `WSL: AlmaLinux`：

- Codex
- Python
- Pylance
- Python Debugger
- Ruff
- 需要直接访问 WSL 环境的数据库、Docker、Kubernetes 等扩展

判断标准：在扩展详情页确认显示类似：

```text
Installed in WSL: AlmaLinux
```

Python 解释器也应来自 Linux，例如：

```text
/usr/bin/python3
/home/<user>/code/tm-lab/.venv/bin/python
```

而不是：

```text
C:\Users\...
```

## Codex 的推荐使用方式

Windows 上日常开发优先：

```text
VS Code + Codex 扩展 + WSL2
```

原因：

- IDE 导航、LSP、F12、调试、pytest、Git diff 等能力集中在 VS Code。
- Codex 可以直接理解当前打开的项目和文件。
- Codex 实际执行命令和修改文件发生在 WSL Linux 环境。
- 项目未来迁移到远程 Linux 主机时，Remote SSH 的使用方式几乎一致。

Windows 原生 Codex App / ChatGPT Work 可以作为补充，适合独立任务、跨文件操作、Review 或其他工作流；日常针对 WSL 内项目开发仍优先使用 VS Code + Codex 扩展。

## WSL 与 Codex 沙箱的边界

不要把 WSL 本身理解成 Codex 安全沙箱。

更准确的结构是：

```text
Windows
└── WSL2 Linux 环境
    └── VS Code Server / Codex Agent
        └── Codex 自身的权限和审批边界
```

如果 Codex 获得较高权限，它仍可能访问：

- WSL 用户可访问的其他目录
- `/mnt/c` 下的 Windows 文件
- `~/.ssh`
- kubeconfig
- 其他开发凭据

因此建议：

- 项目独立放在 `~/code/...`。
- 密码、Token、私钥不要提交到仓库。
- 不长期授予无限制执行权限。
- 对敏感目录和高风险命令保持审批。

## AlmaLinux 中无法执行 `code .` 的问题

曾遇到：

```text
/mnt/c/Users/.../Microsoft VS Code/bin/code: line 62:
.../Code.exe: cannot execute binary file: Exec format error
```

这通常表示 WSL 的 Windows interop 没有正常工作，而不是 VS Code 本身损坏。

推荐 `/etc/wsl.conf`：

```ini
[boot]
systemd=true

[interop]
enabled=true
appendWindowsPath=true
```

修改后，在 Windows PowerShell 中执行：

```powershell
wsl --shutdown
```

重新进入 AlmaLinux，先测试：

```bash
cmd.exe /c echo WSL-Interop-OK
```

再执行：

```bash
cd ~/code/tm-lab
code .
```

如果 Windows 可执行程序可以从 AlmaLinux 正常启动，则 `code .` 会调用 Windows 的 VS Code GUI，并自动连接到 WSL。

## Git 分支显示

查看当前分支：

```bash
git branch --show-current
```

或：

```bash
git status -sb
```

如果希望 Bash 提示符长期显示当前分支，可以在 `~/.bashrc` 中增加：

```bash
parse_git_branch() {
    git branch --show-current 2>/dev/null
}

export PS1='[\u@\h \W$([[ -n "$(parse_git_branch)" ]] && echo " ($(parse_git_branch))")]\$ '
```

加载：

```bash
source ~/.bashrc
```

效果类似：

```text
[sfmon@ab13 tm-lab (main)]$
```

VS Code 左下角状态栏也会显示当前 Git 分支。

## PostgreSQL 17 / PGDG 初始化

WSL AlmaLinux 中使用 PGDG RPM 安装 PostgreSQL 17 时，常见步骤：

```bash
sudo dnf -qy module disable postgresql
sudo /usr/pgsql-17/bin/postgresql-17-setup initdb
```

### `dnf module disable postgresql`

用于禁用 AlmaLinux 自带的 PostgreSQL module stream，避免与 PGDG 官方仓库提供的 PostgreSQL 包发生版本选择冲突。

首次执行时如果反复看到同一个 PGDG GPG key：

```text
0x08B40D20
PostgreSQL RPM Repository
```

被多次导入，不一定代表命令死循环；PGDG repo 包可能配置多个版本仓库，每个仓库首次刷新时都会触发同一 key 的检查。排障时不要加 `-q`，可以检查：

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

## tm-lab 当前开发环境方向

商标项目的数据服务层并入 `tm-lab` 仓库，不另建 `trademark-data-service` 仓库。

Windows 侧推荐最终形态：

```text
ROG G635L / Windows 11
├── VS Code
├── Codex 扩展
└── WSL2 AlmaLinux-9
    ├── ~/code/tm-lab
    ├── PostgreSQL 17
    ├── Python + uv
    ├── Git
    └── 后续按需加入 Docker
```

数据服务层建议在 `tm-lab` 中逐步包含：

- FastAPI
- Application Service
- Domain Model
- Repository 抽象
- PostgreSQL 实现
- 后续 OpenSearch 实现
- 数据导入、标准化、搜索投影

POC 阶段优先使用本地 WSL + PostgreSQL；完整大数据量或长期性能验证时再考虑 ECS/CVM。

## 当前管理原则

```text
WSL Settings        -> 管理全局资源、网络和 VHD
/etc/wsl.conf       -> 管理 AlmaLinux 自身的 WSL 行为
AlmaLinux ext4      -> 项目代码、PostgreSQL 数据、Linux 工具
Windows 文件系统    -> 原始数据、安装包、备份、跨系统共享
Git                 -> 管理代码
wsl --export        -> 关键环境节点恢复点
数据库自身备份       -> 管理大规模 PostgreSQL 数据
Mirrored + 7890     -> WSL 出网代理
VS Code Remote WSL  -> 主要开发入口
```

## 后续恢复上下文时的关键点

如果新会话需要继续 Windows / WSL / Codex / tm-lab 环境工作，优先记住：

1. 主力环境是 `Windows 11 + WSL2 AlmaLinux-9`，默认开发用户为 `sfmon`。
2. VS Code GUI 在 Windows，开发后端在 WSL。
3. 项目代码和 PostgreSQL 数据放在 WSL Linux 文件系统，不放 `/mnt/c`。
4. Codex、Python、Pylance 等项目扩展安装到 WSL。
5. 日常开发首选 VS Code + Codex 扩展；原生 App / Work 作为补充。
6. WSL 全局资源、网络、VHD 优先通过 WSL Settings 管理；发行版特有行为由 `/etc/wsl.conf` 管理。
7. WSL 使用 Mirrored 网络时，本机代理可以使用 `127.0.0.1:7890`。
8. Sparse VHD 适合数据库 POC 的频繁写入/删除场景；1TB `df` 显示只是逻辑上限。
9. WSL 没有传统快照树，关键节点使用 `wsl --export`；大型 PostgreSQL 数据使用数据库自身备份。
10. `tm-lab` 是商标数据与数据服务层的主仓库，本地硬件足以完成 POC。
