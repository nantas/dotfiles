# Magpie 与多 Agent 接入

用 chezmoi 完成 **Magpie 安装 + 无密骨架初始化 + Agent CLI 预装**，再由用户补齐 API Key、
本仓库 Agent 接力完成浏览器 OAuth 鉴权与最终模型接线。支持 macOS、Linux（桌面与 `headless`）
以及 Windows。

---

## 三阶段恢复流程

| 阶段 | 执行者 | 做的事 | 产物 / 边界 |
| --- | --- | --- | --- |
| 1. 自动安装与无密初始化 | `chezmoi apply`（`52-ai-agents` + `55-magpie`） | 安装 `claude` / `codex` / `agy`（Windows 含 `pi`）、安装 Magpie、设通用偏好、装 3 个社区插件、首次 seed `providers.json`（Key 为空）与 Agent 基础配置、开启自启/后台服务 | 全程无 root、无 TTY 阻塞；不含任何 Key 或 OAuth 凭据 |
| 2. 填入密钥 | 用户手工 | 在 Magpie UI → Providers 或 `magpie provider key <id> <key>` 填入启用项的 Key | `deepseek`、`zhipuai-coding-plan`、`teamorouter-claude`、`kimi`、SRV 网关 Key |
| 3. 浏览器鉴权与接线 | 本仓库 Agent（`/magpie-auth`） | 核对 Key 与 SRV，逐项拉起 `workbuddy` / `grok` / `agy-cli` / `codex` 浏览器登录，再跑 `magpie claude` / `magpie pi` / `magpie codex` 完成模型接线与测试 | 提示词见 [`.pi/prompts/magpie-auth.md`](../.pi/prompts/magpie-auth.md) |

---

## 跨机器同步（GitHub 加密）

「为什么不直接托管」一节的种子只管**首次**无密初始化：`chezmoi apply` 在新机器上把骨架铺好，之后不再碰这些文件。
各机之间的持续一致走另一条路：Magpie 自带的 GitHub 加密同步。两者不冲突——同步涉及的文件
本就不在 chezmoi 托管内，下次 `chezmoi apply` 不会覆盖同步带来的内容。

### 机制

`magpie github on github://<owner>/<repo>` 把 `providers.json`、Agent 配置、`library/`、
`memory/` 等内容用 passphrase 密封后，存进一个**私有** GitHub 仓库；GitHub 只收到密封文件，
拿不到明文。token 需要该仓库的 Contents 读写权限，passphrase 各机相同。两者都只存在各机
`~/.config/magpie/sync.json`（权限 600，不入库），经 `magpie github set token=` 或首次 `on`
时的 stdin 提问写入，不进命令行参数、不进仓库。

### 当前部署

一台机器双向同步作为唯一上推方，其余机器 `magpie github follow on`（对应配置里的
`followServer: true`）只接收、不上传。当前是 macOS 为上推方，Linux 桌面机（10.10.10.x 网段）
与 Windows 机为跟随方。自动同步间隔 15 分钟：`magpie github auto 15`（对应 `autoEvery`）。

`follow on` 的语义（依据 `internal/davsync` 的 `Config.FollowServer` 注释）：每次同步把服务器上
变化的部分带进来，本机改动会被覆盖，并在 `~/.config/magpie/sync/` 留一份密封副本，`magpie
github` 会给出 "Replaced here" 提示。该提示用 `magpie github dismiss` 清除（只清提示，不动
内容）；`magpie github undo-sync` 才会真的把本机回滚到同步前。跟随方的同步**不会**让仓库
产生新提交。

### 验证同步一致性

不写死某次哈希，用这几条可复用判据：

- 三台机器 `~/.config/magpie/sync-state.json` 的 `server.etag` 与 `sum` 相同，且
  `server.etag` 等于仓库中同步文件 blob 的 sha；
- 跟随方同步前后，用 `git` / `gh` 查到的仓库 HEAD 不变；
- `magpie github preview` 输出 `nothing would change`。

### 新机器接入

先备份本机已有的 `~/.config/magpie`（Windows 为 `%USERPROFILE%\.config\magpie`），再：

```sh
magpie github on github://<owner>/<repo> token= passphrase= keys=yes agents=yes library=yes memory=yes
magpie github follow on
magpie github auto 15
```

`token=` / `passphrase=` 必须留空：CLI 对空值才提问，非 TTY 时各读一行，按「第一行 token、
第二行 passphrase」喂入。Windows 上可执行文件在 `%LOCALAPPDATA%\Programs\magpie\magpie.exe`。

### 自动同步的前提与限制

同步间隔由常驻的 gateway 进程驱动（每分钟看一次配置），不是 systemd timer。Linux 桌面靠
`magpie autostart` 写的 XDG `~/.config/autostart/magpie.desktop`（图形登录才触发），Windows
靠 `HKCU\...\Run` 里的 `magpie ... tray`。纯 SSH / 无登录会话时没有 gateway，`magpie github`
仍会显示「每 N 分钟」，但不会真的同步。Linux 上没有 `magpie` 的 systemd 用户单元，这是正常的。

### 不要做

- 不要把 `sync.json` 放进 chezmoi 托管：它含凭据，且被 magpie 运行时改写，二次 `apply` 会打架。
- 不要在跟随方上改配置指望它传回上推方：跟随方不上传，本地改动只会被下次同步覆盖（密封副本
  会留在 `sync/`，见上）。

