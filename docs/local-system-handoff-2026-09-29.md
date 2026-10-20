# 本机环境与配置交接记录（2026-09-29）

本文记录本次会话的实施结果及剩余问题，供后续配置、排障使用。状态以本次会话结束前的检查为准；早期故障结论按当时日志记录，不代表已经完成长期稳定性验证。本文不包含订阅地址、访问令牌、私钥、控制 secret、节点名或 `allow-lan` 一类会变的现值。这些用 [`mihomo.md`](mihomo.md) 在本机重读，不要把查到的值补回本文。当时的会话没有推送。


> **最新状态（2026-10-20）**：本文保留历史快照，后面的进展覆盖前文待办。
> Unity 6000.0.73f1 与 Android 全套组件已安装，ARM64/IL2CPP APK 构建通过（第 18 节）；
> SSD 数据分区已临时挂载到 `~/data`（第 19 节）；内网 Git 主仓库和 7 个子模块读取及推送通道
> dry-run 通过，浏览器域名直连已由用户确认（第 20–21 节）。profile 弹窗目前未复现、根因未定
> （第 22 节）；中文候选乱码仍暂停排查。第 23 节汇总可复用流程和剩余待办。

## 1. 最重要的现状

- 系统为 Arch Linux，内核 `7.2.7-arch1-1`，默认使用 niri / Wayland，GNOME 保留备用，登录管理器为 GDM。
- 有线网络已经可用，手机热点已关闭；当前活动连接为 `Wired connection 1`，不启用 802.1X。
- 中文显示、Fcitx5/Rime 输入和候选字显示已恢复；候选框字体修改在重新登录后生效。
- 为规避睡眠恢复卡死，系统目前禁用挂起、休眠及混合睡眠；niri 使用空闲锁屏、关闭显示器的方式节能。
- Mihomo 系统服务正在运行并已启用开机启动；终端和浏览器已经分别配置代理，未开启 TUN 全局接管。
- GitHub CLI 登录、公钥登记、SSH 认证和仓库克隆均已验证成功。
- 绿联蓝牙适配器在 Linux 下的 BLE 配对问题未解决；用户决定暂停，之后尝试 Logi Bolt 接收器。

## 2. 三个目录的关系

| 目录 | 当前用途 |
| --- | --- |
| `~/bin/dotfiles` | **实际使用的 chezmoi 源目录**，包含本次系统适配的未提交修改 |
| `~/projects/dotfiles` | 新通过 SSH 克隆的 `jwu/dotfiles` 仓库，本文存放于此；尚未接管系统配置 |
| `~/projects/network-diagnosis` | 网络、蓝牙、Mihomo 安装与诊断材料；检查时不是 Git 仓库 |

**不要直接从新克隆目录运行全量 apply。** 两个 dotfiles 目录尚未合并，本次本机适配主要保存在 `~/bin/dotfiles`。后续若要迁移源目录，先比较提交和未提交改动，保留本机适配，再修改 chezmoi 的 sourceDir，审阅 diff 后应用。

原源目录的修改涉及浏览器代理、shell、niri、锁屏、输入法环境、Git 模板、默认应用、Neovim 和文档。新增了用户级 niri 空闲服务。完整清单以 `git -C ~/bin/dotfiles status --short` 为准，不应把所有差异当成可丢弃文件。

## 3. 有线网络

- 当时有线链路是 UP，连接名 `Wired connection 1`，未启用 802.1X。
- 接口名、MAC、静态地址、网关和 DNS 不写进仓库。在本机用 `nmcli -f GENERAL.CONNECTION,IP4.ADDRESS,IP4.GATEWAY,IP4.DNS device show` 和 `ip -br link` 看。
- `Wired connection 1` 已移除 802.1X，保留静态地址；原认证设置另有停用的备份连接。
- Windows 曾启用 IEEE 802.1X，但用户不需要手动登录。Linux 实测在当前网络不启用认证也能联网，这不能推断公司网络永远不需要认证。
- 公司认证账号与 Linux 本地账号不是同一个身份；若未来恢复认证，先核实网络侧要求，不要把本地用户名直接当成公司认证账号。

### GitHub 的临时 hosts 修复

早期直连 GitHub 的 DNS 结果无法稳定建立连接，曾在 `/etc/hosts` 把 `github.com` 固定到一个地址。具体地址不记在这里；用 `rg github.com /etc/hosts` 看现在还有没有。当时留有一份 hosts 备份，文件名带日期。代理已经可用，后续应评估是否删除这条临时固定解析，再验证访问；不要直接恢复整份旧 hosts，以免覆盖其他后续修改。

## 4. 中文输入和桌面

- 当前主要输入法：Fcitx5 + Rime；安装检查为 `fcitx5 5.1.23-1`、`fcitx5-rime 5.1.16-1`。
- 早期终端中文显示为问号、候选字显示方块的问题已处理。用户重新登录后确认候选字可见。
- 输入法环境配置位于 `~/.config/environment.d/fcitx5.conf`，对应源位于实际 chezmoi 源目录。
- niri 中 `Super + Space` 打开应用启动器；输入 Chromium 后回车可启动浏览器。
- 当前 Codex 对话输入框使用 `Ctrl+V` 可粘贴文字；`Ctrl+Shift+V` 曾触发图片粘贴失败。不要因此直接判断浏览器与终端使用不同剪贴板。

## 5. 睡眠、锁屏与稳定性

### 故障与判断

用户曾在隔夜后遇到停在时间/锁屏界面，键鼠触发 suspend 相关错误，无法登录，只能断电重启。此前日志出现 nouveau 驱动相关 channel killed 和 xHCI 错误，显卡为 RTX 4060；睡眠恢复路径存在问题。根因未完全解决，也未宣称更换或修复了显卡驱动。

### 已实施的规避配置

`/etc/systemd/sleep.conf.d/90-desktop-stability.conf`：

```ini
[Sleep]
AllowSuspend=no
AllowHibernation=no
AllowHybridSleep=no
AllowSuspendThenHibernate=no
```

