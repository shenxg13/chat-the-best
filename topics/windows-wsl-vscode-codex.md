# Windows + WSL2 + VS Code + Codex 开发环境

## 当前结论

当前主力开发环境为：

```text
Windows 11
├── ChatGPT Native App
│   ├── Chat
│   ├── Work
│   └── Codex
├── VS Code（Windows 原生 GUI）
│   ├── Remote WSL
│   └── Codex Extension
└── WSL2 AlmaLinux
    ├── ~/code/tm-lab
    ├── Git
    ├── Python / uv
    ├── PostgreSQL
    └── Linux 开发工具链
```

不再简单把“VS Code + Codex Extension”视为唯一推荐入口。

当前更准确的状态是：

- **VS Code + Remote WSL + Codex Extension**：仍是最成熟的 IDE 型开发入口，适合人工频繁查看代码、调试、LSP/F12、终端和 Git 操作。
- **Windows Native App + Codex + WSL Agent Environment**：已经实测可以让 Codex Agent 真正运行在 WSL2 中，并直接以 `/home/sfmon/code/tm-lab` 为工作目录；适合以 Agent / Issue / 多会话为中心的工作流。
- **ChatGPT Work**：适合分析、调研、文档、报告、表格、演示等知识工作；可以处理授权的本地文件/目录，但当前不要把它理解成类似 Codex 的 WSL 开发 Agent。
- 当前尚未决定完全切换到 Native App，建议一段时间内并行使用 VS Code Codex 和 Native Codex，以真实 Issue 比较工作效率。

## 当前机器与 WSL 实例

主开发机：

- Windows 11
- ROG G635L
- 32GB 内存
- Windows 本地磁盘剩余空间 700GB+
- WSL2 使用 AlmaLinux 系发行版
- 当前实际开发实例名：`alma-tm`
- Linux 开发用户：`sfmon`
- 主要项目：`/home/sfmon/code/tm-lab`

查看 WSL：

```powershell
wsl -l -v
```

进入实例：

```powershell
wsl -d alma-tm
```

设置默认实例：

```powershell
wsl --set-default alma-tm
```

关闭整个 WSL：

```powershell
wsl --shutdown
```

如果 Native Codex 的 WSL 模式存在默认发行版选择问题，优先确保 `alma-tm` 前带 `*`，即它是 Windows 当前默认 WSL distro。

## WSL2 的角色

WSL2 可以近似理解为 Windows 自动管理的轻量 Linux 虚拟化环境：

```text
Windows
└── WSL2
    └── AlmaLinux
        ├── Linux 文件系统
        ├── PostgreSQL
        ├── Git / Python
        └── tm-lab
```

它不是 Codex 自身的安全沙箱。

更准确地说：

```text
WSL2 Linux 环境
└── Codex Agent
    └── Codex 自身的权限 / 审批边界
```

因此即使 Agent 在 WSL 内运行，也仍需要注意 `~/.ssh`、kubeconfig、Token、`/mnt/c` 等敏感资源。

## WSL 文件系统与数据放置

项目代码和 PostgreSQL 数据优先放在 Linux 文件系统：

```text
/home/sfmon/code/...
/var/lib/pgsql/...
```

Windows 文件系统通过 `/mnt/c` 等路径访问，适合：

- 原始数据文件
- 安装包
- 数据库备份
- Windows / WSL 共享文件

不建议把高频编译项目或 PostgreSQL 数据目录长期放在 `/mnt/c` 下。

WSL VHDX 是动态增长的；Linux 中看到约 1TB 的文件系统通常是逻辑上限，不代表 Windows 已预占 1TB。

数据库 POC 若频繁导入、删除、重建表和索引，可考虑 Sparse VHD，以减少 VHD 只增不减的问题。

## WSL 配置管理

新版 WSL 优先使用 WSL Settings 管理：

- CPU
- 内存
- Swap
- 网络模式
- Auto Proxy
- DNS Tunneling
- VHD 相关配置