---

## 为什么不直接托管 `~/.config/magpie/*.json`

Magpie 与各 Agent 在运行时都会持续写回配置文件：

- `~/.config/magpie/` 下的 `providers.json`、`settings.json`、`plugins.json`、`logins.json`、
  `plugin-auth.json`、`stash.json` 既是配置又是运行时与凭据状态；
- `~/.pi/agent/settings.json`（已于 commit `8c5ee5c` 退出 chezmoi 托管）、`~/.claude/settings.json`、
  `~/.codex/config.toml` 会被 `magpie` 和各自 CLI 改写。

如果用普通受管文件覆盖，`chezmoi diff` 永远不会干净，二次 `apply` 还会抹掉本机填入的 Key、
登录态和模型选择。因此仓库采用 **`scripts/magpie/` 种子 + 状态标记（`~/.local/state/magpie-bootstrap/seeded`）**：

- `settings.json` 通过 `magpie settings set` 只在首次 seed 时写入通用偏好；
- `providers.json`（Unix 下强制 `0600`）首次写入，或仅在首次合并**缺失的 Provider ID**，绝不覆盖已有条目与 Key；
- 路由组不写进 `providers.json`，而是由 `scripts/magpie/groups.json` 声明、`magpie group add/set`
  播种（见 [`magpie-routing.md`](magpie-routing.md)）；因为组只含模型 id 与策略，没有凭据；
- `~/.pi/agent/*`、`~/.claude/settings.json`、`~/.codex/*.toml` 仅在目标文件**不存在**时写入初始偏好。

---

## SRV 远端网关端点的传入方式

带实例 ID 的私人 SRV 地址与 Gateway Key 不进入公开仓库。`run_onchange_after_55-magpie` 按以下优先级获取 SRV 地址：

1. 本机 `~/.config/chezmoi/chezmoi.toml` 中的 `[data.magpie]`（不入库）：
   ```toml
   [data.magpie]
   srvUrl = "https://<srv-host>"
   ```
2. 执行 `chezmoi apply` 时的环境变量 `MAGPIE_SRV_URL="https://<srv-host>"`。
3. 当前还没有 `remote-*` Provider 且**处于交互终端**时，脚本在执行期提示输入一次（Enter 可跳过）。
4. 若前 3 步均跳过，阶段 3 执行 `/magpie-auth` 时由 Agent 向用户询问地址与 Key 并通过 `magpie provider add remote-magpie` 补录。

脚本会自动以主机名首段生成 `remote-<label>` 作为 Provider ID，并仅筛选 `claude/claude-opus-5-5`、
`claude/claude-sonnet-5-5`、`claude/claude-haiku-5-5`。

---

## 锁定的插件与鉴权分工

- **社区插件（3 个）**：
  - `@magpie-community/opencode-workbuddy-auth@latest`
  - `@magpie-community/opencode-grok-auth@latest`
  - `@jianmu-plugins/opencode-agy-cli-auth@latest`
  （`pi-xai-oauth` 已弃用并移除。）
- **API Key Provider（阶段 2 填 Key）**：
  - 启用：`deepseek`（仅 `deepseek-flash`）、`zhipuai-coding-plan`（`glm-5.3` / `glm-5.3-flash`）、
    `teamorouter-claude`（`claude-fable-5-1` / `claude-fable-5-2`）、`kimi`（`kimi-for-coding` / `k3.1-pro`）、`remote-<label>`（SRV）。
  - 保持停用（`off: true`）：`openlux`、`minimax-cn`、`zai-coding-cn`、`teamorouter`。
- **浏览器 OAuth（阶段 3 `/magpie-auth`）**：
  - `workbuddy`、`grok`、`agy-cli`、`codex`（ChatGPT 官方订阅直连）。

---

## 跨平台行为差异

| 平台 | Magpie 安装方式 | 后台运行 | 配置目录 |
| --- | --- | --- | --- |
| macOS | `usemagpie.ai/install.sh`（`/Applications/magpie.app` + `~/.local/bin/magpie`） | `magpie autostart on` | `~/.config/magpie` |
| Linux 桌面 | `usemagpie.ai/install.sh`（若无 TTY 且缺 `webkit2gtk-4.1` 会先装 `--tui`；补装 `webkit2gtk-4.1` 后重跑官方脚本即得桌面版） | `magpie autostart on` | `~/.config/magpie` |
| Linux `headless` | `usemagpie.ai/install.sh -s -- --tui` | `magpie daemon install` | `~/.config/magpie` |
| Windows | 下载 `usemagpie.ai/download/windows-{amd64,arm64}` 到 `%LOCALAPPDATA%\Programs\magpie\magpie.exe` 并加入用户 `Path` | `magpie autostart on` | `%USERPROFILE%\.config\magpie` |

Windows 的 PowerShell 写入统一采用无 BOM UTF-8，避免 JSON/TOML 解析失败；`pi` 的扩展继续由
`~/bin/pi-config`（Windows 下 `C:\bin\pi-config`）独立仓库管理，本方案不接管扩展目录。

---

## 路由组

同一个 agent 需要多个供应商的模型互相兜底时，用 `group/<id>` 把同档次模型并成一个，而不是
在 agent 里逐个换。分组方式、`routing` 语义与参数陷阱见
[`magpie-routing.md`](magpie-routing.md)。
