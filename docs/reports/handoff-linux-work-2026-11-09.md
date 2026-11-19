# linux-work 重启故障交接

日期：2026-11-09。用户将在本机 dotfiles 会话继续处理。用户明确指定把本文放在 ~/projects/dotfiles，覆盖 handoff 技能默认临时目录要求。本文未包含认证材料或完整私人配置。

## 后续更新：数据卷已迁移 ext4（同日）

本机后续会话已将原数据卷备份到机械盘，归档约 281.48 GiB，完整读回校验及源文件比较通过。
用户授权重新格式化原数据分区为 ext4，恢复与完整比较成功；fstab 已改为新 UUID 与 ext4，
挂载单元正常，普通用户写入、同步、读回实测通过。根分区、EFI 和机械盘原有数据未改动，
备份保留。重启后的自动挂载验收待完成。

首次脚本在格式化后的挂载身份检查处停止，后续用不含格式化操作的续跑脚本完成恢复。
旧 WSL 的三种扩展属性未迁入 ext4，原值保留在归档中。过程与验证边界见
[恢复与迁移记录](../ntfs-recovery.md)。下文为迁移前交接，不代表当前数据卷仍为 NTFS。
本次迁移未重新验证 Codex 在线账号路由问题，不能将其标记为已解决。

## 交接时状态（迁移前）

- **磁盘挂载已恢复**：用户执行修复后报告成功，随后通过 SSH 独立确认 /dev/nvme0n1p5 以 ntfs3、rw 挂载到 ~/data，~/data/projects 存在。
- **Codex 在线账号路由问题尚未解决**：不要将挂载恢复视作 Codex 已恢复。
- 未改动 dotfiles 配置、fstab、Herdr 配置或仓库源码；未重启 work 服务，也未提交或推送。本文是本次新增交接文件。

## 初始症状与环境

设备关机后重新启动，Herdr work 会话的 linux-work 上出现两项故障：

1. codex 启动报错：`account/read failed during TUI bootstrap: account/read failed: workspace routing discovery failed (code -32603)`。
2. ~/data/projects 无法访问。

系统：Arch Linux，内核 7.2.11-arch1-1，Codex CLI 0.166.0。系统时间已同步。使用已保存的 linux-work SSH 目标；不要沿用文档里的历史网络地址。远端 sudo 需要用户输入密码，当前代理不能无人值守执行 sudo。

## 磁盘结构与诊断

- 一块约 1 TB Kingston NV2 NVMe（nvme0n1）：p3 344 GiB ext4 是 Linux 根目录；p5 585.9 GiB NTFS 是 ~/data；p1 是 EFI；p4/p6 是不足 1 GiB 的遗留 NTFS 分区。
- 用户确认已无 Windows 系统。p3/p5 很可能对应原 C:/D:，但历史盘符未经证实。
- 另有约 7.3 TiB 的 sda2 NTFS 分区，未挂载，与 ~/data 无关。不要误操作该盘。
- EFI 仍列出 Windows Boot Manager；这只是遗留启动项，不能据此断定 Windows 仍可用。
- fstab 使用 p5 的 UUID、ntfs3、rw、uid/gid=1000、dmask/fmask=0022、nofail、x-systemd.device-timeout=10s。此次没有修改。
- 开机 home-nantas-data.mount 失败，内核明确记录 `volume is dirty and "force" flag is not set!`，建议 chkdsk。
- ntfsfix 检查发现 `$MFTMirr does not match $MFT (record 3)`。ntfs-3g 只读挂载同样失败。
- SMART：PASSED；Critical Warning=0；Media and Data Integrity Errors=0；Error Information Log Entries=0；Percentage Used=5%；Unsafe Shutdowns=7（累计值，不能归因到这次关机）。内核检查未发现底层 NVMe I/O 错误。无硬件损坏直接证据，但未证明文件完整性。

## 已执行的修复过程

用户在远端交互执行了管理员命令：

