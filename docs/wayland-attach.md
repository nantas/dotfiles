# 从 ssh / tty 附加到 niri 会话

`dot_zshrc.tmpl` 里 Linux 段的 `wattach` / `wdetach` / `wstat`，跟同一份文件里的
`pon` / `poff` / `pstat` 是一套的写法：后者开关代理，前者开关「这个 shell 看不看得
见座位上的图形会话」。

## 要解决的问题

ssh 进来（或坐在 tty 上）的 shell，环境里没有 `WAYLAND_DISPLAY`，`XDG_SESSION_TYPE`
是 `tty`。于是：

```console
$ grim /tmp/a.png
(compositor doesn't support the screencopy protocol / cannot connect to display)
$ niri msg outputs
Niri is not running
```

会话本身活得好好的，只是这个 shell 不知道往哪个 socket 说话。手工补四个变量就能用，
但 socket 名里带 niri 的 pid、重启一次就变，所以值得包成命令。

```sh
export XDG_RUNTIME_DIR=/run/user/1000
export WAYLAND_DISPLAY=wayland-1
export DISPLAY=:0
export NIRI_SOCKET=/run/user/1000/niri.wayland-1.1607.sock
```

## 为什么是 zsh 函数，不是 `~/.local/bin` 里的脚本

环境变量不能跨进程传递：子进程改了 `export`，父 shell 一无所知。一个独立可执行脚本
只能把 `export ...` 打印到 stdout 让调用方 `eval`，那是 `ssh-agent` 的做法，代价是
用户每次都得记住 `eval "$(wattach)"`。

`pon` / `poff` / `pstat` 本来就在 `dot_zshrc.tmpl` 里，直接同进程改环境的函数最省事，
也省掉「脚本 + 同名函数包装 + `command` 绕过函数」那层绕。代价是这个工具只能在 zsh
里用（本机 `SHELL=/usr/bin/zsh`，ssh 进来也是 zsh）。

## 变量从哪来

`systemctl --user show-environment` 是主来源。niri-session 启动时把合成器的环境导入
systemd 用户管理器，所以那份拷贝里本来就有整套东西：

```console
$ systemctl --user show-environment | grep -E 'WAYLAND|DISPLAY|NIRI'
DISPLAY=:0
NIRI_SOCKET=/run/user/1000/niri.wayland-1.1607.sock
WAYLAND_DISPLAY=wayland-1
XDG_CURRENT_DESKTOP=niri
XDG_SESSION_CLASS=user
XDG_SESSION_ID=2
XDG_SESSION_TYPE=wayland
```

坑：**不支持按变量名单独查询**，`systemctl --user show-environment WAYLAND_DISPLAY`
会以 1 退出并抱怨 `Too many arguments`。只能整份读进来再筛。整份输出里 `LS_COLORS`
之类的值用 systemd 的 `$'...'` 引号，本工具关心的那几个变量（路径、`:0`、`wayland-1`）
永远不会命中那种写法，所以没有做反转义。

别从 `/proc/<niri-pid>/environ` 读：本机 `/proc` 有 hidepid，同一个用户也读不到，
实测 `Permission denied`。

## 探测顺序与存活校验

1. 先信 systemd 那份的 `NIRI_SOCKET`；
2. 只在对它做存活校验之后；
3. 校验不过就扫 `$XDG_RUNTIME_DIR/niri.*.sock`（按 mtime，取最新的）。

第 3 步不是多余的：**niri 若绕开 niri-session 重启，systemd 那份会指向死掉实例的
socket**，名字里的 pid 是旧的。存活校验就是干这个的——socket 名由
`niri.<display>.<pid>.sock` 组成，取倒数第二段当 pid，`kill -0` 看进程还在不在，
再比对 `/proc/<pid>/comm` 是不是 `niri`，挡掉 pid 回收后的误判。

