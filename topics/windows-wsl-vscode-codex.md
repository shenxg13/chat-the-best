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

## 当前机器与环境

当前主开发机：

- Windows 11
- ROG G635L
- 32GB 内存
- Windows 本地磁盘剩余空间 700GB+
- WSL2 使用 AlmaLinux

当前阶段优先本地开发，不立即购买 ECS/CVM。

原因：

- 32GB 内存足够支撑 Python、PostgreSQL、Codex 和日常开发。
- 700GB+ 剩余空间足够完成项目开发和较大规模数据验证。
- 等需要完整约 300GB 商标数据、建立大量索引或做长期性能测试时，再评估云主机或额外存储。

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

Windows 原生 Codex App 可以作为补充，适合较大的独立任务、多线程 Agent、跨文件重构和 Review，但日常项目开发仍以 VS Code 为主。

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

### 推荐 `/etc/wsl.conf`

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

## tm-lab 当前开发环境方向

商标项目的数据服务层并入 `tm-lab` 仓库，不另建 `trademark-data-service` 仓库。

Windows 侧推荐最终形态：

```text
ROG G635L / Windows 11
├── VS Code
├── Codex 扩展
└── WSL2 AlmaLinux
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

## 后续恢复上下文时的关键点

如果新会话需要继续 Windows/Codex/tm-lab 环境工作，优先记住：

1. 主力环境是 `Windows 11 + WSL2 AlmaLinux`。
2. VS Code GUI 在 Windows，开发后端在 WSL。
3. 项目放在 WSL Linux 文件系统，不放 `/mnt/c`。
4. Codex、Python、Pylance 等项目扩展安装到 WSL。
5. 日常开发首选 VS Code Codex 扩展，Codex App 作为补充。
6. `tm-lab` 是商标数据与数据服务层的主仓库。
7. 当前本地硬件足以完成 POC，暂不需要立刻购买云服务器。
