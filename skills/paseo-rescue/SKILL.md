---
name: paseo-rescue
description: Use when a Paseo agent stalled mid-task (provider usage limit, quota exhausted, rate limit, crash, error) and you are the agent that must pick up its unfinished work. Triggers on 接管 / 救援 / 限额了 / 继续那个 agent / takeover / rescue / "pick up where that agent left off". Not for delegating your own work to a new agent.
user-invocable: true
---

# Paseo Rescue

把一个中断的 Paseo agent 的上下文捞过来，**由你自己继续干**。

**用户参数：** $ARGUMENTS

## 你是接管者，不是派发者

用户说"接管"时，你很容易匹配到 `paseo-handoff` 的交接语义，然后 `list_profiles` → `list_providers` → `create_agent`，把活派给下一个 agent。**那是反的。**

- Handoff = 我把我的活推给别人
- Rescue = 别人的活断了，我捞过来自己干

你就是接管者。派下一层只是把问题往后推一轮，多烧一个 agent 的钱，用户还得多等。

不要调这些：

- `create_agent` — 你自己干
- `list_profiles` / `list_providers` — 你不需要挑执行者
- `send_agent_prompt` 给源 agent — 它正卡在限额上，发过去只会再撞一次墙

## 1. 定位源 agent

先看用户给了什么，这决定了你能不能自己拍板：

| 用户给的 | 做什么 |
| --- | --- |
| agentId 或 shortId | 直接用，跳过检索 |
| 名字、标题、话题关键词 | 检索，按命中数走下面那张表 |
| 什么都没给 | 检索，**把候选列出来让用户选**，停下等回答 |

用户没给线索时，选择权是他的。哪怕检索只回来一个候选，哪怕工作区里的未跟踪文件和最近改动都指向某一个——那些线索用来给候选排序，不能用来替他拍板。

检索**最多两轮**：

```
第一轮  list_agents(cwd: <当前 workspace>, includeArchived: true, sinceHours: 720, limit: 200)
第二轮  仅当第一轮零命中，同样参数改 sinceHours: 4000
```

两轮之后，按命中数做且只做下面一件事，**不再调 `list_agents`**：

| 命中 | 做什么 |
| --- | --- |
| 1 个 | 用它，继续第 2 步 |
| 多个 | 全部列给用户选，停下等回答 |
| 0 个 | 告诉用户没找到，报出搜过的条件，问他要 agentId |

再搜一次不会有新结果：同一组参数重复调返回同样的东西，把 `sinceHours` 调小只会更少。检索不是收敛过程，两轮就是全部信息。

用户的说法和标题常常字面对不上（他说"token 统计"，标题写的是 `token_cost`）。按语义判断，别因为字面不匹配就重搜。

`status: running` 的从候选里去掉。正在跑的 agent 没有中断，不需要救援——它很可能就是你自己，或者派你来的那个会话。

列候选时最可能的放最前：`requiresAttention: true` 且 `attentionReason` 是 `error` → `updatedAt` 最近 → 未 archived。每条给出 shortId、标题、provider、状态、最后活动时间，让用户一眼能认。

**不要猜。** 猜错了你会带着错误的上下文干完一整轮活。

## 2. 捞上下文

先探规模，再决定怎么读。顺序固定：

```
get_agent_activity(agentId: "<id>", limit: 1)
```

返回的 header 写着 `Showing 1 of N activities`。N 决定下一步：

| N | 怎么读 |
| --- | --- |
| ≤ 300 | `get_agent_activity(agentId)` 不传 limit，全量 |
| > 300 | 传 `limit: 300` 取尾部最近的，并在摘要开头说明你只读了最近 300 条 |

**不传 limit 就是全量**（`limit === 0` 返回全部）。长会话全量能到上百万 token，直接超出你的 context window——真实样本：一个 2496 条的会话。所以先探，不要赌。

拿到 N 之后：

```
get_agent_status(agentId: "<id>")
```

`get_agent_activity` 不向源 agent 的 provider 发请求，**不消耗它的额度**。`closed` 状态的 agent 也读得到——Paseo 会按需把它重新加载出来。

### 降级：直接读原始 transcript

如果 `get_agent_activity` 失败（源 agent 加载不起来），走磁盘：