- GNOME 接交流电时的自动睡眠设为关闭，电源按钮行为为交互询问。
- `~/.config/systemd/user/niri-idle.service` 在 niri 启动时运行；导出时为 active。
- 空闲 600 秒运行 `~/.local/bin/niri-lock`。
- 锁屏脚本采用 flock 防止重复启动；锁屏后短时间空闲关闭显示器，活动时恢复显示。
- 移除了相关锁屏布局中的 suspend 操作入口。
- 备份目录包括 `~/.local/state/power-backup/20260929-124357` 和 `20260929-120520`。

当前是“禁用系统睡眠、保留锁屏和显示器节能”，不是恢复了可靠的休眠。没有为验证而主动触发睡眠、重启或锁定用户桌面。是否彻底避免隔夜问题仍需要实际观察。

## 6. 蓝牙及罗技设备（暂停）

- 适配器：绿联 Ugreen 蓝牙 5.4，USB ID `33fa:0010`，版本 `88.91`，调查指向 Barrot 2279 系列。
- 曾从主板原 USB 口换到 USB 2.0 口；Windows 下同一适配器工作稳定。
- 设备：MX Anywhere 3S 鼠标、Wave Keys 键盘，尝试蓝牙连接。
- 原 Linux 驱动出现约每四秒断开重连、HCI 扩展扫描命令超时等问题。
- 临时 btusb 修改曾让扫描恢复，但鼠标和键盘在远端特征读取阶段仍出现 Controller Busy / 认证失败，未成功完成配对。
- 测试包含禁用部分扩展能力、切换连接方式、在相关工作线程加入 500ms 等待，均未解决配对。
- **已恢复原版 btusb，未安装持久 DKMS 修改。** 当时校验的原模块 srcversion 为 `AAC12D819351F438F5E59D9`。
- 蓝牙服务目前 active 不等于适配器或设备已可正常使用；最后一次关闭控制器操作也未确认成功。
- 研究材料在 `~/projects/network-diagnosis/bluetooth-research`；为测试安装过 linux-headers、pahole。

下一步由用户尝试 Bolt 接收器，再判断是否需要 Solaar 管理按键、设备配对或功能。当前不能记录为已经安装并验证了罗技专用驱动或 Bolt。

## 7. Mihomo 与代理

### 服务及数据

- 已安装 `mihomo-bin 1.19.31-1`、`python-yaml 6.0.3-2`。
- 包由 AUR PKGBUILD 构建，下载官方 MetaCubeX 二进制并通过 makepkg 校验；构建资料在诊断目录的 `mihomo-setup/`。
- `mihomo.service` 为系统服务，导出时 active，已 enable。
- 配置：`/etc/mihomo/config.yaml`；服务覆盖：`/etc/systemd/system/mihomo.service.d/override.conf`。
- 客户端约定的混合端口是 `127.0.0.1:7890`；控制 API 由服务参数提供，常见是 `127.0.0.1:9090`。监听地址、`allow-lan`、TUN、模式和当前节点都是运行时状态，不在本文固化。重读步骤见 [`mihomo.md`](mihomo.md)。
- 之前订阅下载发生过 HTTP 403，后来重试同类下载成功，不应据此认定订阅失效或单独归咎于剪贴板。不要把订阅 URL 或响应正文写进文档。
- 第一次系统配置校验失败；用户目录补齐地理数据库后校验通过，重试系统激活成功。当时用的库文件名是 `geoip.metadb`。这台机器现在是这个文件还是 `clash-geoip` 的 `Country.mmdb`，用 [`mihomo.md`](mihomo.md) 重读，不要沿用本节。
- 若用过本机导入，状态在 `~/.local/state/mihomo-setup/`（权限应为 0700）。里面有订阅和 secret，不提交、不摘录。导入脚本在诊断目录的 `mihomo-setup/`；系统配置备份若存在，在 `/var/lib/mihomo-backups/`。

### 终端

实际 chezmoi 源的 `dot_zshrc.tmpl` 已修改并应用到 `~/.zshrc`。Linux 交互式 zsh 检测到 mihomo 命令时，静默自动执行 `pon`：

- HTTP/HTTPS 代理为 `http://127.0.0.1:7890`，SOCKS 为 `socks5h://127.0.0.1:7890`。
- 同时设置大小写代理环境变量及 NO_PROXY/no_proxy；localhost 和私有网段列入绕过列表。
- `pstat` 查看状态，`poff` 清除当前终端的代理变量，`pon` 重新启用。
- 新交互式 zsh 已实测自动经本地代理访问 GitHub。旧终端需要重新打开或执行 `source ~/.zshrc`。
- 这不代表所有程序都自动走代理：程序必须支持这些环境变量，非交互 shell、sudo、系统服务、独立启动的 GUI 不一定继承；SSH 已另行配置。
- 不同客户端对 NO_PROXY 网段的支持不同。订阅里有没有私网直连，用 [`mihomo.md`](mihomo.md) 重读，不要沿用本节的印象；仅公司内网可解析的域名仍需按实际情况补充。
- 自动开启不检查服务健康；若 Mihomo 停止，应恢复服务，或临时 `poff` 直连。

### Chromium

`~/.config/chromium-flags.conf` 与 `chrome-flags.conf` 已配置本地 HTTP 代理及私网/本地名称绕过规则，对应源同步修改。

**重要：`chrome://restart` 会沿用旧启动参数，本次未使新 flags 生效。** 必须保存未提交内容，通过菜单彻底退出所有浏览器窗口，再从应用启动器或 `chromium` 命令重新打开。启动器已确认可读取代理参数。用户随后确认 GitHub 访问顺畅。

代理下 curl 测试：YouTube 首页 HTTP 200 约 0.39 秒、缩略图约 0.18 秒、GitHub 约 0.43 秒。这是单次页面/资源测试，不代表视频持续吞吐测试；用户尚未明确反馈彻底重开后 YouTube 视频播放情况。

### 尚未完成的代理事项

- 未配置定时更新订阅；节点需要更新时应重新导入、保持本地代理和私网规则、校验后再替换。
- 控制面板文件是否完整下载、浏览器登录面板等尚未端到端验证。
- 策略组的当前选择是运行时状态，不记节点名，也不把某次切换的结果写回本文。读取见 [`mihomo.md`](mihomo.md)。
- 需要时再测试 YouTube 视频；避免把首页 HTTP 200 当作全部问题解决。
- 临时 GitHub hosts 条目尚未清理。

## 8. GitHub、SSH 和仓库