发行版内部行为使用 `/etc/wsl.conf`，例如：

```ini
[boot]
systemd=true

[interop]
enabled=true
appendWindowsPath=true
```

修改全局 WSL 或 interop 配置后通常执行：

```powershell
wsl --shutdown
```

## Windows 本地代理

当前 Windows 本地代理端口为 `7890`。

推荐 WSL：

- Mirrored 网络
- Auto Proxy 开启
- DNS Tunneling 开启

此时 WSL 中可以直接测试：

```bash
curl -I -x http://127.0.0.1:7890 https://github.com
```

Shell 常用配置：

```bash
export http_proxy="http://127.0.0.1:7890"
export https_proxy="http://127.0.0.1:7890"
export HTTP_PROXY="$http_proxy"
export HTTPS_PROXY="$https_proxy"
export no_proxy="localhost,127.0.0.1,::1,.local"
export NO_PROXY="$no_proxy"
```

## WSL 备份 / 恢复点

WSL 没有 VMware 式增量快照树，可使用：

```powershell
wsl --terminate alma-tm
wsl --export alma-tm D:\WSL-Backup\alma-tm-base.vhdx --format vhd
```

适合建立恢复点的阶段：

```text
01-clean      系统刚安装
02-dev-base   Git、编译环境、代理完成
03-pg-base    PostgreSQL 和扩展完成
```

导入大规模 PostgreSQL 数据后，不宜频繁对整个 WSL 做完整 export；数据库数据使用数据库自身备份，代码使用 Git。

## VS Code Remote WSL

从 WSL 中进入项目：

```bash
cd ~/code/tm-lab
code .
```

实际结构：

```text
Windows
└── VS Code GUI
    └── Remote WSL
        └── VS Code Server（运行在 alma-tm）
            ├── 读取 Linux 项目文件
            ├── 运行终端
            ├── 调用 Git / Python / PostgreSQL
            └── 承载需要 Linux 环境的扩展
```

需要直接访问项目和 Linux 工具链的扩展应安装到 WSL 侧，包括：

- Codex
- Python
- Pylance
- Python Debugger
- Ruff
- 其他需要访问 WSL 环境的数据库 / Docker / Kubernetes 扩展

Python 解释器也应来自 WSL，例如：

```text
/usr/bin/python3
/home/sfmon/code/tm-lab/.venv/bin/python
```

## VS Code Codex Extension 的特点

优势：

- 与 IDE 文件树天然整合。
- LSP、F12、调试、pytest、Git diff 等能力就在同一个界面。
- Remote WSL 模式下，Codex 扩展和项目都直接处于 Linux 环境。
- 对需要大量人工查看代码、调试和命令行交互的任务非常顺手。

因此 VS Code 目前仍是可靠的主 IDE，不需要因为 Native App 出现就立即放弃。

## Windows Native App 中的 Codex + WSL

### 已确认：Native Codex 可以真正运行在 WSL

Windows Native App 的 Codex 支持把 Agent Environment 设为 WSL。

对当前环境，目标结构是：

```text
Windows ChatGPT Native App
└── Codex
    └── Agent Environment = WSL
        └── alma-tm
            └── /home/sfmon/code/tm-lab
```

项目在 Windows 文件选择器中可以通过类似路径选择：

```text
\\wsl.localhost\alma-tm\home\sfmon\code\tm-lab
```

关键点不是 UI 中是否能展开目录，而是 Agent 的实际执行环境。

### 当前实测结果

已经在 Native App 的 Codex Project 中添加 `tm-lab`，并由 Codex 自检确认：

- Linux 内核名称包含 `microsoft-standard-WSL2`。
- 当前工作目录为 `/home/sfmon/code/tm-lab`。
- `git rev-parse --show-toplevel` 与当前目录一致。
- 当前 Linux 用户为 `sfmon`。
- 可以读取仓库中的 `README.md`，并确认仓库为 TM Lab。

