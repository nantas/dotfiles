# Magpie 路由组：同档次模型并成一个 id

同一档次但来自不同供应商的模型，打包成 agent 眼里**一个**模型（`group/<id>`）。其中某个撞上
5 小时 / 7 天限额时，网关自动换同组下一个成员重发，agent 看不到错误。

本文记录这件事在 magpie 里怎么发生、本机三个组的定义、哪些参数不设会踩坑。

---

## 目标

- 一个 agent 只认一个模型 id，背后是一串同档次成员；
- 成员撞限额时自动往下走，而不是把 429 抛给 agent；
- 尽量多的独立限额桶：每个订阅 / 计划一份窗口，跨供应商才有更多桶。

---

## 为什么是 routing group

magpie 自带这个功能，不需要插件或中间层。相关命令只在 `magpie group` 一族，配置落在
`~/.config/magpie/providers.json`，不在 chezmoi 管辖内（原因见 [`magpie.md`](magpie.md)）。

**被否决的做法**：

- 每条会话手工换模型 —— 限额总是在坏时机撞上；
- `magpie provider fallback <id> <model>` —— 只覆盖「这个 provider 死了」，不覆盖
  「provider 还活着但这个模型的窗口用完了」，也不跨 provider 编排优先级；
- 外部代理（New API 之类）做多渠道轮询 —— 多一层进程、看不到 magpie 读到的配额窗口，
  且 `stays` 的会话粘性要重做一遍。

---

## 机制要点

以下均为 `magpie 0.1.4503` 的实际行为（`magpie group help` 与上游 `docs/reference.md`）。

**failover 触发点**：只要回复还没开始流式输出，任何成员失败都会**立刻**换组内下一个成员重发
同一个请求 —— 429 命中配额、5xx、连不上、key 欠费都算。已经流出去的回复不重试，agent 拿到错误。

**限额怎么被感知**（两条腿并用）：

- 主动读：`magpie quota` 能读到每个订阅 / 计划的窗口用量与重置时间；
- 被动学：429 响应里带的 reset 时间（Claude 的 `usage limit reached|…`、ChatGPT 的
  `resets_at`）让该成员 **rest 到窗口重启**，期间不再被尝试；窗口一重启，magpie 重读一次配额
  确认后立刻复活，不等供应商自己说的时刻。

**`routing=order` 的语义**：

| 成员状态 | 行为 |
| --- | --- |
| < 90% | 新会话一律先去它 |
| ≥ 90% | order 下仍是它（order 不看余量，只看顺序） |
| 已经在它上面的会话 | 一直留着，直到该窗口 100%（这是 `order` 的阈值；`smart` 是 98%） |
| 100% / 撞 429 | 移出候选，rest 到窗口重启 |

`order` 是「用到干为止再换下一个」，可预测性最高，代价是快重置的窗口不会主动去用掉。
需要「先烧快过期的窗口」时改用 `routing=smart`（默认值）。

**`stays`**：默认 `auto` —— 会话粘在回答过它的成员上，直到该供应商的 prompt cache 不再值钱。
这是想要的：只在必要时才切。`stays=strict` 永不搬，等于关掉 failover，不要用。
`stays=session` 在 `order` 下无意义（新会话照样去第一个成员），那是 `smart` / `usage` /
`expiring` 才有的分散行为。

---

## 本机的三个组

三个组都用 `routing=order stays=auto`，链的顺序按「换了之后行为跳跃最小」排。

| 组 | 用途 | 上下文 | effort | 成员（按优先顺序） |
| --- | --- | --- | --- | --- |
| `group/grunt` | 主要用于 subagent，一般不直接和人对话 | 500k | low | `zhipuai-coding-plan/glm-5.3` → `grok/grok-4.7@500k` → `codex/gpt-6.1-terra` |
| `group/workhorse` | 直接和人对话，执行一般任务 | 500k | medium | `workbuddy/deepseek-v4.1-flash` → `grok/grok-4.7@500k` → `codex/gpt-6.2-sol` → `deepseek/deepseek-flash` → `agy-cli/gemini-4-hydrogen` |
| `group/leader` | 高智能：与人对谈、规划、设计，指挥调度其他 agent | 1M | medium | `srv/claude/claude-opus-5-5` → `agy-cli/gemini-4-neon` → `kimi/k3.1-pro` |

