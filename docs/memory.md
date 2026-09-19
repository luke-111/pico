# 分层记忆 + Auto-dream

pico 的记忆系统让 agent **跨 session 保持对项目的认知**。它不把整段对话历史塞回 prompt，而是分四层落地，每层有自己的生命周期。

## 为什么需要分层

把一次 session 的所有内容都喂给下次对话，上下文会爆。完全不记，agent 每次都是第一次见你。分层的思路是：

- **当前任务相关**：保留高保真，但只在本 session 内有效。
- **长期可复用**：经过提炼，跨 session 持久化。
- **零散观察**：先 append-only 写日志，定期再整理。

## 四层结构

```
.pico/memory/
├── MEMORY.md                       # 索引：列出哪些 topic 文件值得看
├── topics/                         # durable memory（4 类）
│   ├── user-preferences.md
│   ├── project-conventions.md
│   ├── key-decisions.md
│   └── dependency-facts.md
├── logs/                           # daily logs
│   └── YYYY/MM/YYYY-MM-DD.md       # append-only
└── .consolidate-lock               # auto-dream 锁文件 + 上次整合时间戳
```

加上 **working memory**（保存在 session JSON 的 `memory` 字段里），一共四层：

| 层 | 生命周期 | 内容 | 注入 prompt？ |
|----|---------|------|---------------|
| **working memory** | session 内 | 当前任务摘要 + 最近接触的文件 + 文件短摘要 | 是 |
| **daily logs** | 永久 append-only | 当天的零散观察、`/remember` 写入 | 否（除非整合后） |
| **durable topics** | 长期，可更新 | 经过 dream 整合的稳定事实 | 是（通过 MEMORY.md 索引） |
| **MEMORY.md** | 长期 | topic 文件的索引（不超过 200 行） | 是 |

## 4 类 durable topic

dream 整合时只写这四个文件：

- `user-preferences` — 用户的角色、知识水平、协作偏好
- `project-conventions` — 仓库约定、构建工具、命名风格
- `key-decisions` — 长期生效的设计决策和理由
- `dependency-facts` — 关键依赖的版本、行为、踩过的坑

## 写入路径

### `/remember <text>`：一行写入 daily log

```text
> /remember 这个项目用 pytest 不用 unittest，并发测试用 pytest-xdist
Saved to daily log.
```

### `<memory>...</memory>`：agent 在 final answer 里自动追加

模型在回答里包一对 `<memory>` 标签，pico 就把内容 append 到当天的 daily log。

### 后台 auto-dream：自动整合

满足这三个条件时后台触发：

- 距上次整合 >= 24 小时（`--dream-interval`）
- 至少有 5 个新 session（`--dream-min-sessions`）
- 当前没有正在跑的 dream

后台会起一个隔离的 pico 实例，write_scope 限制在 `.pico/memory/`，把 daily log 和最近的 session ID 一起交给模型，让它写入或更新 topic 文件和 MEMORY.md。

### `/dream`：手动触发

不想等后台的时候：

```text
> /dream
Consolidation complete. Wrote 2 topic updates, refreshed index.
```

## 读取路径

每轮 prompt 自动注入两段：

1. **memory section**：working memory 加 MEMORY.md 索引，让模型知道有哪些长期记忆可查
2. **relevant_memory section**：按当前用户请求做关键词检索，从 daily log 和 topic 里挑最相关的 3 条

模型也可以手动 `read_file .pico/memory/topics/<name>.md` 读完整 topic。

## 用户可见命令

| 命令 | 说明 |
|------|------|
| `/memory` | 显示 MEMORY.md 索引 |
| `/working-memory` | 显示当前 session 的工作记忆 |
| `/remember <text>` | 追加一条到 daily log |
| `/dream` | 立即整合 daily log → topic |

## 关闭

不需要 memory 的场景：

```bash
pico --no-auto-dream     # 只关 auto-dream，保留 /remember /dream
```

也可以在 toml 或启动时设 `feature_flags.memory = false`，但**不推荐**，这是 pico 区别于其他 coding agent 的核心能力。

## 文件级 freshness 保护

执行 patch_file 或 write_file 之前，pico 会用 sha256 freshness 检查这个文件最近有没有被 read 过。没读过就改，会被 `prior_read_required` 拒绝。这层保护和 memory feature flag **是解耦的**：即使关掉 memory，read freshness 仍然在追踪，避免 agent 改自己没看过的文件。

## 故障排查

| 现象 | 原因 / 解决 |
|------|-------------|
| `/dream` 输出 `nothing to consolidate` | daily log 是空的，先用 `/remember` 写几条 |
| auto-dream 不触发 | 看 `.pico/memory/.consolidate-lock` 的 mtime，距上次整合不到 24 小时 |
| topic 文件没更新但 dream 说成功 | 早期版本的已知问题，2026-05 已修复，freshness 追踪不再依赖 memory feature flag |
| MEMORY.md 太长 | dream 会自动裁到 200 行，也可以手动 `/compact` |

## 推荐的工作流

1. 第一次进项目：让 pico 跑 `/skills`，看一遍 README，再用 `/remember` 写下 1 到 2 条这个仓库的关键约定。
2. 每天收工：跑一次 `/dream`，把当天的观察沉淀下来。
3. 切分支或者隔了一段时间再用：直接 `pico --resume latest`，让它从工作记忆和 topic 里恢复上下文。

记忆只存在本地，**不会上传**。删掉 `.pico/memory/` 就回到第一次见你的状态。