因此当前已经确认：

> **Native App 的 Codex Agent 确实运行在 `tm-lab` 所在的 WSL2 环境中，不只是通过 Windows 读取 `\\wsl.localhost` 文件。**

### Project 中只显示目录名是正常的

Native Codex Project 添加 `tm-lab` 后，界面中可能只显示一个项目 / 目录名，而不像 VS Code Explorer 那样展开完整文件树。

只要 Agent 能正常执行：

```bash
pwd
uname -a
whoami
git rev-parse --show-toplevel
ls -la
```

并读取仓库文件，就可以认为项目配置正确。

这反映了两个产品的界面设计差异：

```text
VS Code       -> 文件 / IDE 中心
Native Codex  -> Agent / 任务 / 会话中心
```

## Native Codex 与 VS Code Codex 的会话历史

当前不要假设两者的 Codex 对话历史互通。

实际已经观察到：

- VS Code Codex Extension 中已经存在 `tm-lab` 的对话。
- Native App 中添加同一个 WSL `tm-lab` Project 后，看不到这些历史对话。

即：

```text
同一个 repo
!=
同一个会话列表 / session store
```

原因可能与 Windows Native App 的 Codex 状态目录和 WSL 中 VS Code Extension 使用的 Codex 状态目录不同有关。

典型理解：

```text
VS Code Remote WSL
└── Codex Extension
    └── WSL 侧 Codex state / sessions

Windows Native App
└── Codex UI / state
    └── Windows 侧状态管理
        └── Agent execution 再进入 WSL
```

虽然 Codex 产品体系可能支持复用一部分配置和历史，但在当前 Windows + WSL 实际场景中，**不要依赖 Native App 自动显示 VS Code Extension 已有会话**。

也不建议为了共享聊天历史手工强行让 Windows 和 WSL 共用同一个 `CODEX_HOME`，避免 SQLite state、路径和锁等兼容问题。

比较稳妥的迁移方式是：

```text
旧 Issue / 已有上下文 -> 继续原来的 VS Code Codex 会话
新 Issue              -> 可以逐步尝试 Native App Codex
```

## Work 与 Codex 的边界

当前 Native App 中的 Work 和 Codex 不应混为一类 Agent。

### Work

更适合：

- 技术调研
- 架构分析
- 数据分析
- 报告
- 文档
- 表格
- PPT
- 其他最终交付物

Work 可以处理用户授权的本地文件 / 目录，但当前不要把它理解成“绑定到某个 WSL distro、直接作为 Linux 开发 Agent”。

例如：

```text
读取 tm-lab
+ Web 调研
+ 数据库架构分析
+ 生成设计评审报告
```

更适合 Work。

### Codex

更适合：

- repository
- terminal
- Git
- 修改代码
- 跑测试
- debug
- diff / review
- 一个 Issue 一个 Agent / 会话

例如：

```text
读取 Issue
-> 修改 tm-lab
-> pytest
-> git diff
-> 修复失败
```

更适合 Codex。

## Native App 与 VS Code 的定位判断

当前尚未决定完全切换哪一种方式。

可以按下面的标准比较：

### 更适合 VS Code 的场景

- 高频人工查看源码。
- 需要文件树、LSP、F12。
- 需要交互式 debug。
- 经常直接操作终端。
- 对 WSL 稳定性和 IDE 整合要求高。

### 更适合 Native Codex 的场景

- 一个 Issue 一个 Agent / 会话。
- 多个开发任务并行推进。
- 更关注任务 / 会话而不是文件树。
- 需要集中查看 Agent 任务和 diff。
- 希望把 AI 开发工作从 IDE 操作中解耦。

当前最可能的长期组合不是绝对二选一，而是：

```text
ChatGPT Native App
├── Chat  -> 日常讨论
├── Work  -> 调研 / 架构 / 报告 / 文档
└── Codex -> Issue / Agent / coding / tests

VS Code
└── 人工深入代码、调试、LSP、终端
```