每个成员背后的限额桶各不相同，这是分组的实际收益：

| 成员 | 桶 |
| --- | --- |
| `zhipuai-coding-plan/glm-5.3` | GLM 5 小时 + 月 |
| `codex/*` | ChatGPT 7 天（+ 积分） |
| `grok/grok-4.7@500k` | Grok 7 天 |
| `workbuddy/*` | WorkBuddy 月积分 |
| `deepseek/*` | 余额（按量付费，无窗口） |
| `agy-cli/*` | Antigravity 的 Gemini 5h + 7d，或 Claude&GPT 5h + 7d |
| `srv/claude/*` | Claude 订阅 5 小时 + 7 天 |
| `kimi/k3.1-pro` | Kimi 5 小时 + 月 |

这三个组由仓库播种，不手工建：

- 声明：`scripts/magpie/groups.json`（成员顺序、`note`、`routing`、`context`、`levels`、`autoGroups`）
- 执行：`run_onchange_after_55-magpie.sh` / `.ps1` 里的 `seed_groups`，改动 `groups.json`
  会让脚本在下一次 `chezmoi apply` 时重跑

播种等价于下面这组命令（跑一次即把本机改回声明里的样子）：

```sh
magpie group add grunt models=zhipuai-coding-plan/glm-5.3,grok/grok-4.7@500k,codex/gpt-6.1-terra \
  routing=order stays=auto context=500k levels=low
magpie group set grunt note='主要用于 subagent，一般不直接和人对话'
magpie group add workhorse \
  models=workbuddy/deepseek-v4.1-flash,grok/grok-4.7@500k,codex/gpt-6.2-sol,deepseek/deepseek-flash,agy-cli/gemini-4-hydrogen \
  routing=order stays=auto context=500k levels=medium
magpie group set workhorse note='直接和人对话，执行一般任务'
magpie group add leader models=srv/claude/claude-opus-5-5,agy-cli/gemini-4-neon,kimi/k3.1-pro \
  routing=order stays=auto context=1m levels=medium
magpie group set leader note='高智能，主要和人对话，处理复杂问题，规划、设计、指挥调度其他 agent'
magpie group auto off
```

**成员缺失时降级，不是失败**：播种时逐个核对 `magpie models`，本机不提供的成员被**丢掉并打印
一行警告**，组照样按剩下的成员建出来。所以一份 `groups.json` 能落在登录状态、暴露模型各不
相同的机器上；代价是某台机器的链可能比声明的短，看 `magpie group <id>` 和 apply 的警告确认。

### 与需求原话的一处偏差

`srv/claude-opus-5.5` → **`srv/claude/claude-opus-5-5`**。`srv` 是远端 magpie，模型 id 自带
厂商段（`claude/claude-opus-5-5`），完整写法是三段。

`deepseek` 那一格用的是 `deepseek/deepseek-flash`（DeepSeek V4.1 Flash，与
`workbuddy/deepseek-v4.1-flash` 同代），而不是 mac 上另外暴露的 `deepseek/deepseek-v4-flash`
（V4）。三台机器里只有 mac 有 V4，另两台只有 V4.1，用 V4.1 才能让三台机器的链等长。

### 与需求原话的两处偏差

- `srv/claude-opus-5.5` → **`srv/claude/claude-opus-5-5`**。`srv` 是远端 magpie，模型 id
  自带厂商段（`claude/claude-opus-5-5`），完整写法是三段。