`DISPLAY` 不单独探测，直接抄会话环境里的值。niri 自己集成 xwayland-satellite 并导出
`$DISPLAY=:0`（见 [`xwayland-satellite.md`](xwayland-satellite.md)），不需要去翻
`/tmp/.X11-unix` 或猜 Xwayland 的 pid。

覆盖的变量：`WAYLAND_DISPLAY`、`NIRI_SOCKET`、`XDG_SESSION_TYPE=wayland`、
`XDG_SESSION_CLASS=user`，加上会话真有就一并抄来的 `DISPLAY`、`XDG_SESSION_ID`、
`XDG_CURRENT_DESKTOP`、`DBUS_SESSION_BUS_ADDRESS`。

## 撤销：为什么要存 undo

`wdetach` 把 shell 恢复成附加前的样子，而不是简单地 `unset` 一批变量。区别在于
`XDG_SESSION_TYPE`：ssh 会话原本是 `tty`，这是有意义的值，一律清成未设置会让别的
工具猜错。所以 `_wattach_apply` 第一次附加时把每个变量的原值（有值就
`export K=<引号转义后的值>`，没值就 `unset K`）拼成一段 shell 代码存进 `WATTACH_UNDO`，
`wdetach` 直接 `eval` 它。

两个由此而来的约束：

- **已经在附加状态的 shell 再次 `wattach`，不会覆盖 `WATTACH_UNDO`**。否则 undo 会把
  「附加后」的值记成原值，`wdetach` 就变成原地踏步。
- 没有附加过的 shell 上跑 `wdetach` 直接报错退出。在本地图形 shell 上误按 `wdetach`
  属于此类，报错比照着 `WATTACH_UNDO` 这个空变量乱清一气安全。

undo 是一段存在环境变量里的 shell 代码，只在同一个 shell 里由自己写入、自己 `eval`，
不落盘、不跨用户。

## 命令形式

```sh
wattach foot             # 在图形会话的环境里开一个终端
wattach grim /tmp/a.png  # 单条命令，不留痕
wattach -d niri-lock     # -d：脱离 ssh 会话，断线也不收 SIGHUP
```

这一支在**子 shell** 里 export，再 `exec` 目标命令：调用方 shell 的环境不受影响，
退出码照常传回。子 shell 里 `_wattach_apply` 的 stdout 被丢弃，免得 `[niri] Attached
to ...` 那行横幅混进命令自己的输出（比如接管道的时候）。裸调用的 `wattach` 才在父
shell 里 export，并打印那行横幅。

## 被拒绝的做法

- **写死 socket 路径**：名字里的 pid 每次重启都变。
- **独立可执行脚本 + `eval "$(wattach)"`**：见上，多一层记忆负担，本仓库也没有先例。
- **`machinectl shell` / `systemd-run --user --scope` 切进会话**：对「跑一条命令」可行，
  但对「让当前 shell 看见会话」无用——它要的正是改变调用方自己的环境。
- **`systemctl --user import-environment`**：方向反了，那是把本 shell 的环境推给
  systemd，不是把会话的环境拉进来。
- **从 `/proc/<pid>/environ` 读图形会话的环境**：hidepid 拦着，读不到。

## 限制

- 只认 niri。`docs/` 里其它合成器（hyprland、sway）的 socket 命名与 liveness 判据都不同，
  没做。
- 同用户有多个图形会话时只取一个（systemd 那份只有一个 `NIRI_SOCKET`，扫描取最新），
  多座位场景没考虑。
- 不带 `-d` 的命令形式会随 ssh 会话结束收到 SIGHUP；`wattach -d` 用 `setsid -f` 脱离。
  锁屏是这条路上后果最重的一个，见 [`lockscreen.md`](lockscreen.md)。

## 验证

```sh
wstat                    # 会话存在 + 本 shell 是否已附加
wattach
env | grep -E 'WAYLAND|NIRI|DISPLAY'
grim /tmp/a.png          # 真的连上了才算数
niri msg outputs
wdetach                  # XDG_SESSION_TYPE 应回到 tty
wdetach                  # 再跑一次应报错并返回 1
```