1. 安装 ntfs-3g；发现当前 Arch 已拆包，随后安装 ntfsprogs（两者版本 2026.9.28-1）。ntfsfix 由 ntfsprogs 提供。
2. 最初建议 `sudo ntfsfix --no-action /dev/nvme0n1p5`，其模拟输出显示镜像不一致；用户未明确确认首次命令是否保留 --no-action，不应把模拟输出视为真实修复。
3. 明确实际执行 `sudo ntfsfix /dev/nvme0n1p5`：修正 MFTMirr record 3，处理日志、检查备用引导扇区，最终报告 processed successfully。
4. 后续曾使用 /mnt/data-check 临时挂载并卸载；用户没有贴出目录列表，不宣称项目文件已逐项验证。
5. `sudo systemctl restart home-nantas-data.mount` 仍失败，内核仍报 dirty。ntfsfix 默认保留/设置需检查标记，并非前一步修复无效。
6. 指示用户执行 `sudo ntfsfix -d /dev/nvme0n1p5`，成功后重启挂载服务；用户报告挂载成功。未提供该命令完整输出，但最终挂载状态已独立确认。

没有使用 force 挂载，没有格式化或更改分区。ntfsfix 不是完整 chkdsk；尚未备份、做完整文件系统扫描或校验项目数据。

## Codex 调查证据（未解决）

- codex 路径是 ~/.local/bin/codex，指向 ~/.codex/packages/standalone/current/bin/codex。
- `codex login status` 显示 Logged in using ChatGPT；这不证明在线账号刷新成功。
- 从家目录启动独立 `codex app-server --stdio`，发送 initialize、initialized、account/read：
  - params={"refreshToken":false} 返回成功（缓存路径）。
  - params={"refreshToken":true} 复现 code=-32603，message=`workspace routing discovery timed out`。
- 临时 PTY 启动 TUI，在约 18 秒观察期内未出现原始错误，但进程仍运行后被清理；不能据此判定完整启动成功。
- 普通 HTTPS 探测可连接：chatgpt.com/backend-api/ 返回 HTTP 404，auth.openai.com 返回 HTTP 403。这只证明 TLS/HTTP 可达，不证明已认证路由接口可用。
- SSH 非交互环境中未发现 proxy/Codex/OpenAI 相关环境变量。不能据此推断 Herdr 服务器、交互 shell 或 Codex 进程的环境相同。
- ~/.codex/config.toml 的窄范围检查未输出 proxy/base_url/chatgpt/routing 配置键；未导出整份私人配置。
- 日志在 ~/.codex/logs_2.sqlite，非 ~/.codex/log/codex-tui.log。logs 表含 ts、level、target、feedback_log_body；排查时只读查询并隐藏凭据、账号信息。发现模型列表在线刷新超时，但未确认这些记录与本次启动一一对应。
- 诊断用临时 app-server/TUI 进程已终止，没有向已有 Agent 注入输入或提交模型任务。

## Herdr 影响

挂载恢复前，work 的两个 GitNexus 窗格返回 `Saved directory is unavailable. Restore the directory and restart this session.`；另一个 dotfiles 窗格正常。挂载恢复后尚未重新检查失败窗格的恢复情况。不要直接重启整个 work：先查询当前 pane、进程和任务，避免结束用户工作。旧 pane/terminal ID 不应当作持久有效标识。

## 下一步建议

1. 验证 ~/data/projects 下重要仓库、未提交修改及文件可读性，安排备份。调查正常关机流程和挂载卸载日志，避免把累计 unsafe shutdowns 当成本次根因。
2. 在发生故障的真实交互环境重新启动 Codex，并复测在线 account/read。区分认证路由接口超时、网络/代理上下文差异和服务端问题；不要无证据退出登录或删除凭据。
3. 若需持久调整 NTFS 使用策略，先看现有 dotfiles 管理模式；当前只有 Linux，是否迁移 ext4 应在完整备份后另行决定，不属于本次已授权修复。
4. 恢复 Herdr 项目入口前核对实际工作目录与运行任务；不要假定重启会自动恢复 Codex 对话。

## suggested skills

- `diagnosing-bugs`：以真实在线 account/read 或 TUI 错误建立反馈循环，一次改变一个变量。
- `openai-docs`：需要查询 Codex 认证、配置和故障排查的官方说明时使用，优先本机现有证据。
- `ponytail`：最小完整修复，避免为单次故障新增无关配置。
- 若涉及 Herdr 个人环境，参考控制端 Herdr 仓库 workflows/nantas/README.md 与 workflows/nantas/bootstrap.md。不要假设本机 dotfiles 中存在这些文件。
