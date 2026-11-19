---
description: 完成 Magpie API Key 核对、浏览器 OAuth 鉴权与 Agent 模型接线
argument-hint: "[focus]"
---

接力完成新机器上的 Magpie 鉴权与 Agent 模型绑定。假设 `chezmoi apply` 已跑完
`run_onchange_after_52-ai-agents` 与 `run_onchange_after_55-magpie`，本仓库参考文档见
[`docs/magpie.md`](../../docs/magpie.md)。

## 本次焦点

${@:-（未指定，执行完整 Stage 3 恢复流程）}

## 1. 环境与网关状态体检

在终端执行以下只读检查并把摘要告诉我：

```bash
command -v magpie pi claude codex agy
magpie autostart 2>/dev/null || magpie daemon status 2>/dev/null || true
magpie providers
magpie plugin
magpie accounts
```

确认：
- `magpie` 网关已在运行（默认 `127.0.0.1:3425`；未启动时桌面机可启动应用/托盘，无桌面机跑 `magpie daemon start` 或 `magpie serve`）。
- 3 个锁定插件已安装且为 `on`：
  - `@magpie-community/opencode-workbuddy-auth@latest`
  - `@magpie-community/opencode-grok-auth@latest`
  - `@jianmu-plugins/opencode-agy-cli-auth@latest`
  （若曾装过 `pi-xai-oauth`，先执行 `magpie plugin rm pi-xai-oauth` 卸载。）

## 2. 核对 API Key 与 SRV 远端网关

逐一确认以下启用的 Provider 均已填入可用 Key（停用项 `openlux`、`minimax-cn`、`zai-coding-cn`、`teamorouter` 保持 `off`）：
- `deepseek`
- `zhipuai-coding-plan`
- `teamorouter-claude`
- `kimi`
- `remote-*`（Remote Magpie / SRV）

处理缺口：
- 若 `providers` 中还没有 `remote-*`：向我询问 SRV 网关地址（`https://...`）与远端签发的 Gateway Key，计算主机首段标签作为 `id=remote-<label>`，执行：
  ```bash
  magpie provider add remote-magpie <gateway-key> \
    url=<srv-url> id=remote-<label> \
    models=claude/claude-opus-5-5,claude/claude-sonnet-5-5,claude/claude-haiku-5-5
  ```
- 若任何启用的 API Key Provider 仍缺少 Key：停下并列出缺失项，等我在 Magpie UI 或命令行（`magpie provider key <id> <key>`）补齐后再继续。

## 3. 逐项完成浏览器 OAuth 鉴权

以下登录需要用户在浏览器中点击授权；**每次只启动一项**，输出授权链接后等待我确认完成再进行下一项：

1. WorkBuddy：`magpie plugin login workbuddy`
2. Grok：`magpie plugin login grok`
3. Antigravity (`agy-cli`)：`magpie plugin login agy-cli`（若提示需要官方 CLI 登录，先核对 `agy` 状态）
4. Codex（ChatGPT 官方订阅）：`magpie accounts add codex`（若尚未登录；已登录则跳过）

若我希望在 Pi 中保留原生入口（如 `antigravity` / `openai-codex`），另行询问并协助登录；`kimi` 固定走 Magpie API Key，不走 OAuth。

## 4. 核对可用模型目录

全部鉴权完成后检查实际目录：

```bash
magpie models
magpie models pi
magpie models claude
```

确认实际可用模型（尤其核对 `agy-cli` 下的 `gemini-4-argon` / `gemini-4-hydrogen`、`grok/grok-4.7`、`deepseek/deepseek-flash` 与 SRV 的 Claude 5.5 家族）。

## 5. 完成最终 Agent Wiring

记下 `magpie providers` 中实际的 SRV ID（记作 `<srv>`），依次执行：

1. **Claude Code**（由 Magpie 写入鉴权与选择器）：
   ```bash
   magpie claude <srv>/claude/claude-opus-5-5[1m]
   magpie claude effort xhigh
   magpie claude opus <srv>/claude/claude-opus-5-5[1m]
   magpie claude sonnet <srv>/claude/claude-sonnet-5-5[1m]
   magpie claude haiku <srv>/claude/claude-haiku-5-5[1m]
   magpie claude subagent <srv>/claude/claude-sonnet-5-5[1m]
   ```
2. **Pi**：
   ```bash
   magpie pi magpie/deepseek/deepseek-flash
   magpie pi effort high
   ```
   然后检查 `~/.pi/agent/settings.json` 中的 `enabledModels`（Ctrl+P 轮换列表），补充 `magpie/<srv>/claude/claude-opus-5-5`、`magpie/<srv>/claude/claude-sonnet-5-5`、`magpie/<srv>/claude/claude-haiku-5-5`（剔除目标机不可用的条目），并确认扩展目录已按当前扩展仓接入。
3. **Codex**：
   ```bash
   magpie codex default
   ```
   保持官方 ChatGPT 直连与 `gpt-6.2-astra` / `medium`（或当前默认可用模型）。

## 6. 连通性测试与收尾汇报

对启用的 API Provider 运行：

```bash
magpie provider test deepseek
magpie provider test zhipuai-coding-plan
magpie provider test teamorouter-claude
magpie provider test kimi
magpie provider test <srv>
```

最后输出 `magpie claude`、`magpie pi`、`magpie codex` 与 `magpie plugin` 摘要，确认：
- 全部启用 Provider 测试通过；
- `openlux`、`minimax-cn`、`zai-coding-cn`、`teamorouter` 仍保持停用；
- 浏览器 OAuth 账号均已显示 signed in；
- 按需重建机器特有的 Claude `SessionStart` hooks、Codex `mcp_servers` / `projects` 与 Magpie Library。
