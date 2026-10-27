# Windows + WSL2 + Claude 桌面 App（Code）与手机 Remote Control

更新：2026-10-27

## 当前结论

- Claude 桌面 App 的 Code 标签页可以直接在 WSL2 发行版里运行会话：会话、工具和 Git 都在发行版内执行，工作目录是 Linux 路径。
- 只用桌面 App 时，**WSL 内不需要单独安装 Claude CLI**。App 会把自带的 CLI 部署到发行版的 `~/.claude/remote/`，由 App 自己管理和升级。
- 手机接续会话：打开桌面 App 的 **Settings > Claude Code > Connect new sessions to Remote Control** 即可，按会话接入。2026-10-27 决定保持这种方式，不采用 `claude rc` server 模式。
- 手机推送需要在**会话所在发行版**的 `~/.claude/settings.json` 打开两个开关，每个 WSL 发行版各配一次。2026-10-27 已实测能收到推送。

## 运行结构

```text
Windows 11
└── Claude 桌面 App（Code 标签页）
    └── WSL 会话（环境选择器 → WSL → 发行版 + 目录）
        └── alma-<项目>
            ├── ~/.claude/remote/srv/...          App 部署的服务
            ├── ~/.claude/remote/ccd-cli/<版本>    App 自带 CLI，实际执行会话
            └── ~/.claude/                         设置、记忆、凭据
```

- 同一发行版里的 VS Code Claude 扩展也用 `~/.claude/`，所以设置、权限规则和按项目路径存放的记忆能从 VS Code 延续到桌面 App。
- `~/.claude/remote/` 是 App 内部目录，目录名带版本号。不要修改它，也不要让脚本依赖这个路径。
- 在终端里单独运行 App 自带的 CLI 时没有登录态（提示 `You must be logged in to use Remote Control`）。要在 WSL 终端里用命令行版，需要另装独立 CLI，再执行 `claude auth login`。
- WSL 会话读的是发行版内的 `~/.claude/settings.json`。在 App 里用 `/config` 改的设置不一定写到这里，改完要在 WSL 里核对这个文件，并新开会话才生效。

## 与 VS Code 并行期间

- VS Code 扩展和桌面 App 的会话不要同时修改同一个 checkout；需要并行时用独立的 git worktree。
- 桌面 App 自带的 worktree 默认建在 `<repo>/.claude/worktrees/`。项目有自己的 worktree 约定时（例如 agent-harness 要求 `<repo>-issue-N` 放在仓库旁边），先按项目约定建好 worktree，再在 App 里打开该目录新开会话。
- agent-harness 的主流程（Issue 生命周期、`status:*` 标签、worktree、联网限制、质量检查）与客户端无关，切换到桌面 App 不需要适配。
- 例外是可选的 decision-review（debate）脚本：查找 CLI 的顺序是 `CLAUDE_BIN`／`CODEX_BIN` → VS Code 扩展目录 → `PATH`，找不到桌面 App 的 CLI。只有用到 debate 才需要处理，可以运行时显式传 `*_BIN`，或在模板上游调整查找顺序。当前项目不用 debate，暂不处理。
- 用 Codex 桌面版前，先确认它在 WSL 里执行：`uname -a; pwd; command -v bash flock gh`。

## Remote Control（手机或 claude.ai/code 接续会话）

Remote Control 只能按会话接入，没有"把整台设备加入"的开关。

| 方式 | 范围 | 要点 |
| --- | --- | --- |
| 桌面 App 的 Connect new sessions to Remote Control | 本机桌面 App 新开的所有会话，包括 WSL 会话 | 由 App 控制，不需要每个发行版配置。手机只能接续已开的会话，不能在手机上选目录新开 |
| 会话的 Remote Control 开关，或 `/remote-control`（`/rc`） | 单个会话 | 手动接入或断开 |
| 手机 Code 页 Add device → `claude rc`，即 `claude remote-control` server 模式 | 运行它的那个目录 | 需要独立 CLI 和登录，终端要一直开着。`--spawn same-dir/worktree/session`，默认最多 32 个并发会话。断网约 10 分钟后进程退出。这些会话不在桌面 App 侧栏 |
| Dispatch（通过 Cowork 配对手机与桌面 App） | 最接近整机 | 部分 Pro/Max 账号限量 beta。文档没说明能否在 WSL 里开 Code 会话 |

- 当前选择：只用自动接入，在电脑上开好会话，出门用手机接着推进；不装独立 CLI，也不跑 `claude rc`。
- 不想让某个会话出现在手机上：关掉它的 Remote Control 开关，或者归档。

## 手机推送

在会话所在发行版的 `~/.claude/settings.json` 中：

```json
{
  "agentPushNotifEnabled": true,
  "inputNeededNotifEnabled": true
}
```

- `agentPushNotifEnabled`：Claude 自己判断需要时推送，例如长任务完成。
- `inputNeededNotifEnabled`：Claude 等你操作时推送，例如权限确认、选择题。本次实测只开前者时，选择题没有推送；补开后者、新开会话并离开电脑后，收到了推送。
- **在场时不推送**：你正在电脑上看这个会话时，推送会被跳过（工具返回 `this terminal is active`）。测试时要锁屏，或至少把焦点从桌面 App 切走。
- 修改配置后要新开会话才生效。
- iOS 的专注模式和定时推送摘要会拦截或延迟推送。在手机上打开一次 Claude App 可以刷新推送标识。

## 每个项目一个 WSL 发行版时

当前做法是每个项目单独一个发行版，例如 `alma-tm`、`alma-<项目>`。

- 每个发行版有自己的 `~/.claude/`：推送开关、模型和 effort 等用户设置、按项目存放的记忆，都不跨发行版。
- Remote Control 自动接入由桌面 App 控制，不用逐个配置。
- 新发行版初始化时设置推送开关（需要 `jq`，AlmaLinux 上用 `sudo dnf install -y jq`；不会覆盖已有设置）：

```bash
mkdir -p ~/.claude && f=~/.claude/settings.json && { [ -s "$f" ] || echo '{}' > "$f"; } && jq '.agentPushNotifEnabled=true | .inputNeededNotifEnabled=true' "$f" > "$f.tmp" && mv "$f.tmp" "$f"
```

- 如果发行版是用 `wsl --export`／`wsl --import` 从基础镜像创建的，在基础镜像里设一次即可。
- 不建议用软链接让多个发行版共用一个 `settings.json`：Claude 改写配置时可能把软链接换成普通文件，共享会悄悄失效。

## 后续恢复上下文时的关键点

1. 桌面 App 的 WSL 会话由 App 部署到 `~/.claude/remote/` 的 CLI 执行，WSL 内不需要另装 Claude CLI。
2. 同一发行版里，VS Code 扩展和桌面 App 共用 `~/.claude/`（设置、记忆）；两边不要同时改同一个 checkout。
3. Remote Control 按会话接入；桌面 App 的自动接入开关已开，手机接续已开的会话。
4. 手机 Add device 的 `claude rc` 是按目录的 server 模式，需要独立 CLI 和常驻终端，当前不用。
5. 推送要在每个发行版的 `~/.claude/settings.json` 打开 `agentPushNotifEnabled` 和 `inputNeededNotifEnabled`，改完新开会话。
6. 在电脑前看会话时不会推送；测试推送要锁屏或切走焦点。
7. agent-harness 主流程不受客户端切换影响；只有可选的 debate 脚本依赖 VS Code 扩展里的 CLI。
