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
