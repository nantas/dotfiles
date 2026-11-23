# zsh

`dot_zshrc.tmpl` 用 `{{ if eq .chezmoi.os }}` 分平台。以下几处值得记下：

## 配置不许依赖没装的命令

模板曾经是「现代 CLI 全家桶」：`starship init`、`zoxide init`、`source <(fzf --zsh)` 加
`FZF_*`（预览用 `bat`/`fd`）、`eza` 的 `ls/ll/la/lt`、`BAT_*` 与 `MANPAGER`、`gdircolors`、
`EDITOR=nvim`。这些命令都由 `bootstrap/macos.sh` 的 `PACKAGES` 列表负责装，而**这台 Mac
从来没跑过那一段**，于是每次 `zsh -ic` 都先吐三条 `command not found`，`ls` 之类别名则是在
真正用到时才炸。

现在的判据：**`~/.zshrc` 只引用 macOS / Arch 自带、或当前 shell 已经能解析到的命令**。
可选工具要么整段不写，要么带 `(( $+commands[…] ))` / `[[ -x … ]]` 守卫，绝不让 `eval`
无条件执行——启动报错会淹没真正的问题，而别名指向不存在的二进制属于「配了用不了」。
`PACKAGES` 里的清单仍在，装了也不会自动生效：想用哪一套就自己把初始化加回来，别假设
模板会替你打开。

一个小坑：`fd` 现在能解析到的是 pi 自带的 `~/.pi/agent/bin/fd`，那属于 agent 的私有
PATH，不该被 `~/.zshrc` 当作预览工具的前提。

## darwin 的 PATH 顺序

```sh
export PATH=/opt/homebrew/bin:/opt/homebrew/sbin:$HOME/.local/bin:$PATH
```

Homebrew 必须排在前面。macOS 自带 `/usr/bin/git`，与 brew 的 `git` 是两个版本；`git` 会
把 `git fsmonitor--daemon` 留在后台，daemon 与客户端的版本必须一致，错配时 `git status`
会报 fsmonitor 相关错误。让 Homebrew 目录先命中，客户端、daemon、hook 就都落在同一个构建上。
`$HOME/.local/bin` 在 `/opt/homebrew` 之后，是因为它只放用户装的散装脚本，不该覆盖同类
工具的 brew 版本。

Linux 分支保留原顺序（`~/bin:~/.local/bin:/usr/local/bin`）：那里 `git` 来自发行版包，
没有 Intel Homebrew 与 Apple Silicon Homebrew 共存的 `/usr/local` 分裂问题。

顺带记两个**不在**模板里的东西：

- `TERM=xterm-256color` 不进模板。ghostty 会给自己设置 TERM 并据此协商能力，
  覆盖它会破坏 ghostty 的 shell 集成。旧手写配置里有这行，是 iTerm 时期留下的。
- `COLORTERM=truecolor` 只在 Linux 设，与 `PI_NERD_FONTS` 同一段。

## nvm 有两种装法，只认一种会让它静默消失

macOS 分支必须同时认：

- Homebrew 的 formula 装在 `$(brew --prefix nvm)`；
- `git clone` 装的在 `$NVM_DIR`。

原本只探 brew 那一路。问题在于 **`brew --prefix nvm` 对一个没有安装的 formula 也会
打印一个路径并以 0 退出**——所以它不会报错，只会让 nvm 加载不到。只能靠 `-s` 探
文件本身，不能看命令是否成功。

Arch 的 `extra/nvm` 是第三种位置：`/usr/share/nvm/nvm.sh`（另有 `init-nvm.sh`），既不在
`$NVM_DIR` 也不在 brew 的 prefix。这一路**有意不支持**：Linux 分支只探 `$NVM_DIR`，而
`run_onchange_before_12-dev-runtimes.sh.tmpl` 用 `git clone` 把 nvm 装进 `$NVM_DIR`；理由与
取舍见 [`dev-env.md`](dev-env.md)。

## Apple Silicon 的 MPS 变量

```sh
export PYTORCH_ENABLE_MPS_FALLBACK=1
export PYTORCH_MPS_HIGH_WATERMARK_RATIO=0.0
```

PyTorch 的 MPS 后端没有覆盖全部算子，缺算子时默认直接报错而不是回落到 CPU。高水位
设为 0 是避免它在显存紧张时把张量提前换出。

这两个变量只在 darwin 设：对 Linux 没有意义，而源曾经是从 Linux 生成的，所以它们
一度被漏掉，`apply` 会把它们从这台机器上删掉。

## 模板空白的坑

`{{ if }}` / `{{ end }}` 独占一行时，自身贡献一个换行（充当空行）；而 `-}}` 会吃掉
**所有**连续空白，不是一个换行。用错就会丢空行，所以改动这段后要渲染比对：

```bash
chezmoi execute-template < dot_zshrc.tmpl | diff - ~/.zshrc
```

## 其它

- `PI_NERD_FONTS` / `COLORTERM` 只在 Linux 设。
- macOS 的 `/Applications/*` alias 只在 darwin 设，且都带 `[[ -x … ]]` 守卫：
  `neovide`、`zed` 在本机没装，守卫让它们「装上就生效、没装不报错」。
- `pi()` 是代理自动探测包装：`nc -z -w 1 127.0.0.1 7890` 探到监听才注入
  `http_proxy` 等变量。里面必须写 `command pi "$@"`，否则函数会调用自己，无限递归。
- `OBSIDIAN_VAULT_PATH` 写成 `$HOME/projects/obsidian-mind`，不写绝对家目录路径：
  仓库是 public，禁止出现个人绝对路径。