- `gh auth status` 已确认登录并启用账号，Git 协议选择 SSH。
- OAuth 范围：`repo`、`read:org`、`admin:public_key`、`gist`。授权范围不等于对任意仓库拥有写权限。
- 复用本机 `~/.ssh/id_ed25519` / `.pub`；公钥已通过 GitHub API 核对登记成功，未读取或输出私钥。
- `~/.ssh/config` 仅为 GitHub 配置 `HostName ssh.github.com`、端口 443、用户 git、指定 ed25519 密钥及 `IdentitiesOnly yes`。
- `ProxyCommand` 使用 OpenBSD nc，经 `127.0.0.1:7890` HTTP CONNECT 建立 SSH 通道。
- GitHub 官方 ED25519 主机指纹已核对，并为 `[ssh.github.com]:443` 写入 known_hosts；没有关闭主机校验。
- `ssh -T git@github.com` 返回成功认证欢迎语；GitHub 不提供交互 shell，此命令退出码 1 是该成功提示的常见行为。
- 已用 `git ls-remote git@github.com:jwu/dotfiles.git HEAD` 验证 SSH 读取，并成功克隆到 `~/projects/dotfiles` 的 main 分支。
- 未进行试验性 push，未修改已有实际配置源的 remote，也没有验证对上游的写权限。
- 检查时调用环境未连接 ssh-agent；当前批处理 SSH 认证已可用，没有另行部署常驻 agent 服务。未来若使用加密密钥再配置解锁流程。

