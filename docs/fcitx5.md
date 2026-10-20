# fcitx5

## GTK 输入法模块：全局使用 fcitx immodule

公共基线由 `dot_config/environment.d/fcitx5.conf.tmpl` 设置 `GTK_IM_MODULE=fcitx`，让 GTK
应用使用 fcitx5 的 GTK immodule（DBus frontend）。公共基线不设置 `QT_IM_MODULE`。

本机 `[data] localDesktop = true` 是例外：environment.d 不全局设置输入法变量，
由 niri 模板给桌面应用设置 GTK/QT/XMODIFIERS/INPUT_METHOD，以保留备用 GNOME 的 IBus。
具体部署见 local-system-handoff-2026-09-29.md。

### Ghostty 候选窗位置（2026-09-29）

在本机 Ghostty 1.3.1、fcitx5 5.1.23、niri 26.04 下，未设置 `GTK_IM_MODULE` 时，Ghostty
使用 `wayland_v2` 原生 text-input。pi 和普通 shell 的底部输入行都会遇到候选窗遮挡：Niri
在窗口内没有足够空间把候选窗放到光标下方时，会将它翻到上方。

用 `env GTK_IM_MODULE=fcitx ghostty` 单独启动 Ghostty 后，fcitx5 DebugInfo 确认焦点 IC 改为
`frontend:dbus`（`program` 为空）；候选窗正常出现，且用户实测不再遮住输入行。因此选择全局
设置 GTK immodule。该设置是否影响其他 GTK 应用，仍应以本机逐个实测为准。

### 历史记录：候选窗曾经消失（原因未确定）

此前在 Ghostty 中确实记录过一次不同症状：拼音 preedit 正常，但候选窗不出现；当时观察到
`GTK_IM_MODULE=fcitx` 对应 `frontend:dbus`、`program` 为空，而 unset 后走 `wayland_v2`、候选窗可见。
当时还观察到 fcitx5 从 5.1.22 升至 5.1.23 后出现该现象，并怀疑 DBus 路径上的光标矩形处理，
但没有证实根因。

本次在相同版本上重新单独启动 Ghostty，`GTK_IM_MODULE=fcitx` 的 DBus 路径候选窗正常，旧现象
未能复现。因此保留这条历史记录，但不再把它归因于 fcitx5 5.1.23 回归；**候选窗偶发消失的原因
仍未查明**，也不能据此断言该变量在所有情况下都稳定。

`QT_IM_MODULE` 不随之开启：本次验证只覆盖 GTK/Ghostty；Qt Wayland 应用继续使用原生
text-input。


## 字体升级后候选字异常（2026-09-30）

本机安装工具时同步升级 `ttf-sarasa-gothic`，仍运行的 fcitx5 进程（PID 3932）没有重启。
其 `/proc/3932/maps` 中的 Sarasa-Regular.ttc 和 fontconfig cache 已显示 `(deleted)`，
而候选字体仍是 `Sarasa Mono SC 14`。新启动的 fc-match 能正确匹配新字体，旧进程却仍持有
旧映射；这支持字体升级后进程内字体状态失效的判断，尚未用候选窗截图确认具体渲染机制。

先执行 `fc-cache -f`，再通过 DBus `org.fcitx.Fcitx.Controller1.Restart` 重启。
新进程 PID 548505，旧字体的 deleted 映射消失；未修改候选字体、词库、用户词频或 Rime 数据。
用户需要重新输入拼音确认显示。仅 `fcitx5-remote -r` 重载配置不一定清除进程内字体对象，
字体包升级后优先真正重启输入法；同样受影响的其他应用按需重开，不强制退出整个桌面。

### 后续验证与候选字体替换（2026-10-20）

刷新字体缓存并再次完整重启 Fcitx5 后，用户截图仍显示候选字异常，连候选序号也变成
`t u v w x`。旧 Ghostty 进程确实持有已删除的 Sarasa 字体映射，且加载了 GTK4 的
Fcitx5 模块；但独立启动新 Ghostty 后，用户确认候选仍乱码，因此旧进程残留不足以解释问题。

使用 Rime 用户目录的临时副本，通过 librime API 直接输入 `nihao`，得到正常的
`你好、👋、拟好、你、尼`。独立 Pango 测试也能正常绘制 `Sarasa Mono SC 14`，
字体包完整性检查通过。这些结果将问题缩小到候选窗使用 Sarasa 的显示路径，
不能据此断言字体文件损坏、词库损坏，或已确定具体的字体索引／缓存故障。

把 Classic UI 的 `Font` 和 `MenuFont` 换成 `Noto Serif CJK SC 14`，重载配置，
再在独立 Ghostty 进程中测试，用户确认汉字和序号均恢复正常。因此保留这套已验证字体，
不改 Rime 词库、用户词频、终端字体或全局 fontconfig。