1. `~/.paseo/agents/<paseo-slug>/<agentId>.json` → 取 `persistence.nativeHandle`，那就是 provider 的 session id
2. Claude 的 transcript：`~/.claude/projects/<claude-slug>/<sessionId>.jsonl`

Paseo 的 agent json 只有元数据，2KB 上下，里面没有对话。对话始终在 provider 自己的 transcript 里，`get_agent_activity` 也是从那儿读的——所以这条路拿到的是同一份源数据，只是没经过 curator 筛选。

两个 slug 规则**不一样**，别混：

| | 规则 | `/Users/me/proj` 的结果 |
| --- | --- | --- |
| `<paseo-slug>` | 去掉根，分隔符换 `-` | `Users-me-proj`（无前导 `-`） |
| `<claude-slug>` | 所有 `/` 换 `-` | `-Users-me-proj`（有前导 `-`） |

原始 transcript 比 curated timeline 全，但噪音大得多。只在 timeline 拿不到、或明显缺了关键细节时才用。

## 3. 清洗

timeline 是模型的原始输出。直接转述会把噪音一起带进新会话。

### 语言串扰

模型会零星地在中文句子里串出日文假名或韩文，通常落在句首语气词、连接词的位置，句子其余部分完全正常：

```
う，总数从联调前的 124 变成 71 了 —— 我只加了 9 条。这不对，得查清楚。
↑ 本该是「嗯」或「等等」
```

判断依据是**上下文主语言**，不是词表：看用户消息用的什么语言，那就是主语言。与主语言不一致的孤立 token，按它在句中的语义翻译回主语言。

**不要碰**：

- 用户的原话
- 代码、路径、标识符、命令、日志输出
- 任务本身就在处理这门语言的内容（i18n 的 ja/ko 文案、多语言测试数据）——那是数据，不是噪音

拿不准是噪音还是内容，保留原文并标注，别猜。

### 其他噪音

剔除：

- 同一个命令的反复失败重试 —— 只留最终结论和失败原因
- 无结果的探索性搜索 —— 除非"这里没有"本身就是结论
- 模型自我推翻的中间结论 —— 只留最后站得住的那个
- 被截断的 tool input —— curated 视图把工具入参截到 400 字符，别把半截 JSON 当事实转述
- 客套话、进度播报、"让我看看…"

保留：

- 用户的明确指令和约束，**逐字保留**
- 失败的方案连同失败原因（防止你重蹈覆辙）
- 已经做出的技术决策和理由
- 改动过的文件和改动内容

## 4. 输出接管摘要，然后停下

读完先给摘要，**等用户确认再动手**。

```markdown
## 接管：<源 agent 标题>（<shortId>，<provider>，停在 <状态>）

**原始任务**
<用他的原话>

**已完成**
- <做了什么；验证过没有>

**未完成**
- <剩下什么>

**试过但失败**
- <方案> — <为什么失败>

**已定决策**
- <决策> — <理由>

**改动的文件**
- `path` — <改了什么>

**中断点**
<断在哪一步>

**我的下一步**
<你打算怎么做>
```

没有内容的小节直接删掉，不要写"无"。

如果源 agent 是**断在一个没人回答的提问上**，把那个问题原样重新抛给用户——那就是中断点，也是你要的下一步指令。

摘要里不要写"根据 timeline 显示…"、"从活动记录看…"这类元叙述，直接陈述事实。

用户确认后再继续干活。

## Red flags

看到自己在做这些，停下：

- 正要 `create_agent` / `list_profiles` / `list_providers` → 你是接管者，自己干
- 正要 `send_agent_prompt` 给源 agent → 它限额了，别捅
- 正要第三次调 `list_agents` → 两轮就是全部信息，去列候选或问用户
- 用户没指定目标，你却自己挑了一个开始读 → 列出候选，等他选
- 候选里有 `status: running` 的 → 去掉，那不是待救援的会话
- 直接全量读 activity，没先 `limit: 1` 探规模 → 可能一次撑爆 context
- 还没读完 activity 就开始改文件 → 先读完
- 把 `う`、`ね`、`니다` 这类残留原样抄进摘要 → 按主语言翻译
- 摘要里出现半截 JSON → 那是被截断的 tool input，不是事实
- 摘要写完直接开干 → 先等用户确认