官方参考：[SSH 443 端口](https://docs.github.com/en/authentication/troubleshooting-ssh/using-ssh-over-the-https-port)、[主机指纹](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/githubs-ssh-key-fingerprints)。

## 9. Codex 会话权限

当前会话实际权限为 `danger-full-access`、approval policy 为 `never`，网络访问开启。操作系统 root 权限仍独立存在，pkexec 弹窗要求的本机密码不是 Codex 工具审批。不要因为关闭 Codex 审批就认为系统管理操作不需要授权。

## 10. 后续优先级

1. 合并并明确两个 dotfiles 目录的用途，保留实际配置源所有未提交适配，避免从新 clone 覆盖本机。
2. 确认 Chromium 彻底退出再打开后的 YouTube 播放表现；如仍慢，再采集实际代理连接及视频 CDN 情况、比较节点。
3. 完善订阅更新和面板操作流程，评估清理临时 GitHub hosts。
4. 实际观察隔夜锁屏稳定性；恢复睡眠前先解决显卡/USB 恢复问题，当前睡眠禁用是规避措施。
5. 用户拿到 Bolt 后测试鼠标键盘；蓝牙自定义驱动研究保持暂停。
6. 视后续用途核实具体仓库写权限；当前登录、SSH 读取和克隆已完成。

## 11. 快速只读检查

```sh
chezmoi source-path
git -C ~/bin/dotfiles status --short
git -C ~/projects/dotfiles status --short
nmcli connection show --active
systemctl is-active mihomo
systemctl is-enabled mihomo
systemctl --user is-active niri-idle.service
pstat
chromium --help
gh auth status
ssh -o BatchMode=yes -o StrictHostKeyChecking=yes -T git@github.com
git ls-remote git@github.com:jwu/dotfiles.git HEAD
```

排障时不要把完整订阅配置、gh 凭据文件、控制 API secret 或私钥输出到共享记录里。本文是本机交接快照，不是所有主机都适用的安装规范。


## 12. 仓库配置对账与待办（2026-09-30）

本节补充工具、开发环境、运行时依赖、桌面设置和维护流程。依据为本仓库 README、
`bootstrap/arch.sh`、`run_*.tmpl`、实际配置与主题文档，并在本机做只读检查。
未安装软件、切换源目录、执行 apply、重启服务或提交。勾选表示完成；未勾选项按优先级推进。

### 已核对的基线

- [x] Arch bootstrap 的 `PACKAGES` 数组内 **44 个包全部已安装**，包括桌面、字体、输入法、编译依赖与 tree-sitter-cli。
- [x] README 另列的 ripgrep、yazi、gitui、glow、Neovide，以及 Zed 已安装。
- [x] Oh My Zsh、zsh-autosuggestions、pi CLI 与 `~/bin/pi-config` 已存在。
- [x] PipeWire、WirePlumber、桌面 portal、Mihomo 与蓝牙服务 active；system/user failed unit 均为 0。
- [x] Waybar、mako 进程正在运行。Waybar 由 niri 直接启动，不能用 `waybar.service` inactive 判断状态栏故障。
- [x] Waybar 窗口模块实际文件 `~/.config/waybar/waybar-niri-windows.so` 已存在；gpu-watch 已编译。
- [x] Neovim 插件目录及 Mason 工具已存在，Mason bin 中有 clangd、OmniSharp、pyright、rust-analyzer、vtsls、biome、Lua/JSON 服务和格式化工具。后续仍需验证实际语言项目。
- [x] UTF-8 locale 可用；实际输出 HDMI-A-1 为 3840×2160 @ 60 Hz、scale 1.5。
- [x] 当前 chezmoi 源仍为 `~/bin/dotfiles`，其配置层 diff **0 行**、status 为空。

当前源基线为 `e2f7f03`，本仓库为 `2c5e833`。当前源有本机未提交适配；对本仓库执行
只读 `chezmoi --source "$PWD" diff --include=files`，得到 **23 个文件差异**。
旧源干净只表示 home 与旧源一致，不表示新功能已部署，也不表示所有动作脚本成功。

### P0：先保留本机适配，再接入新配置

- [ ] **逐项合并两个源目录的差异**。保留旧源未提交改动和旧源里的接入记录，不以新仓库整份覆盖。对 23 个文件差异判定方向，另核对 run 脚本、`create_` 目标与 `.chezmoiignore` 差异；源层变化不一定出现在文件 diff 中。
- [ ] **保留显示、授权与空闲锁屏设置**。旧源含 HDMI-A-1 的 1.5 倍缩放、polkit-gnome 自启及 niri-idle.service；新仓库没有相同本机适配。确认合并后仍保留 600 秒锁屏、锁后熄屏及系统睡眠禁用，不套用旧 MacBook 的显示/合盖策略。
- [ ] **保留 Chromium 与本地代理接线**。新仓库 MIME 默认值、Chrome desktop、代理绕过列表与旧源不同；继续使用本机 Chromium，验证浏览器私网直连、终端 pon/poff 和 GitHub SSH。不要让合并恢复已经处理过的问题。
- [ ] **对账输入法和字体策略**。新源 environment.d、fontconfig 与旧源不同；旧接入记录采用 niri 内设置变量以保留 GNOME/IBus，而 09-29 又修过候选框。以现有文件和 niri/备用 GNOME 的实际表现为依据，验证中文显示、候选窗、终端和浏览器输入。
- [ ] **完成接入验收**。形成审阅过的合并源后，确认 sourceDir；应用前审阅配置 diff 和待跑脚本，应用后 diff 为空、第二次 apply 零输出。失败脚本记账不能当作成功，先排障再安排重试。

以上需跨越未对账机器 apply、删除或大规模重命名、修改 bootstrap/排除规则、提交推送时，按 AGENTS.md 先取得相应操作授权。

### P1：补齐已配置功能依赖

| 待办 | 本机证据与影响 | 完成判据 |
| --- | --- | --- |
| [ ] btop | 未安装；Waybar CPU/内存点击调用 `ghostty -e btop` | 两个模块点击都能打开监控 |
| [ ] bluetuith | 未安装；蓝牙模块点击调用它。不是 blueman-manager | 能打开蓝牙管理；不据此宣称适配器配对问题已解决 |
| [ ] pavucontrol | 未安装；扬声器/麦克风右键调用它 | 两个入口可用，确认实际播放设备及录音源 |
| [ ] Rust stable 工具链 | rustup 已装，但 `rustup show active-toolchain` 报 no active toolchain | cargo/rustc 版本可用，并完成一次小程序编译 |
| [ ] bun | PATH 和 `~/.bun/bin/bun` 均不存在 | bun 可运行，新交互 shell PATH/补全正确，安装器不追加改写受管 zshrc |
| [ ] nvm + Node 策略 | `~/.nvm/nvm.sh` 缺失；系统 Node v26.10.0、npm 12.1.0 已可用 | 按 docs/dev-env.md 决定是否加入 nvm/LTS，核对新 shell 中 Node 来源与 npm prefix；无需先卸载系统 Node |
| [ ] 新的开发环境脚本 | 当前源没有 `run_onchange_before_12-dev-runtimes.sh.tmpl` | 合并后审阅并执行，确认各工具来源、PATH 与后续维护方式 |
| [ ] AUR 安装入口 | yay 缺失；部分 AUR 包此前手工 makepkg 安装 | 确立可重复的 AUR 安装/更新方式；采用仓库路线时补 yay，不重跑整个 bootstrap |

uv 0.12.19、Go、GCC、make、pkgconf 和 GTK3 已可用。不要因 uv 位于 `/usr/bin` 就重复安装；
也不要把 `/usr/bin/cargo` 存在误记成 Rust 工具链可用。仓库开发环境脚本会安装 nvm 自己的
Node，系统已有 Node 不会使该步骤短路，执行前应明确版本来源。

### P2：新能力与按需依赖

| 待办 | 状态与范围 | 完成判据 |
| --- | --- | --- |
| [ ] voxtype 完整链路 | voxtype-bin、wordseg-rs、模型目录、用户 unit、键盘 udev 规则均缺失；只有配置文件存在 | ONNX 后端、paraformer-zh、分词过滤器、uaccess 和用户服务齐全；F9 中英文录音、识别、粘贴成功 |
| [ ] 语音键盘与输出配置适配 | 仓库 udev 规则按其他机器设备匹配；`paste_keys=ctrl+shift+v` 面向终端，而此会话输入框使用 Ctrl+V | 按本机实际键盘标识授权；测试浏览器和终端粘贴，避免误触图片粘贴；确认麦克风真的能录音 |
| [ ] WirePlumber 默认麦策略 | 新仓库优先级规则尚未部署；当前有内建模拟录音源，但没有录音实测 | 本机麦克风录音正常，再决定是否需要规则；不照搬 AU05 专属规则 |
| [ ] niri-session-mng | 当前未部署脚本/配置及新版 shell 宣告逻辑 | save/show 与 restore --dry-run 正常，正确识别 cwd/前台命令；真实恢复安排在合适桌面状态 |
| [ ] pi 管理项同步 | CLI 和扩展仓库已在，新仓库增加 settings 同步脚本与扩展设置 | 核对受管 settings 项及扩展加载，保留 provider/model 等本机状态，不输出认证资料 |
| [ ] Xwayland 修复版 | 当前为官方 xwayland-satellite 0.8.2，未装 -git | 如需仓库描述的 Steam/X11 弹窗修复，核实上游现状后选包并验证；替换涉及卸包时先授权 |
| [ ] Yazi 预览/解压 | ffmpeg、pdftoppm 已在；7z/7zz、chafa、magick 缺失 | 按文件使用需求补齐对应依赖，实测压缩包、图片和 PDF 预览 |
| [ ] Zed 会话恢复入口 | 已有 zeditor，白名单却使用 `zed` | 核对实际 zed 命令/别名与 desktop 启动方式，恢复时能正确启动 |
| [ ] 通知与 portal 实测 | mako 进程与 portal active，只验证了运行状态 | 普通通知、文件选择器和需要时的屏幕共享端到端可用 |

### P2：系统策略与长期维护

- [ ] **显卡与稳定性**：RTX 4060 当前仍使用 nouveau，nvidia-smi 缺失；这不是基础安装漏装。gpu-watch 依赖 NVML，编译成功不意味着 GPU 数据可用。结合隔夜故障调查是否调整驱动，先查当前 Arch/NVIDIA 官方方案；改驱动和恢复睡眠分开验收，继续观察现有禁睡眠策略。
- [ ] **代理维护**：延续第 7 节未完事项，完善订阅更新、校验与回退，验证面板和视频播放，评估临时 GitHub hosts。overlay、`clash-geoip`、loopback 和 `allow-lan` 都用 [`mihomo.md`](mihomo.md) 重读，不要把某次检查的缺包或 `allow-lan` 现值当成长期策略。仓库只打算钉 controller loopback 与 `mixed-port`；secret 不进文档。
- [ ] **TTY 字体**：当前 `/etc/vconsole.conf` 的 FONT 为 default8x16，未采用 bootstrap 的 ter-v16n。需要一致的 TTY 外观时再配置；它不负责桌面中文字体。
- [ ] **磁盘温度**：`/etc/modules-load.d/drivetemp.conf` 不存在；本机磁盘为 NVMe，先看 Waybar/sysfs 是否正常取温度，再决定是否需 SATA 专用 drivetemp，不盲补。
- [ ] **sudo 缓存策略**：没有 00-global-timestamp，这是先前明确保留的差异；不会阻碍普通 chezmoi apply。仅在需要跨终端的提权缓存时评估，不作为必装依赖。
- [ ] **Bolt 验收**：等待接收器，先验证鼠标键盘，再按需安装 Solaar；蓝牙研究继续暂停。
- [ ] **开发验收**：以实际项目验证 Neovim LSP、Tree-sitter、格式化与编译。OmniSharp 文件存在不证明运行时齐全，dotnet 尚缺；仅在需要 C# 时补。Python 已有，pip 缺失不影响采用 uv 的项目，无需为此补全局 pip。

### P3：参考清单中的可选软件

`docs/reference-terminal.md` 与 `docs/reference-desktop.md` 明确是外链参考，**不是装机必装清单**。
Obsidian、mpv、btop 等虽被提及，优先级应按实际配置调用和使用需求决定：btop 已列 P1；
Obsidian、mpv 本机未安装，可按需补。其他记录/远程桌面/下载/密码管理工具同样按需。

- [ ] 若需要笔记、影音、录屏、远程访问等，再从参考清单选应用并记录配置与验收。
- [ ] AU05、Apple suspend/audio 修复均为硬件专项；本机不是 MacBook，尚无 AU05 使用证据，不作为缺项补装。
- [ ] niri 自带截图且已配置 Print/Ctrl+Print/Alt+Print；grim/slurp 缺失不导致当前截图功能缺依赖，只有需要额外截图流程时再考虑。

推进顺序：P0 源与本机设置对账 → P1 现有入口及开发运行时 → P2 按需功能与维护验收 → P3 可选应用。
每项更新状态时记录实际验证结果，区分“包已安装”“配置已部署”“功能已验证”。


## 13. 配置同步与环境安装进展（2026-09-30）

用户已指定当前仓库为唯一源，并授权删除旧副本、同步配置及安装环境/工具。

### 已完成

- chezmoi sourceDir 切换为当前仓库；旧 `~/bin/dotfiles` 已删除。
- 旧副本与原 chezmoi 配置归档到 `~/.local/state/dotfiles-sync/20260930-172601/`，不入 Git。
- 用本机 `[data] localDesktop = true` 保留 HDMI-A-1 4K/1.5、polkit 代理、niri-idle、niri 输入法环境与隐藏睡眠入口。标记未设置时使用公共基线。
- 保留 Chromium 默认浏览器、代理绕过规则、shell 自动 pon、锁屏互斥/显示器恢复、Neovim 下载并发限制。Git helper 改为 PATH 上的 gh。
- 新配置、字体回退、会话管理脚本、pi 受管 settings 与扩展设置已同步。
- Rust stable **1.98.1** 已初始化并编译运行小程序；nvm **0.40.7**、Node LTS **24.21.0** / npm **11.19.0**、bun **1.4.2** 已安装。uv **0.12.19** 保留系统安装。
- 原 `.npmrc` 的 prefix 与 nvm 冲突，先备份再移除该项；新 npm 全局目录属于 nvm 当前 Node。原用户目录中的 pi/zed 命令仍可用，未卸载系统 Node。
- wordseg-rs **0.2.1** 已安装到 `~/.local/bin`。
- voxtype Paraformer 中文模型已下载；AUR voxtype-bin **1.1.0-2** 已通过 SHA/PGP 校验并构建，蓝牙 bluetuith **0.2.7** 与 yay-bin **13.0.1** 安装包已构建。
- AU05 动作脚本检测硬件后才启用服务；本机没有 AU05，因此服务保持 disabled。
- 增加实际 USB 键盘 `1321:5721`、`30fa:2031` 的 voxtype uaccess 规则，系统部署仍需提权。

### 已验证

- 当前源 `chezmoi diff --include=files`：0 字节。
- 复跑普通 `chezmoi apply`：退出码 0、输出 0 字节。
- verbose apply 成功，但每次显示 run_after_55 的脚本正文（3168 字节），与仓库“第二次 -v 零输出”的字面要求不一致；脚本没有同步变更。未为隐藏输出改变其每次同步语义。
- niri 本机分支和默认分支均渲染并通过 validate；zshrc/niri-lock 渲染与目标一致。
- 新 shell 中 Node/npm/bun/uv/Rust 命令可用；Node/bun 执行、Rust 编译运行成功。
- Neovim headless 启动成功，Zed 命令可用；会话管理 save 与 restore --dry-run 成功，未真实恢复窗口。
- niri-idle active、默认浏览器 chromium.desktop；未测试自动锁屏、睡眠、录音或重启桌面。

### 剩余安装与验收

系统包安装需要本机 polkit 密码认证；sudo 当前没有可用缓存。已经准备系统包事务：
btop、pavucontrol、7zip、chafa、ImageMagick、PipeWire ALSA、GTK4 layer shell，及已校验的
yay/bluetuith/voxtype 安装包。本机认证完成后部署 udev、选择 ONNX 后端并启动 voxtype 用户服务。
不能将构建完成记为系统安装完成，模型下载也不代表录音与 F9 粘贴通过验收。

xwayland-satellite 仍用官方版；显卡驱动、睡眠禁用、代理系统服务策略未调整。
clash-geoip/代理 overlay、Bolt/Solaar、C# dotnet、可选桌面应用仍按第 12 节用途推进。


## 14. 系统工具安装完成与重复锁屏修正（2026-09-30）

本机认证完成后，系统安装事务成功：btop 1.4.7、pavucontrol 6.2、7zip 26.03、
chafa 1.18.3、ImageMagick 7.1.2.32、yay 13.0.1、bluetuith 0.2.7、voxtype 1.1.0，
以及 PipeWire ALSA 与 GTK4 layer shell。此次 pacman -Syu 同时更新 uv 至 0.12.20，
官方 xwayland-satellite 至 0.8.3；未替换为 AUR -git，Steam 弹窗效果仍未实测。

voxtype 已选择 ONNX AVX2，模型加载成功；用户服务 enabled/active，状态 idle，
成功打开本机两种 USB 键盘的 4 个 event 节点监听 F9，udev ACL 已生效。
`voxtype setup check` 通过。未主动录音或注入文字，实际麦克风及应用粘贴仍待使用验收；
当前 Ctrl+Shift+V 面向终端，浏览器输入框采用 Ctrl+V，不能假定所有应用同一键位可用。

用户报告每次解锁立即又锁一次。日志两次确认“成功解锁后同秒启动新 hyprlock”。
外层 `swayidle -w` 在前台 locker 整段运行期间阻塞事件分发，解锁后积压请求才执行，
而此时 flock 已释放。已移除 niri-idle 的 `-w`，让并发请求在锁屏存活时被 flock 拦截，
并重启该 idle 服务。服务 active，运行参数已更新，600 秒空闲锁屏仍保留。
依据、推断范围和单独的 wl_shm 退出错误记录见 [lockscreen.md](lockscreen.md)。
未主动锁屏打断工作，下一次真实空闲解锁需确认只出现一次认证。

最后配置 diff 为空，普通复跑 apply 零输出；用户级 failed unit 为 0。
未提交或推送。第 12/13 节保留历史检查结果，本节状态覆盖此前的未安装/待认证描述。


## 15. 使用反馈与剩余待办（2026-09-30）

- [x] 用户确认重复解锁已经解决，现在只需要输入一次密码。
- [x] 候选乱码排查发现 Fcitx5 仍映射已经删除的旧 Sarasa 字体；已刷新缓存并真正重启输入法，新进程不再持有旧字体映射。
- [ ] 候选字显示需用户重新输入拼音确认；若仍异常，继续区分字体图形异常与 Rime 候选文本异常，不盲删词库。
- [ ] 用户暂未准备麦克风，语音端到端验证延期；程序、模型和服务已就绪，不主动录音。
- [ ] 代理订阅更新/回退、控制面板端到端操作、临时 GitHub hosts 的清理评估仍未完成。
- [ ] Nouveau 显卡与睡眠恢复根因仍未修复；继续保持禁睡眠，锁屏改善不等于睡眠可靠。
- [ ] Bolt 接收器到位后再验证罗技设备，必要时加入 Solaar；蓝牙驱动研究继续暂停。
- [ ] Xwayland 官方 0.8.3 的实际 X11/Steam 弹窗表现、真实会话恢复、具体语言项目 LSP 等按使用需求验收。
- [ ] 源改动尚未提交或推送；执行这些操作仍需用户明确授权。

没有发现阻碍当前基础桌面使用的额外必装工具；其余参考应用、C# 运行时等按项目需要安装。


## 16. 输入法问题暂停与 Unity 调研（2026-09-30）

用户反馈重启 Fcitx5 后候选仍乱码。旧字体映射虽已清除，问题未解决，不能据此认定字体
升级是全部根因。按用户要求暂停排查；后续需区分候选窗口的字形问题与候选文本本身异常。
语音仍等麦克风准备后验收。

用户另要求研究当前系统安装 Unity 编辑器。本轮仅核对本机条件与官方安装资料，未安装 Hub、
下载 Editor、激活许可证或更换显卡驱动。Arch x86_64、约 62 GiB 内存及约 302 GiB 空闲磁盘
满足基础硬件规模；Unity 6.3 官方 Linux 编辑器支持组合是 Ubuntu 22.04/24.04、GNOME、
NVIDIA 官方驱动或 AMD Mesa，当前 Arch/niri/Nouveau 不在该组合内。
官方提供 Hub AppImage，AUR unityhub 将官方 deb 重打包；安装可行性与图形运行稳定性分开验收。

参考：[Hub Linux 安装](https://docs.unity.com/en-us/hub/install-hub-linux)、
[Unity 6.3 系统要求](https://docs.unity3d.com/6000.3/Documentation/Manual/system-requirements.html)。


## 17. 指定 Unity 版本安装（2026-10-10）

用户授权安装，并指定编辑器 **6000.0.73f1**；没有安装此前建议的 6.3 LTS。
官方 Unity Hub AppImage **3.21.3** 安装到 `~/.local/opt/unityhub/UnityHub.AppImage`，
Hub GUI 已打开。chezmoi 管理 unityhub 命令包装、应用菜单入口、官方图标与 unityhub URL handler。

编辑器官方下载包约 4.50 GB、展开约 8.48 GB，changeset `a166abc3bf0e`。
下载记录、大小和 MD5 取自 Unity 官方发布 API，并对完整包校验匹配。
代理访问 download.unity3d.com 会重定向到缺文件的 unitychina 镜像，导致 CLI 和 curl 的 404；
直连官方地址正常。直连曾出现连接停顿/重置，改为分段断点续传后完成下载。

编辑器安装到 `~/Unity/Hub/Editor/6000.0.73f1/Editor/Unity`，已经通过 Hub 的
`editors --add` 注册，Hub 返回该版本与安装位置。额外 Android/Web/Windows 等模块未安装。
ldd 发现缺 `libxml2.so.2`，已准备安装 Arch `libxml2-legacy`，正在等待本机密码认证。
账号/许可证由用户在 Hub 登录激活；未读取或记录认证凭据。显卡驱动未改变。
尚未完成原生编辑器启动及项目渲染验收，不把注册成功视为运行成功。

安装缓存暂保留于 `~/.cache/unity-install/`（完整包及续传分段），以便排障。
官方参考：[6000.0.73f1 发布记录](https://unity.com/releases/editor/whats-new/6000.0.73f1)。


## 18. Android 构建支持安装与验收（2026-10-20）

用户已登录 Unity Hub 并激活许可证，要求在 **6000.0.73f1** 中加入 Android 构建支持。

- `libxml2-legacy 2.13.9-2` 已通过系统认证安装；编辑器 `-version` 返回 6000.0.73f1。
- 初次 Hub CLI 追加模块因手动定位的编辑器缺少 modules.json 而拒绝。读取该版本官方 CDN 的
  release.json/unity-linux-x86_64.json，按照当前 Hub 的格式补齐模块清单和 metadata.hub.json，
  所有模块起初标记为未安装，再由 Hub CLI 实际安装；未更改 Hub 程序或账号状态。
- 通过 `unityhub --headless install-modules --version 6000.0.73f1 --module android --childModules`
  完成 Android 全套组件安装。Hub 报 All Tasks Completed Successfully，11 项 Android/CMake
  记录均由安装器更新为 installed：Build Support、OpenJDK 17.0.9、NDK r27c
  （27.2.12479018）、CMake 3.22.1、Build Tools 36.0.0、Platform Tools 36.0.0、
  SDK Platforms 34/35/36 与 Command Line Tools 16.0 等。
- 配套 java/adb 已运行检查。编辑器 batchmode 启动并成功完成许可证校验。
- 首次启动检查漏指定项目路径，Unity 在仓库生成了空项目目录；已将此次新生成的
  Assets/Library/Packages/ProjectSettings/Logs/UserSettings 全部移到
  `~/.local/state/unity-install/android-check-project`，未删除文件或动原有源改动。
- 用上述外部测试项目执行 **Android ARM64 + IL2CPP** 的真实 APK 构建。首次 Gradle 直连
  Maven 仓库停顿；Unity 不向 Gradle 子进程保留 JAVA_TOOL_OPTIONS，因此改为本机
  `~/.gradle/gradle.properties` 的 systemProp.http/https.proxyHost/Port，走已有 Mihomo 7890。
  修改前若有配置则备份到本机 unity-install state 目录；该机器代理配置不入公共源。
- 重试成功生成 `Build/AndroidInstallCheck.apk`；编辑器进程退出码 0，日志包含
  ANDROID_INSTALL_CHECK_SUCCEEDED。构建日志位于
  `~/.local/state/unity-install/android-build-check.log`。没有修改用户的实际游戏项目，
  未签生产包、连接设备或安装 APK 到手机。

Hub 的 Android 模块安装与工具链验收已完成；Nouveau 下的编辑器交互图形稳定性仍不等于
本次无图形构建验收。首次使用真实项目时仍应核对该项目需要的目标 API、架构和签名设置。


## 19. 磁盘识别、数据分区访问与挂载（2026-10-20）

### 已确认的布局

| 设备 | 类型与标称容量 | 分区及用途 |
| --- | --- | --- |
| `/dev/nvme0n1` | Kingston NV2，1 TB NVMe SSD | p3 为当前 Linux 根分区；p5 为约 586 GiB 的 NTFS 数据分区 |
| `/dev/sda` | Seagate，8 TB SATA 机械硬盘 | sda2 为 NTFS 数据分区，当时未挂载 |

本机识别到两块物理硬盘，只有第一块是 SSD。Unity Hub、指定编辑器、Android 组件和本机
测试项目均在 SSD 的 Linux 系统分区；8 TB 机械盘未用于此次安装。
根分区当次查询为总量约 338 GiB、已用 48 GiB、可用 273 GiB；这是历史读数，容量以
`df -h /` 的当前结果为准。两个数据分区卷标均为“新加卷”，不能只凭卷标区分。

### 挂载实施与验收

`/dev/nvme0n1p5` 起初未挂载，因此无法 `cd` 到它；`/dev/...` 是块设备文件，不是目录。
用户指定挂载到 `~/data`，先确认该路径不是符号链接、没有现有内容，再创建挂载点。
实际挂载按本机查询出的分区 UUID 定位，使用内核 ntfs3，指定当前用户 uid/gid 与 umask=022。
系统操作通过 pkexec 发起，由用户在本机 polkit 弹窗认证。

复用流程（先用 lsblk 确认本机设备，不能把设备编号当作跨机器固定值）：

```sh
lsblk -o NAME,SIZE,FSTYPE,LABEL,UUID,MOUNTPOINTS
# Verify the target is empty before mounting.
mkdir -p "$HOME/data"
pkexec mount -t ntfs3 -o "uid=$(id -u),gid=$(id -g),umask=022" /dev/nvme0n1p5 "$HOME/data"
findmnt --mountpoint "$HOME/data" -o SOURCE,TARGET,FSTYPE,OPTIONS
cd "$HOME/data"
```

实际验收：findmnt 显示 p5 → `~/data`、ntfs3、rw，用户可读写，目录枚举成功。
**本次是临时挂载**，未写 fstab，重启后不会自动恢复；持久挂载仍是待办。
不能为了修复挂载错误而强制清除 NTFS 脏标记、格式化或覆盖已有目录内容。

## 20. 内网 Git 远端、SSH 认证与只读验证（2026-10-20）

### 结果与边界

用户指定的数据分区工程包含一个主仓库远端和 7 个已初始化子模块，全部连接同一内网 Git 主机。
检查时本机 DNS 已解析到用户指定的内网服务器，SSH/HTTP 可达，因此 **未修改 hosts**。
公司域名、内网地址、实际仓库路径和逐项结果保存在本机受限报告中，不复制到公共配置：

`~/.local/state/git-remote-checks/20261020/remote-check.json`（父目录 0700，文件 0600）。

首次 SSH 连接尚无 known_hosts 记录。按首次使用信任流程以 `accept-new` 建立主机密钥记录，
之后改用严格校验；没有关闭校验或替换冲突密钥，也没有取得管理员提供的独立指纹验证。
随后服务器拒绝本机公钥；用户在该 Gitea 账户添加已有公钥后，`ssh -T` 成功认证。
GitHub 的既有授权不能替代这台内网 Gitea 的授权。

最终结果：主仓库及 7 个子模块的 **8/8 读取测试、8/8 推送通道 dry-run 均通过**。
主仓库当前 `Demo/Gamescom` 分支的常规 dry-run 返回 fetch first，表示远端存在本地缺少的
提交，当前不能直接快进推送；不是域名、SSH 或认证故障。没有执行 fetch/pull、合并、
实际 push、创建远端分支或改动工程文件。

### 可复用检查流程

1. 核对主仓库所有 remote 的 fetch/push URL、insteadOf/pushInsteadOf、core.sshCommand，
   并检查 `.gitmodules` 与实际初始化子模块的 remote；不能只看主仓库的 origin。
2. 通过 `getent hosts`、SSH 有效配置 `ssh -G` 与端口连接区分 DNS、路由、端口问题；
   将 Git SSH 和浏览器 HTTP 分开验证。
3. known_hosts 已建立后使用 BatchMode、ConnectTimeout 和 StrictHostKeyChecking=yes，
   防止检查挂在密码提示上；认证失败时先修公钥授权，不重复尝试所有仓库造成无意义失败。
4. 对每个实际 remote 执行 `git ls-remote --symref <remote> HEAD`，确认读取与默认分支。
5. 推送通道使用 `git push --dry-run --no-verify --porcelain <remote> HEAD:<临时检查ref>`。
   显式 HEAD refspec 可用于 detached HEAD 的子模块；使用唯一检查 ref 避免现有分支进度
   干扰；dry-run 不写远端，`--no-verify` 避免运行本地 pre-push hook。
6. 区分“receive-pack 通道可达”与“实际目标分支允许更新”。dry-run 不证明保护分支策略
   一定允许正式推送；fetch first 也不能通过 force push 绕过。

本次只检查 Git 配置和连接，没有启动 Unity、切换版本或平台，没有进入工程代码修改流程。

## 21. 浏览器内网域名无法访问（2026-10-20）

Git SSH 认证修好后，用户反馈浏览器无法识别同一域名。对照测试：系统解析正确，HTTP 直连
返回 **200**，经本机 HTTP 代理访问返回 **502**。浏览器原有绕过列表只包含私网 IP 网段、
本地名称等，不能据此假定域名形式的 URL 会自动直连。

处理：将具体域名加入本机 chezmoi.toml `[data] browserProxyBypass` 列表；Chrome 与
Chromium flags 模板统一读取该列表，缺省为空，不给其他机器写入这台机器的公司域名。
`chromium-flags.conf` 为此改成模板，Chrome 原有显卡分支保留。未改 bootstrap、平台排除
规则、系统 DNS、hosts 或 Mihomo 订阅内容。

验证顺序：渲染模板并审阅 diff → apply → 渲染与目标逐字比对 → 独立临时 profile 的
headless Chromium 打开站点，确认拿到 Gitea 页面 → 用户保存工作、彻底退出浏览器再打开。
**用户已确认重启后域名有效**。该站点当时只有 HTTP，访问时需使用明确的 `http://` URL。
已有主进程不会因新开窗口重读 flags，不能把新开窗口当作代理变更已生效。

公共配置和重启说明见 [chrome.md](chrome.md)；实际域名只留本机 data 与受限报告。

## 22. 浏览器 profile 弹窗排查（2026-10-20）

用户报告代理唤起浏览器时反复出现 `Profile error occurred` /
`Something went wrong when opening your profile. Some features may be unavailable.`。

已完成的检查：

- 当前 Chromium 与 profile 的 Last Version 均为 **153.0.8010.52**，profile 文件属于当前用户。
- 浏览器打开期间，直接只读访问 Web Data/History/Favicons 返回 database is locked；
  这本身不能作为数据库损坏证据，不能据此删除文件或锁。
- 在本机 0700 诊断目录中复制相关配置/数据库及其 journal/WAL 边车后检查，Web Data、History、
  Favicons、Login Data、Login Data For Account 的 quick_check 均为 ok；只读取完整性和 schema
  元数据，不打印密码、Cookie 或历史记录内容。活动数据库副本仍有并发快照局限。
- 诊断副本启动 headless Chromium 成功，未捕获 profile 数据库初始化错误。
- 按用户要求实际执行 `chromium --new-window about:blank`，并通过系统默认入口
  `xdg-open <内网站点URL>` 再次打开页面；调用日志为 Opening in existing browser session，
  新窗口截图没有该弹窗。**用户确认这次没有出现**。

结论：本轮没有复现故障，具体根因未定；版本、权限、检查过的数据库副本均未给出损坏证据。
不能把“重启后未复现”写成已经找到并修复数据库问题，也不能断言一定是旧进程、代理或密钥环。
未清理、重置或删除正在使用的 profile，没有更改书签、登录状态或密码库。
诊断副本位于 `~/.local/state/chromium-profile-diagnostics/<时间>/`，含浏览器私密数据，不入源。
若再次出现，优先保留弹窗和当时进程，再采集实际启动日志，而非先泛化清理整个 profile。

## 23. 后续操作流程与当前待办（2026-10-20）

### 已确认完成

- [x] 当前仓库作为唯一 chezmoi 源，旧源备份后删除。
- [x] 基础工具与开发运行时安装；重复解锁修正且用户确认有效。
- [x] Unity 指定版本、Android 全套组件、许可证与 ARM64/IL2CPP APK 构建验收。
- [x] 两块物理硬盘及 SSD 数据分区识别；用户指定的 `~/data` 临时挂载可读写。
- [x] 内网 Git 主仓库/7 子模块的读取与推送通道验证。
- [x] 浏览器域名直连配置，用户重启确认有效。
- [x] profile 弹窗本轮复现检查完成，结果为未复现；不等同根因修复。

### 仍需处理或按需求启动

- [ ] 中文 IME 候选乱码：用户要求暂停，重启输入法未解决，不记为已恢复。
- [ ] 麦克风准备后再验证语音输入的录音、识别和应用粘贴。
- [ ] `~/data` 如需重启后持续可用，再配置并验收持久挂载；当前没有自动挂载承诺。
- [ ] 主仓库目标分支正式推送前先评估本地改动与远端提交，再安排同步；本轮未自动 pull。
- [ ] profile 弹窗若复现，保留现场继续定位；目前保留原 profile。
- [ ] 代理订阅更新/回退、面板验收、临时 GitHub hosts 清理、Nouveau 睡眠恢复、Bolt 设备验收
  等仍沿用此前待办，不因 Git 或浏览器连通而视作全部完成。
- [ ] Git 提交和推送仍需用户明确授权，本次仅更新文档，不提交或推送。

### 本次工作沉淀的通用顺序

先核对本机真实状态及源目录 → 保留现有改动、限定操作范围 → 通过最小对照测试区分
网络/认证/运行库/配置问题 → 修改真源或明确属于本机的设置 → 渲染和审阅配置 diff →
应用并检查 no-op → 使用真实功能验收 → 记录证据、限制和剩余项。
系统提权由本机 polkit 认证完成；会话工具权限不等于 root 权限。不能把“等待用户输入密码”
当作安装成功，也不能把“文件已下载”“Hub 已注册”“构建成功”混为同一个验收阶段。
