# 上下文、记忆和压缩：长任务里哪些信息该留下

我做上下文这一层，重点不是 prompt 写得漂不漂亮，而是长任务里的信息寿命分层。当前请求、稳定前缀、工作记忆、相关记忆、历史转录、技能说明，这几类信息不能混成一段字符串。

![上下文与记忆分层](assets/03-context-memory-compact.png)

## ContextManager 是 prompt 的组装器

`pico/core/context_manager.py` 把 prompt 明确拆成六段：

```text
prefix -> memory -> skills -> relevant_memory -> history -> current_request
```

默认总预算 60000 字符，每段有自己的预算和 floor。超预算时按这个顺序收缩：

```text
relevant_memory -> skills -> history -> memory -> prefix
```

`current_request` 不裁。这条判断很关键：最新用户请求一旦丢语义，留再多旧上下文都没意义。

这一层还有一个容易被低估的点。`build()` 返回的不只是 prompt，还有 metadata。每段的原始长度、渲染长度、预算收缩、selected notes、context usage 都会进 trace 和 report。也就是说，我不只知道发出去的 prompt 是什么，还能解释它为什么长成这样。

## Prefix 是稳定运行时资产

`runtime.py` 里的 `PromptPrefix` 带这几个字段：

- `text`
- `hash`
- `workspace_fingerprint`
- `tool_signature`
- `built_at`

我已经不每轮临时拼 system prompt 了。系统在跟踪稳定前缀什么时候可以复用，什么时候因为工具签名、工作区或模式变化需要刷新。

Claude Code 这一层更完整。它把 system prompt parts、tools、model、beta headers、effort、cache strategy 都当成 prompt cache 的相关因素。我目前只做到 prefix/hash/fingerprint/tool signature，还没有更细的 section registry 和 cache break detection。

## Working memory 是当前任务的工作面

`pico/features/memory.py` 里的 `LayeredMemory` 保存一份很小的工作集：

- `task_summary`
- `recent_files`
- `file_summaries`
- `episodic_notes`
- `durable_topics`

它不是知识库，作用是让下一轮少做重复劳动。比如刚读过的文件会留下短摘要和 freshness，下一轮 `ToolPolicyChecker` 也能用这个 freshness 判断能不能改这个文件。

这比普通聊天摘要实用。普通摘要只回答“前面聊了什么”，我的 working memory 还参与工具策略：编辑前必须 fresh read，recent file 和 file summary 都会影响下一轮的 prompt 和 policy。

## Relevant memory 是按需召回

`retrieval_candidates()` 没用 embedding，用的是 tag、关键词重叠和时间排序，默认取 3 条。

这个选择不花哨，但符合 Pico 现在的定位。行为容易调试，出错时我能直接看出某条 note 为什么被选中。代价也明显：语义召回能力弱，跨语言和同义表达不稳。

Claude Code 的 memory 体系复杂得多，有 `CLAUDE.md` stack、auto memory、team memory、session memory、background extractor。它不只做召回，还按来源、寿命、权限分开管理。我学到的是分层思路，还没做到平台形态。

## Durable memory 是文件化的长期事实

durable memory 放在 `.pico/memory/`，包含 `MEMORY.md` 索引和 topic 文件。长期记忆不会自动吞掉所有会话内容，只通过显式 intent、`<memory>` tag、`/remember`、`/dream` 和 auto-dream gate 慢慢进入。

`promote_durable_memory()` 只接受有限格式的稳定事实，比如：

- `Project convention: ...`
- `Decision: ...`
- `Dependency: ...`
- `Preference: ...`

同时会拒绝 secret 形态的文本、当前目标、下一步、stdout/stderr、过长日志这些不该进长期记忆的内容。

这个口径很重要。长期记忆最危险的问题，就是把临时状态、错误日志和密钥都记进去。

## Compact 是历史瘦身，不是 durable memory

`pico/core/compact.py` 按 turn 分组，把旧 turn 变成一条 `compact_summary` system history，只留最近几个 turn。summary 里会记目标、读过的文件、改过的文件、关键决策、当前进度和下一步。

它和 durable memory 的职责不同：

- compact 服务当前 session 的续航。
- durable memory 服务未来的 session。
- working memory 服务下一轮 prompt。

Claude Code 这层更细，有 snip、microcompact、autocompact、post-compact messages、session memory compact。我现在只有手动 compact 和预算触发 checkpoint，还比较基础。

## 对标 Claude Code 的差距

| 问题 | Pico 当前做法 | Claude Code 参照 |
| --- | --- | --- |
| 指令记忆 | prefix + skills + memory system section | CLAUDE.md 多层栈，managed/user/project/local，支持 include |
| 工作记忆 | `LayeredMemory` 小状态 | session memory markdown，compact 后继续接现场 |
| 长期记忆 | `.pico/memory/MEMORY.md` + topics | auto memory + team memory，后台 extractor 写入 |
| 压缩 | `CompactManager` 汇总旧 turn | snip、microcompact、autocompact、post-compact cleanup |
| cache | prefix hash 和 provider metadata | cache break detection，system/tools/cache-control 多维解释 |

## 当前取舍

