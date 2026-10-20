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
wattach grim -           # 画面写到 stdout；SSH 或管道直接拿到，不另传文件
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
wattach grim - >/tmp/a.png # 真的连上了才算数；远程不要再把文件拷回
niri msg outputs
wdetach                  # XDG_SESSION_TYPE 应回到 tty
wdetach                  # 再跑一次应报错并返回 1
```

## 从 Mac 执行

一次 SSH 跑一条 `wattach` 命令。结果就是这次 SSH 的 stdout。不要在 Linux 落盘，再 `scp` 或再开一次连接去读文件。

### 配置

- `sshd` 在跑，局域网能连上。登录用户就是跑 niri 的那个用户，默认 shell 是 zsh，已部署本仓库的 `.zshrc`。
- 已登录 niri。停在 GDM 时没有可附加的桌面。锁屏不等于 SSH 停止，但这时看到的是锁层，不是锁后面的窗口。
- 公钥追加到该用户的 `~/.ssh/authorized_keys`，保留已有条目。不要传输私钥，不要把账号、局域网地址或主机指纹写进仓库。无人值守执行前，公钥认证和主机指纹都要已经核对过。首次连接在 Linux 上对照：

```sh
ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
```

- `grim` 在 `PATH` 里，而且重启后还在。看画面只用它。不要用 niri 的 `screenshot-screen` / `screenshot-window`，也不要假设 `xdg-desktop-portal-wlr` 会把 grim 装上。只放在 `/tmp` 的解包会在重启后消失。

### 调用

非交互 SSH 不加载 `.zshrc`，要用交互 zsh 才有 `wattach`。占位不要写回公共配置：

```sh
remote='LINUX_USER@LINUX_LAN_IP'
ssh "$remote" "zsh -ic 'wattach niri msg -j windows'"
ssh "$remote" "zsh -ic 'wattach grim -'" > frame.png
```

每次 SSH 都是新 shell，都要写 `wattach COMMAND`。命令形式在子 shell 里丢掉附加横幅，stdout 只有命令自己的输出。`zsh -ic` 可能在 stderr 报 `can't change option: zle`；不要为此加 PTY，控制字符会混进要解析的输出。

`grim -` 写出的是 PNG。窗口标题、应用标识和窗口 ID 走 `niri msg`，那份 JSON 里没有画面。断线后还要留着的进程用 `wattach -d`。不要把锁屏、关闭窗口或重启桌面当作第一项控制测试。坐标点击和键盘注入不是 `wattach` 的功能。

niri 重启后重新执行 `wattach`，不要复用之前保存的 `NIRI_SOCKET`。挂起期间 SSH 不可用。

## 合成器停掉之后

niri 卡死时不要从 SSH 执行 `systemctl --user restart niri.service`。这个单元是 `Type=notify`，`niri-session` 用 `systemctl --user --wait start` 守着它。进程不响应退出时，强杀会结束这次图形会话，座位回到 GDM。用户服务里再拉起的 niri 打不开已经被占用的 TTY。恢复桌面要在座位上重新登录。

重新登录后，旧会话里的进程还在，环境里仍是死掉的 `NIRI_SOCKET`。它们不会出现在新桌面的窗口列表里，也会继续占内存。要在新会话里重新打开同一个程序，先结束旧进程。名字里带旧 pid 的 socket 文件可以删掉。`wattach` 会跳过已经死掉的 pid，但别让别的程序再连上去。

## 持久终端

要留在远端终端里操作，用 tmux，不要用一次性的 `ssh host '命令'`。会话名固定为 `main`。

```sh
ssh -t "$remote" 'tmux new -A -s main'
```

`-t` 必须有，否则 tmux 没有终端。断开 SSH 后会话还在。再执行同一条命令就接回去。会话里按 `Ctrl-b` 再按 `d` 只是脱离，不会停掉里面的进程。整机重启会清掉 tmux 服务器；`~/.zshenv` 和 tmux 本体还在，用同一条命令重建会话。

这条命令是非交互 SSH，不读 `.zshrc`。`PATH` 和 `LANG` 必须写在那台机器的 `~/.zshenv`。这个文件不进仓库：macOS 上已有各自的 `~/.zshenv`，用源覆盖会清掉机器私有的环境。

- `PATH` 要包含 tmux 所在的目录。只写在 `.zshrc` 里时，这条命令会报 `command not found`。
- `LANG` 必须是 UTF-8，本仓库用 `en_US.UTF-8`。客户端以 POSIX locale 启动时，tmux 关掉 UTF-8 输出，TUI 里的中文会被画成 `_`。窗格内容本身仍是 UTF-8。改完 `LANG` 后要脱离再连，已经附上的客户端不会自己换 locale。

tmux 可以是系统包，也可以是 `~/.local/bin/tmux`。后者要同时带上 `~/.local/lib/libutempter.so.0`，并且这些文件必须在根文件系统上，不能放 `/tmp`。`~/.local/bin` 在 `PATH` 里排在 `/usr/bin` 前面，所以即使以后装了系统包，仍然先用这个包装。

进入会话后的 shell 仍然看不到图形桌面。要操作 niri 时先跑 `wattach`。

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
Ghostty 标签页，没有关闭用户已有窗口或标签页。

## 画面

`wattach` 只把 niri 会话的环境补进当前命令，不读取合成器的内部缓存。要像素，就在同一次 SSH 里执行 `wattach grim -`。niri 26.04 提供 `zwlr_screencopy_manager_v1`，grim 走这条协议；没有 `ext_image_copy_capture_manager_v1`。

niri IPC 的截图动作只接受绝对路径，并把图放进剪贴板。它的回复里没有像素，不能当作画面返回调用方的方法。文字和控件语义先查应用自己的接口。