## Mac 通过 SSH 操作本机：逐步验收

此流程不需要远程桌面。Mac 上的 agent 通过 SSH 执行命令，用 niri IPC 读取窗口状态，
需要检查实际绘制效果时再捕获图片并读取。以下命令分为 Mac 端与登录后的 Linux 端，
每一步成功后再进行下一步。

### 0. 前提与当前验证范围

Linux 端必须满足：

- `sshd.service` 已启用并运行，局域网可以访问 SSH 端口。
- 使用运行 niri 的同一个普通用户登录 SSH，默认 shell 是 zsh，已部署本仓库的 `.zshrc`。
- 已登录 niri 桌面。SSH 服务开机启动不等于图形会话自动登录；停在 GDM 登录界面时，
  SSH 可管理系统，但 `wattach` 没有现成桌面可附加。
- 电脑处于开机、联网且未挂起的状态。锁屏与挂起不同，锁屏不代表 SSH 服务停止。

2026-10-20 本机检查：SSH 已启用并运行；移除图形环境变量后，模拟 SSH 环境的 zsh
能执行 `wstat`、`wattach`、`niri msg outputs` 与 `wdetach`，部署的 `.zshrc` 与模板一致。
当时未部署 Mac 公钥，可先使用 Linux 账户密码登录。Mac 到 Linux 的真实登录、截图传输、
窗口操作及重启／唤醒后的复验，仍需按下列步骤验收，不能以模拟测试替代。

### 1. 从 Mac 登录

在 Mac 终端设置实际目标，替换占位内容，不要把个人账号与地址写回公共配置：

```sh
remote='LINUX_USER@LINUX_LAN_IP'
ssh -t "$remote"
```

第一次连接应在 Linux 本机运行下列命令，核对 Mac 提示的主机指纹：

```sh
ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
```

密码提示需要的是 Linux 账户密码。公钥登录是可选的后续配置，应将 Mac 的公钥追加到
Linux 用户的 `~/.ssh/authorized_keys`，保留已有条目；不要传输私钥，也不要将身份文件
纳入仓库。使用 agent 无人值守执行 SSH 前，先验收公钥认证与主机指纹记录。

验收：能进入 Linux shell，`whoami` 和 `hostname` 对应预期目标。

### 2. 发现、附加和撤销图形会话

在刚登录的 Linux shell 中执行：

```sh
whence -w wattach wstat wdetach
wstat
wattach
niri msg outputs
wdetach
wstat
```

验收：三者都是函数；`wstat` 发现存活的 niri 会话；附加后能列出显示器；撤销后当前
shell 恢复附加前的环境。若提示 `no live niri session found`，先确认本机已登录 niri，
且 SSH 使用同一用户，不要硬编码旧 socket 路径。

### 3. agent 执行单条远程命令

回到 Mac 终端执行。SSH 的单条命令通常通过非交互 shell 运行，不加载 `.zshrc`，所以
必须显式启动交互 zsh 来取得函数：

```sh
ssh "$remote" "zsh -ic 'wstat'"
ssh "$remote" "zsh -ic 'wattach niri msg -j windows'"
ssh "$remote" "zsh -ic 'wattach niri msg -j workspaces'"
```

验收：获得窗口和工作区 JSON。窗口状态包含标题、应用标识、窗口 ID 等，不包含窗口内
的全部文字、控件结构或画面。每次 SSH 命令都是独立 shell，前一次裸调用 `wattach`
不会让后一次自动继承附加状态，因此每次使用 `wattach COMMAND`。

若远程 shell 启动插件出现 TTY 报错，交互诊断可用 `ssh -t`；但不要把 PTY 的控制字符
混入 agent 要解析的 JSON。应先确认无 PTY 的查询结果干净，再依赖自动解析。

### 4. 捕获实际画面并在 Mac 读取

本机提供 niri 内建截图命令，不依赖额外安装 `grim`。在 Mac 执行：