- `deepseek/deepseek-flash` → **`deepseek/deepseek-v4-flash`**。DeepSeek provider 当前暴露的是
  `deepseek-v4-flash`；`deepseek-flash` 在 vendor 列表里但未暴露（`magpie provider models
  deepseek +deepseek-flash` 可补）。两者同一档，选了已暴露的那个，避免再多一个需要维护的
  exposure 开关。

### `levels` 同时就是 effort 的约束

组把 `levels` 设成单一档位（`low` / `medium`），agent 在组上能选的推理档就只有那一档，
请求出去的 effort 也就是它。成员不提供该档时收到「最近的档」—— 实际影响三处：

| 成员 | 自身档位 | 组要 `medium` 时收到 |
| --- | --- | --- |
| `kimi/k3.1-pro` | none/low/high/max | medium 不在列，收到 low 或 high 之一 |
| `workbuddy/deepseek-v4.1-flash` | none/low/high/max | 同上 |
| `deepseek/deepseek-v4-flash` | none/low/high/max | 同上 |

`grunt` 的 `levels=low` 三个成员都有 low，不受影响；`workhorse` 的 `codex/*` 与
`grok/*` 有 medium，也不受影响。

想让某个成员在组里用另一个档位，不要改组，用规则：
`magpie group rule add leader use=kimi/k3.1-pro use-effort=high`。

---

## 不设会踩坑的参数

**`context`** —— 组对外宣告的上下文窗口，**默认取最小成员**。三组恰好都把成员的最小值显式写了
（grunt / workhorse 有 500k 成员，leader 全是 1M），所以现值等于默认值；写出来是为了以后加成员
时不会莫名其妙地让整组跟着缩水。宣告得比某个成员大时，magpie 会自己对超长请求跳过装不下的成员
—— 在 `order` 链下正好落到下一个能装的。

**`levels`** —— 见上一节。

**`stays`** —— 必须留在 `auto`。

**`compact`** —— Codex / Claude Code 在哪压缩对话。默认留给各模型自己；只有压缩点明显错位时才设。

**`listed`** —— 默认 `yes`，成员仍可单独被 agent 选中。想减少模型选择器的噪声才设 `listed=no`
（成员从 `magpie models` 和每个 agent 的列表里消失，只能通过组访问）。

---

## 风险与缓解

| 风险 | 说明 | 缓解 |
| --- | --- | --- |
| 丢 prompt cache | 跨成员切换后整段前缀要重算，成本和首字延迟都跳 | `stays=auto` 只在必要时才切；链上把同家模型放前面 |
| 行为断层 | 组内成员换了，工具调用风格、命名习惯、解释详略都变 | `order` 把异构模型排在后面；重要任务手工锁单模型，例如 `magpie claude srv/claude/claude-opus-5-5` |
| 非配额失败也切 | 5xx / 连接抖动同样落到下一个成员 | 给不稳的中转设 `magpie provider rest-cap off`（不 rest，下一轮就回去） |
| 流式开始后失败不重试 | 已输出半个回复就报错给 agent | 无法规避；配额通常在请求开头就拒，命中概率低 |
| 推理块跨厂商回放 | 前一个模型的 thinking 内容能否被翻译给下一个厂商，上游文档未明说 | 待实测：切一次后看会话是否被拒 400 |
| `order` 不吃快过期窗口 | 第一个桶会被吃到 100% 才换 | 这就是 `order` 的语义；想榨窗口改 `routing=smart` |

---

## 接线与验证

组已经存在，接线是让 agent 使用它：

```sh
magpie claude group/leader        # Claude Code：对谈与规划
magpie codex  group/workhorse     # Codex：按计划执行
magpie pi     group/leader        # Pi：默认
magpie visible claude leader       # 让 Claude Code 的选择器只显示这个组，不列出每个成员
```

agent 侧的 effort 要与组的 `levels` 对齐，否则 agent 可能发出组里没有的档位：

```sh
magpie codex effort medium
magpie claude effort medium
```

验证：

