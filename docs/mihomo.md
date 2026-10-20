# mihomo

仓库只钉客户端接线和两项服务策略。订阅、凭据、当前节点、`allow-lan` 不进仓库，也不把某次查到的值写进文档。下面只写怎么在本机重读。

设计取舍（为什么用 drop-in、为什么不把配置收进源）在 [`design.md`](design.md) 的「mihomo：哪些能管，哪些不能」。

## 仓库管什么

- 终端：`dot_zshrc.tmpl` 的 `pon` / `poff` / `pstat`。Linux 交互式 zsh 发现 `mihomo` 命令时静默执行 `pon`。默认混合端口 `127.0.0.1:7890`。
- 浏览器：`dot_config/chrome-flags.conf` 与 `chromium-flags.conf` 的 `--proxy-server` 和 `--proxy-bypass-list`。没跑代理的机器要删这两行，否则浏览器会整体断网。
- 新装机时 `bootstrap/arch.sh` 还会做两件事。**已经接入的机器不要假设这两步跑过**，用下一节的检查核对：
  - drop-in 把外部控制钉在 `127.0.0.1:9090`，面板目录钉在 `ui/xd`
  - `scripts/mihomo-overlay.sh` 只把 `mixed-port` 钉回 `7890`（mihomo 没有对应的命令行 flag）

不钉、也不记录现值的：`allow-lan`、`bind-address`、`mode`、`dns`、TUN、`secret`、策略组的当前选择。它们在订阅或 mihomo 的运行时缓存里。手工改 `config.yaml` 活不过下一次整文件刷新。

`~/.gitconfig` 的 `[http]` 段和 `~/.ssh/config` 的 `ProxyCommand` 是个人层，同样不进仓库。

## 不要写进文档或提交

- `/etc/mihomo/config.yaml`，以及任何订阅链接、节点名、套餐日期、服务器地址、密码
- 控制 API 的 secret，和读它的那条命令的输出
- `/proxies` 的原始 JSON。它能露出当前选择；整份响应不要贴进仓库、聊天或交接记录
- 某次测到的 `allow-lan`、`bind-address`、选中节点。这些会变，写下来就会过时

## 怎么查

服务和监听，不涉及 secret：

```sh
systemctl is-active mihomo
systemctl is-enabled mihomo
tr '\0' '\n' < /proc/$(pidof mihomo)/cmdline
ss -lnt | awk 'NR==1 || /:7890|:9090/'
```

这台机器有没有装上仓库里的那两步。drop-in 与 `bootstrap/arch.sh` 的 `install_mihomo_overlay` 对照，不要把 unit 正文抄进文档：

```sh
test -x /usr/local/bin/mihomo-overlay && echo overlay-installed || echo overlay-absent
pacman -Q mihomo-bin clash-geoip
systemctl cat mihomo
```

运行时配置和当前节点走本机 API。secret 不在命令行上，也不在仓库里。API 返回 `Unauthorized` 时，到 root-only 的 `/etc/mihomo/config.yaml` 里找 `secret:`，或到本机导入状态目录（若用过那套导入，常见位置是 `~/.local/state/mihomo-setup/`，权限应为 0700）。读出来之后不要回显、不要提交。

```sh
# SECRET 来自上一段，不要 echo
curl -fsS -H "Authorization: Bearer ${SECRET}" http://127.0.0.1:9090/configs \
  | jq '{mode, "mixed-port", "allow-lan", "bind-address", tun: .tun.enable}'
curl -fsS -H "Authorization: Bearer ${SECRET}" http://127.0.0.1:9090/proxies \
  | jq '[.proxies[] | select(.type=="Selector") | {name, now}]'
```

没有 `jq` 时不要改用会打印全文的格式化。下面只取同样的字段：

```sh
python3 - "$SECRET" <<'PY'
import json, sys, urllib.request
secret = sys.argv[1]
def get(path):
    req = urllib.request.Request(
        "http://127.0.0.1:9090" + path,
        headers={"Authorization": "Bearer " + secret},
    )
    with urllib.request.urlopen(req, timeout=8) as resp:
        return json.load(resp)
cfg = get("/configs")
print({k: cfg.get(k) for k in ("mode", "mixed-port", "allow-lan", "bind-address")})
print("tun.enable:", (cfg.get("tun") or {}).get("enable"))
for name, info in get("/proxies").get("proxies", {}).items():
    if info.get("type") == "Selector":
        print(f"{name} -> {info.get('now')}")
PY
```

`mode` 为 `rule` 时，`GLOBAL` 的 `now` 不是普通流量的出口；看各策略组自己的 `now`。

切换发生在控制 API 或面板，结果留在 mihomo 自己的缓存里，不在本仓库。改完用上面的查询确认，不要把新节点名写回文档。

客户端和未纳管的个人层：

```sh
pstat
rg -n 'proxy-server|proxy-bypass-list' ~/.config/chrome-flags.conf ~/.config/chromium-flags.conf
git config --global --get-regexp 'proxy'
rg -n 'ProxyCommand|7890' ~/.ssh/config
```

面板在 drop-in 带了 `-ext-ui` 且目录已存在时，是 `http://127.0.0.1:9090/ui/`。登录用的 secret 同样不记录。