```sh
ssh "$remote" "zsh -ic 'wattach niri msg action screenshot-screen --path /tmp/wattach-screen.png'"
scp "${remote}:/tmp/wattach-screen.png" ./wattach-screen.png
open ./wattach-screen.png
```

验收：Mac 能看到当前聚焦显示器的画面；agent 也应通过自己的图像读取能力打开下载的
文件，不能只确认文件存在就认定界面正常。单条固定路径便于复验，多任务并发时应改用
各自唯一的路径，避免互相覆盖。截图命令也会更新 Linux 剪贴板，测试前留意正在使用的
剪贴板内容。

软件单窗口检查可以通过 ID 指定目标；先从 `niri msg -j windows` 获取 ID，再把下面
的 `123` 替换成实际数值：

```sh
ssh "$remote" "zsh -ic 'wattach niri msg action screenshot-window --id 123 --path /tmp/wattach-window.png'"
scp "${remote}:/tmp/wattach-window.png" ./wattach-window.png
open ./wattach-window.png
```

窗口截图不保证包含独立的输入法候选弹窗等其他表面。排查候选字、菜单遮挡或跨窗口
布局时优先截屏幕；必要时先聚焦目标窗口再捕获，并检查图片确实反映目标状态。
锁屏不能作为查看锁屏背后桌面的正常途径，验收画面时应确认桌面已解锁。

### 5. 操作桌面并再次读取结果

从窗口列表取得实际 ID 后，在 Mac 执行：

```sh
ssh "$remote" "zsh -ic 'wattach niri msg action focus-window --id 123'"
ssh "$remote" "zsh -ic 'wattach niri msg -j focused-window'"
```

验收：聚焦窗口的 ID 与目标一致，再按上一步截图检查。启动需要在 SSH 断开后保留的
图形应用，可执行：

```sh
ssh "$remote" "zsh -ic 'wattach -d ghostty-tab'"
```

验收：现有 Ghostty 有窗口时新增标签页；没有窗口时启动窗口。退出该次 SSH 后应用仍存在，
再查询窗口列表确认；标签页不会新增 niri 窗口 ID。`-d` 表示脱离 SSH 会话，
不表示开机自启，也不保证应用已成功启动；失败时应检查应用日志。
不要用锁屏、关闭窗口或重启桌面作为第一项控制测试。确需远程锁屏时使用
`wattach -d niri-lock`，原因见 [lockscreen.md](lockscreen.md)。

任意坐标点击、键盘注入和控件操作不是 `wattach` 的功能。优先使用应用自身 CLI/API、
浏览器 DOM 或可用的无障碍接口；这些接口是否可用需要逐个软件验证，不应仅凭能截图
就宣称任意 GUI 自动化已就绪。

### 6. 重启与唤醒后复验

按实际使用安排分别测试，不由 agent 为验收自动触发重启或挂起：

- 整机重启后先验收 SSH；本机登录 niri 后再验收会话发现、窗口 JSON 与截图。
- 挂起期间 SSH 不可用；本机唤醒且网络恢复后重新登录，重新运行 `wstat` 与截图步骤。
- niri 重启后重新执行 `wattach`，不要复用之前保存的 `NIRI_SOCKET`。

每项记录「通过／失败、时间、报错或图片」，至少分开记录：SSH 登录、图形会话附加、
窗口状态读取、屏幕截图传输、窗口截图、窗口聚焦、应用脱离 SSH 存活。连接超时先查网络
与服务；认证失败先查用户和凭据；图形查询失败再查会话环境。

## 默认复用标签页

Linux 上部署了 `~/.local/bin/browser-tab` 与 `~/.local/bin/ghostty-tab`，并在仓库
`AGENTS.md` 约定 agent 默认使用这两个入口。它们不是拦截所有进程的全局策略；其他仓库
或 Mac 端 agent 也应读取本节并遵循此约定。显式运行原始应用、桌面菜单的“新窗口”操作、
网站主动弹窗或自动化工具的独立浏览器实例，不会被这两个入口改写。