```sh
magpie group grunt                # 成员、顺序、routing、stays、levels
magpie groups                     # 三个组 + 四个自动组
magpie quota                      # 每个桶还剩多少、何时重置
curl -s localhost:3425/v1/models -H 'Authorization: Bearer magpie' \
  | python3 -c 'import json,sys;[print(m["id"],m.get("context_length"),m.get("reasoning_levels")) for m in json.load(sys.stdin)["data"] if m["id"].startswith("group/")]'
```

期望输出：

```
group/grunt | ctx 500000 | levels ['low']
group/workhorse | ctx 500000 | levels ['medium']
group/leader | ctx 1000000 | levels ['medium']
```

`magpie group <id>` 只打印 `levels`，不打印 `context` —— 后者要到 `/v1/models` 里看
（上面那条命令），或直接看 `~/.config/magpie/providers.json` 的 `groups` 条目。

运行期看谁答的：响应头 `X-Magpie-Provider` / `X-Magpie-Model`，或 app 的 Routing 页面
（每次请求的排序理由都写在里面）。

回滚：

```sh
magpie group rm grunt workhorse leader
```

---

## 自动组已关闭

magpie 默认会为「两个以上供应商都提供的同名模型」自动建组（`auto/<model>`，永不落盘）。
`scripts/magpie/groups.json` 里 `"autoGroups": false` 表达的就是关掉它 —— 播种时跑一次
`magpie group auto off`，`auto/glm-5.3`、`auto/glm-5.3-flash`、`auto/kimi-k3.1-pro`、
`auto/glm-5.2` 不再列出、不再被服务，以后新出现的同名模型也不会再生成。改成 `true`
（或删掉这一行）就恢复。

**副作用**：Claude Code 的 `auto_mode` 原本挂在 `auto/glm-5.3-flash` 上，开关关闭时被 magpie
改写成单供应商的 `zhipuai-coding-plan/glm-5.3-flash` —— 单点，不再有跨渠道 failover。要把
failover 拿回来，就把它指向自己的组：

```sh
magpie claude auto_mode group/grunt      # 低档、cheap，符合 auto mode 的定位
```

`auto_mode` 属于 agent 配置，不在这份声明里。

逐个 `magpie group rm auto/glm-5.2` 只隐藏那一个，以后依然会生成新组，所以选择关开关。

---

## 固化：仓库管什么、不管什么

**管**（无凭据、且是手调出来的那部分）：

- 三个路由组：成员与顺序、`note`、`routing`、`stays`、`context`、`levels`
- 自动组开关（`autoGroups`）

**不管**：

- `~/.config/magpie/providers.json` 整体 —— 里面有 API key、OAuth 登录态，且 magpie 运行时
  持续改写。仓库只播种「缺失的 Provider ID」，从不覆盖已有条目。见 [`magpie.md`](magpie.md)。
- `settings.json` 的偏好项 —— 由 `magpie settings set` 在首次 seed 时写入一次。
- agent 侧的模型接线与 `auto_mode` —— 属于各 agent 自己的配置文件。

**触发方式**：`seed_groups` 挂在 `run_onchange_after_55-magpie` 里，脚本头部记录了
`scripts/magpie/groups.json` 的 sha256，所以改声明 → 下一次 `chezmoi apply` 重跑一次；
没改就不动。重跑会**把组改回声明的样子**——这正是固化，本机手工调过的顺序会被覆盖。

**Windows 的一个坑**：`magpie.exe` 把输出写到控制台而不是管道，PowerShell 的 `& $exe models`
捕获到 0 行（在 `cmd` 里跑却正常）。`Seed-Groups` 因此走 `cmd /c "… models > 临时文件"` 再回读。
同一台机器上 `autostart` 的状态判断也受此影响 —— 读不到就当没开，每次重新 `autostart on`，
无害但会重复执行。

改分组的正确路径：改 `scripts/magpie/groups.json` → 提交 → 各机器 `chezmoi update`。
想先在一台机器上试，直接跑播种那几条 `magpie group ...` 命令，满意后再回写声明。