选择 Noto Serif 是为了避开现有 fontconfig 将 `Noto Sans CJK SC` 重定向到 Sarasa 的规则。
GTK 输入法模块可能由应用进程绘制候选窗，参见
[Fcitx 官方 FAQ](https://fcitx-im.org/wiki/FAQ#Candidate_window_is_blinking_under_wayland_with_Fcitx_5)；
因此重启输入法不能替代对独立新应用进程的验证。

### 持久性核查

本次替换同时写入 chezmoi 源与 `~/.config/fcitx5/conf/classicui.conf`，两份文件的
SHA-256 一致；不是只通过 DBus 修改当前进程。字体由已安装的 `noto-fonts-cjk` 提供。
本机 niri 登录时直接启动 `fcitx5`，未发现仓库、用户 autostart/systemd 或系统睡眠钩子
中存在将候选字体改回 Sarasa 的配置。`run_onchange_after_40-fcitx5.sh.tmpl` 只处理
Rime 数据与重启，不覆盖 Classic UI 字体。

完整重启 Fcitx5 后，进程 PID 从 587111 变为 595997；通过 DBus `GetConfig` 确认
新进程的 `Font` 与 `MenuFont` 均为 `Noto Serif CJK SC 14`，磁盘文件未变，定向
chezmoi apply 无输出、diff 为空。由此确认字体配置能够跨输入法进程重启保存。
尚未执行整机重启或挂起／唤醒，不能把配置持久性检查等同于这两种场景的实际显示验证。

### 复用修复与重启／唤醒后的验收

候选字体的唯一配置源为 `dot_config/fcitx5/conf/classicui.conf`，目标是
`~/.config/fcitx5/conf/classicui.conf`，已验证的值为：

```ini
Font=Noto Serif CJK SC 14
MenuFont=Noto Serif CJK SC 14
```

在已接入且完成比对的机器上，只部署这一项配置即可；不要为输入法修复直接应用其他
尚未核对的本机差异：

```sh
fc-match -f '%{family}\n%{file}\n' 'Noto Serif CJK SC:size=14'
chezmoi diff --include=files ~/.config/fcitx5/conf/classicui.conf
# Review the diff before applying.
chezmoi apply --include=files -v ~/.config/fcitx5/conf/classicui.conf
fcitx5-remote -r
chezmoi apply --include=files -v ~/.config/fcitx5/conf/classicui.conf
chezmoi diff --include=files ~/.config/fcitx5/conf/classicui.conf
```

字体应匹配到 Noto Serif CJK SC；最后一次 apply 和 diff 应为空。若文件被手工改过，
chezmoi 可能要求确认覆盖，应先检查差异，不直接强制覆盖。

需要排除旧进程状态时，可完整重启 Fcitx5，再读取实际生效的配置：

```sh
pgrep -a fcitx5
busctl --user call org.fcitx.Fcitx5 /controller org.fcitx.Fcitx.Controller1 Restart
# Run these after the replacement process has started.
pgrep -a fcitx5
busctl --user call org.fcitx.Fcitx5 /controller org.fcitx.Fcitx.Controller1 \
  GetConfig s fcitx://config/addon/classicui
```

确认 PID 已改变，返回值中的 `Font` 和 `MenuFont` 均为 `Noto Serif CJK SC 14`。
输入法重启可能切回英文，需要重新切到中文。用独立新进程验证候选显示，避免普通新窗口
复用旧 Ghostty 进程：

```sh
env GTK_IM_MODULE=fcitx ghostty --gtk-single-instance=false
```

输入 `nihao`，检查候选汉字、序号是否正常。整机重启后，以及挂起再唤醒后，都应分别
复验一次，并记录结果；只有进程重启检查通过时，不应标记这两项已经验收。

若唤醒后复发，先保留现场截图，并在重载／重启前记录以下信息：

```sh
pgrep -a fcitx5
pgrep -a ghostty
chezmoi diff --include=files ~/.config/fcitx5/conf/classicui.conf
busctl --user call org.fcitx.Fcitx5 /controller org.fcitx.Fcitx.Controller1 \
  GetConfig s fcitx://config/addon/classicui
busctl --user call org.fcitx.Fcitx5 /controller org.fcitx.Fcitx.Controller1 DebugInfo
journalctl --user -b --since '10 minutes ago' --no-pager
```

区分磁盘配置被覆盖、进程生效值变化，以及配置正常但绘制异常三种情况，再进行字体
对照与独立应用测试。不要先删除 Rime 用户目录、词频数据库或重装词库。