上下文治理的主线我已经抓住：不同寿命的信息分开，最新请求不能裁，prompt metadata 必须可审计，记忆不能等于 transcript。

下一步我不打算直接上 embedding，而是先补两个更工程化的点。一是把 prefix section 注册表做实，让每段的来源、稳定性、缓存影响都可解释。二是把 auto-dream 改成更像 Claude Code 那样的受限后台 memory writer，收窄普通 Pico 子运行时在 memory consolidation 里的权限面。

## 设计文档级补充：context 是运行时资产

对 coding agent 来说，context 不是“把历史拼起来”。它是运行时最贵也最脆弱的资产。拼少了模型缺关键证据，拼多了模型被旧信息污染，provider 成本和延迟也跟着涨。

ContextManager 要解决三个问题：

1. 哪些信息每轮都必须出现。
2. 哪些信息可以按预算裁剪。
3. 哪些信息应该离开 prompt，进 memory、compact summary 或 artifact。

### prompt section 的寿命模型

按寿命看 Pico 的 prompt：

| section | 寿命 | 是否固定注入 | 例子 |
| --- | --- | --- | --- |
| prefix | session/runtime 级 | 是 | 工具规则、runtime identity、workspace 摘要 |
| current request | 当前 turn | 是 | 用户最新输入 |
| runtime mode | 当前模式 | 是 | plan mode、tool profile |
| working memory | session 内 | 是，但可裁 | task summary、recent files |
| relevant memory | query 相关 | 否，按需 | durable note retrieval |
| history | session 内 | 可裁 | 旧的 user/assistant/tool turn |
| compact summary | session 内 | 是，替代旧 history | 压缩后的旧 turn |
| durable memory index | 跨 session | 是，短索引 | `.pico/memory/MEMORY.md` |

这张表比“context 里放什么”重要。不同寿命的信息混在一起，是长任务失控的根源。

### 为什么 durable memory 不能等于 transcript

Reflexion 那类研究说明，语言 agent 可以把反馈写成自然语言记忆，在后续尝试里提升决策。但这不等于要保存所有 transcript。反馈记忆必须低噪声、高密度、可复用。

我现在的 rejection 规则就是在做这个过滤：secret、stdout/stderr、traceback、当前下一步、过长日志都不进 durable memory。这和 Dream 文档里的维护逻辑是一套：短期输入流可以脏，长期记忆不能脏。

### compact 的正确边界

Compact 不是总结知识库，它只服务当前会话续航。它该回答：

- 当前任务是什么。
- 已经读过哪些文件。
- 改过哪些文件。
- 做过哪些关键决策。
- 哪些失败已经排除。
- 下一步从哪里继续。

它不该回答：

- 用户长期偏好是什么。
- 项目永久架构是什么。
- 依赖事实是否长期成立。
- 这次任务以外的知识整理。

这些该进 durable memory 或 release docs。

### 与成熟上下文系统的对应

成熟 coding agent 一般把上下文分成多层：

- system prompt：长期行为约束。
- user/project instructions：项目和团队协作契约。
- memory attachments：长期索引和 topic 文件。
- transcript tail：最近会话。
- compact boundary：压缩后的旧历史。
- tool result storage：长结果脱离 prompt，只留引用。
- cache metadata：解释哪些前缀可复用，哪些动态内容会破坏缓存。

我已经有 section 和预算，但 section 注册还不够显式，很多规则散在 ContextManager 和 runtime prefix 构建里。下一步要让每个 section 都能声明：

```text
name
source
lifetime
minimum budget
max budget
cache stability
drop strategy
report metadata
```

这样 context 出问题时，我能直接从 report 看出哪一段被裁掉，不用读代码猜。

### 失败模式和防线

| 失败模式 | 当前防线 | 改进方向 |
| --- | --- | --- |
| 最新请求被裁 | current request 不裁 | 保持 hard floor |
| 历史工具结果孤立 | history rendering 折叠 | 增加 tool pair integrity 检查 |
| durable memory 污染 | rejection rules + Dream | 增加 memory review/report |
| prefix 变动导致 cache miss | prefix hash metadata | 记录具体 section diff |
| compact 丢关键文件 | compact prompt 写 files/decisions | 用 run evidence 校验恢复质量 |
| relevant memory 召回噪声 | 简单 token matching | 后续加 topic-aware retrieval |

### 改进路线

1. **Section registry**：把 prefix、memory、skills、history、todo、runtime mode 都登记成 section spec。
2. **Context report**：每次 run report 记录各 section 的原始长度、渲染长度、裁剪原因。
3. **Tool result storage**：统一长输出落盘，不让大结果挤压 history。
4. **Compact verification**：compact 之后跑一个轻量检查，确认 task、files、decisions、next step 都在。
5. **Memory hygiene report**：Dream 之后写 changed topics、removed duplicate、replaced stale facts。

### 最小验收清单

一次 context/memory 改动至少要证明：

- 当前请求不被裁剪。
- plan mode、todo、worker notification 能进 prompt。
- durable memory 不保存 secret 形态的文本。
- compact 之后仍能恢复 task summary 和 touched files。
- report 能解释各 prompt section 的预算和实际长度。
- 长工具输出不会把 history 挤爆。