```sh
browser-tab https://example.com
browser-tab                         # Open a blank tab.
ghostty-tab
ghostty-tab --list                   # List GTK window IDs.
ghostty-tab --window 7               # Use an ID from --list.
```

从 Mac 执行时仍需按前文设置 `remote`：

```sh
ssh "$remote" "zsh -ic 'wattach browser-tab https://example.com'"
ssh "$remote" "zsh -ic 'wattach -d ghostty-tab'"
```

`browser-tab` 通过系统 `/usr/bin/chromium` 打开 URL，沿用其 flags 和默认用户目录，
不传 `--new-window`，无参数时传入 `about:blank`，拒绝把浏览器启动选项当作 URL 传入。
默认用户目录已有普通窗口时，Chromium 将 URL 交给现有会话；没有时才创建窗口。
多 Chromium 窗口时由浏览器选择接收窗口，不保证始终是同一个窗口。若现有实例使用了
另外的 `--user-data-dir`、独立 profile、访客或隐身会话，不能保证与默认入口合并。
不要给常规 agent 浏览加临时用户目录；浏览器自动化若要复用用户实例，需要单独验证
该工具的连接方式，本入口不等于浏览器自动化或 CDP 连接器。

`ghostty-tab` 通过会话 DBus 查询默认 Ghostty 实例的 GTK 窗口，并触发该窗口的
`org.gtk.Actions` 的 `new-tab` 动作。默认选择 GTK ID 最大的窗口，也就是该实例当前
最新创建的窗口；不声称是当前聚焦窗口。可用 `--window` 明确指定，ID 取自 `--list`，
不是 niri 的窗口 ID。没有已有窗口时才通过 `/usr/bin/ghostty --gtk-single-instance=true`
启动窗口。依赖 Python PyGObject/Gio 和 GTK 版 Ghostty 的窗口动作接口，本机已具备。
自定义 Ghostty `class` 或 `--gtk-single-instance=false` 的独立实例不在此入口的选择范围。

普通 `ghostty` 启动命令和 `gtk-single-instance=true` 只保证复用应用进程，不保证复用
窗口为标签页。本入口不支持 `-e`、工作目录和其他启动参数；直接执行任务优先使用 SSH，
不要把命令伪装成终端标签页参数。窗口动作可参考
[Ghostty GTK 实现](https://github.com/ghostty-org/ghostty/blob/main/src/apprt/gtk/class/window.zig)。

本机 2026-10-20 实测两个入口各调用一次后，Chromium 与 Ghostty 的 niri 窗口数量均未增加；
Ghostty 的 `new-tab` 动作存在且可调用。测试打开了一个 Chromium 空白标签页和一个
Ghostty 标签页，没有关闭用户已有窗口或标签页。Mac 经 SSH 的调用仍需远端验收。

## 画面捕获与所谓“读取渲染缓存”的边界

Wayland 应用将绘制后的缓冲区交给合成器；普通客户端没有任意读取其他应用缓冲区的
通用接口。`wattach` 只补足连接会话的环境，不授予缓冲区读取能力。
参见 [Wayland 架构](https://wayland.freedesktop.org/docs/book/Architecture.html)。

合成器支持的捕获协议可以把画面写入共享内存或 DMA-BUF，不必保存 PNG 到磁盘，
但这仍是画面捕获，不是直接获得软件内部语义状态。具体协议支持必须查询当前合成器，
不能因为存在协议就认定本机可用；参见
[Image Copy Capture 协议](https://wayland.app/protocols/ext-image-copy-capture-v1)。

本工作流先验收已经具备的 niri IPC 与内建截图路径。尚未实现无落盘捕获程序或 GPU
缓冲区到远程视觉 agent 的传输，不把它们列为现成能力。需要文字和控件语义时先查应用
接口，需要判断乱码、像素、颜色、遮挡与布局时读取实际捕获的图片。