如果 Native Codex 的 WSL 稳定性、多 Agent 和 Issue 工作流长期表现良好，它可以逐步承担更多 Codex 主入口；VS Code 仍保留为 IDE。

## tm-lab 当前开发环境方向

商标数据服务层继续放在 `tm-lab`，不另建独立 `trademark-data-service` 仓库。

当前结构：

```text
ROG G635L / Windows 11
└── WSL2 alma-tm
    ├── /home/sfmon/code/tm-lab
    ├── PostgreSQL 17
    ├── Python + uv
    ├── Git
    └── 后续按需加入 Docker
```

数据服务层后续逐步包含：

- FastAPI
- Application Service
- Domain Model
- Repository 抽象
- PostgreSQL 实现
- 后续 OpenSearch 实现
- 数据导入、标准化、搜索投影

POC 阶段优先使用本地 WSL + PostgreSQL；完整大数据量或长期性能验证时再考虑额外计算 / 存储资源。

## PostgreSQL 17 / PGDG 关键点

AlmaLinux 中通过 PGDG 使用 PostgreSQL 17：

```bash
sudo dnf -qy module disable postgresql
sudo /usr/pgsql-17/bin/postgresql-17-setup initdb
```

`dnf module disable postgresql` 用于避免 AlmaLinux 自带 module stream 与 PGDG 包冲突。

`postgresql-17-setup initdb` 主要负责初始化 PostgreSQL cluster，例如默认：

```text
/var/lib/pgsql/17/data
```

会创建系统 catalog、WAL、`postgres/template0/template1` 等，但不会启动服务、创建业务用户、导入业务数据或设置远程连接。

初始化后：

```bash
sudo systemctl enable --now postgresql-17
```

## 当前管理原则

```text
WSL Settings        -> 全局资源、网络、VHD
/etc/wsl.conf       -> 发行版行为、systemd、interop
AlmaLinux ext4      -> 项目代码、PostgreSQL、Linux 工具
Windows 文件系统    -> 原始数据、安装包、备份、共享文件
Git                 -> 代码版本管理
wsl --export        -> 关键环境恢复点
数据库自身备份       -> PostgreSQL 大规模数据
Mirrored + 7890     -> WSL 出网代理
VS Code Remote WSL  -> IDE 型开发入口
Native Codex + WSL  -> Agent / Issue 型开发入口
Work                -> 调研 / 分析 / 交付物
```

## 后续恢复上下文时的关键点

1. 主力系统是 `Windows 11 + WSL2 AlmaLinux`，当前实际开发实例为 `alma-tm`，Linux 用户为 `sfmon`。
2. `tm-lab` 位于 `/home/sfmon/code/tm-lab`，代码与 PostgreSQL 数据均优先留在 WSL Linux 文件系统。
3. VS Code GUI 在 Windows，Remote WSL / VS Code Server 在 Linux；Codex Extension 可直接在 WSL 中工作。
4. Windows Native App 的 Codex 已经实测可以真正进入 `alma-tm` 的 WSL2 环境，并以 `tm-lab` 为工作目录。
5. Native Codex Project 只显示项目名、不展示完整文件树并不代表失败；以 `pwd / uname / git root / ls` 验证实际环境。
6. Native Codex 与 VS Code Codex 当前不要假设聊天历史互通；同一 repo 下已有 VS Code 对话在 Native App 中可能不可见。
7. 不建议为了共享历史手工强行合并 Windows / WSL 的 `CODEX_HOME`。
8. Work 与 Codex 定位不同：Work 偏研究分析和交付物，Codex 偏 repo / terminal / coding / test。
9. 当前尚未决定完全从 VS Code 切换到 Native App；建议用真实 Issue 并行评估。
10. 最可能的长期形态是 Native App 负责 Chat / Work / 多 Agent Codex，VS Code 保留为深入编码和调试 IDE。
