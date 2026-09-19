# 子 agent、计划模式和 Todo：控制面怎么长出来

我的子 agent 是主 runtime 下面的受限 child run。它解决长任务里的分工问题：主 agent 不该把所有探索、修改和跟进都塞进一个上下文，子 agent 也不能变成没有边界的第二个全能 agent。

![子 agent、计划模式和 Todo](assets/05-workers-plan-todo.png)

## WorkerManager 管生命周期

`pico/core/worker_manager.py` 负责 worker 的生命周期：

- `spawn()` 创建任务。
- `continue_task()` 续接 idle worker。
- `drain_notifications()` 把 worker 完成消息注入主 history。
- `stop_task()` 请求停止。
- `shutdown()` 在退出时请求后台 worker 停止。

每个 worker 是一个 `WorkerTask`，包含 id、description、subagent_type、write_scope、child runtime、thread、stop flag 和 runtime state。

最关键的是 `_new_task()`。它调用 `build_child_runtime()` 生成一个新的 `Pico` 子 runtime，而不是把 prompt 丢给同一个 agent。

## Explore 和 worker 的边界

我现在支持两类子 agent：

- `Explore`：只读，approval policy 是 `never`，tool profile 是 `readonly`。
- `worker`：可写但必须受 `write_scope` 约束，tool profile 是 `worker`，不暴露 `run_shell`。

plan mode 下只能启动 `Explore`。这条边界很重要：计划阶段应该允许调查，但不能通过子 agent 绕过写限制。

## 后台执行靠 model_client_factory

`WorkerManager._can_run_background()` 要求 parent 有 `model_client_factory`。有 factory 就起 thread 后台跑，没有就同步跑。

这里有个实际原因：同一个 model client 未必线程安全，也未必能并发请求，child runtime 应该拿新的 client。没有 factory 时同步执行虽然慢，但行为可控。

## send_message 是续接同一个 child runtime

`send_message` 不是新开一次任务。`continue_task()` 会找到 active worker task，用同一个 child runtime 继续跑，这样 child 的 history、memory、tool state 才能延续。

这比“每次 send_message 都重新 spawn 一个 agent”更接近真实协作。主 agent 可以先让 Explore 查入口，结果回来后再问它更窄的问题。

## worker notification 回到主循环

`worker_execution.py` 跑完 child runtime 后，把结果、tool_steps、attempts、worker artifacts、duration 写回 worker item，再放进 notification queue。主 `Engine` 在模型请求前、工具之后、final 之前都会 drain notifications，并把通知记进主 history。

这样后台 worker 就不只是默默跑完。它的结果会进主 agent 的后续 prompt，主 agent 能基于这些结果继续决策。

## Plan mode 是写边界，不只是提示词

`pico/core/plan_mode.py` 进入计划模式后，会把 session 的 `runtime_mode` 改成：

```text
mode: plan
topic: ...
plan_path: .pico/plans/<topic>-plan.md
```

同时切到 `plan` tool profile，刷新 prefix。plan 模式下：

- 可以读文件。
- 可以用 todo。
- 可以启动 Explore。
- 写操作只能写 active plan artifact。
- final 之前必须保证 plan artifact 非空。

这比在 prompt 里写一句“你现在在计划模式”可靠得多，因为真正的写边界在 `PermissionChecker` 里执行。

## TodoLedger 是运行时任务账本

`pico/core/todo_ledger.py` 管 session 级 todo。它支持 `pending / in_progress / done / blocked` 和 `low / normal / high`，每次 add/update 都会写 session、发 event，并记进当前 `TaskState.todo_changes`。

Todo 会进 `ContextManager` 的 memory section。模型下一轮能看到当前任务账本，run report 也能记录 todo 变化，所以它不是 UI 装饰。

## 和 Claude Code 的对标

Claude Code 在这一层明显更大。它有 `AgentTool`、`SendMessageTool`、`TaskCreateTool`、`TaskUpdateTool`、`TaskStopTool`、TeamCreate/TeamDelete、local/remote/in-process teammate、worktree 模式和任务输出文件。`Task.ts` 里的 task type 也更细，能区分 local bash、local agent、remote agent、workflow、monitor、dream。

我这边只做了最小控制面：

| 维度 | Pico | Claude Code |
| --- | --- | --- |
| 子 agent 类型 | Explore / worker | local agent、remote agent、in-process teammate、team |
| 续接 | `send_message` 续接 child runtime | resume agent、teammate mailbox、task output |
| 停止 | stop flag + abort child turn | task kill、interrupt、terminal state |
| 写边界 | worker `write_scope`、plan artifact | permission context、worktree、scratchpad、agent-specific context |
| 任务账本 | TodoLedger session 状态 | Task tools、TaskList UI、disk output |

## 当前取舍

我这套子 agent 设计最值得保留的是边界清楚：Explore 只能读，worker 不能用 shell，写入必须有 scope，plan mode 不能启动 worker。这个取舍比一上来做复杂多 agent 协作重要。

下一步我想补三层。一是把 worker output artifact 更系统地纳入主 report。二是把 worker 的 tool profile 再细分，比如允许跑测试但不允许任意 shell。三是让 plan mode 和 todo ledger 形成明确的交付门槛，比如计划项全部 done 或者有 blocked reason 才允许退出计划。

## 设计文档级补充：复杂任务的控制面

单 agent loop 最大的问题不是完不成任务，而是所有状态都挤在同一个上下文里。探索、计划、修改、验证、复盘、子任务结果混在一起时，模型很容易忘边界。

我引入 plan、todo、worker，本质是在主循环外面建一个复杂任务控制面。

```text
Plan mode: 限制写入范围，先形成计划 artifact
Todo ledger: 把任务拆分变成 session state
Worker manager: 把子任务放进受限 child runtime
Notification queue: 把子任务结果回流主循环
```

### Plan mode 为什么必须是 runtime mode

plan mode 如果只是 prompt 文案，模型照样可能调用写工具改源码。我的做法是把 plan mode 写进 runtime state，并切换 tool profile。

这带来三个效果：

- 权限层能识别当前是 plan mode。
- 写操作只能落到 active plan artifact。
- final 之前可以检查 plan artifact 有没有内容。

这比“请先制定计划”可靠，因为约束在工具边界执行，不靠模型自觉。

### TodoLedger 为什么不是 UI 装饰

Todo 只显示在 TUI 的话，模型下一轮不一定看得到。我的 TodoLedger 是 session state：

- add/update 会写 session。
- todo changes 会进 TaskState。
- todo view 会进 prompt。
- report 能看到 todo 变化。

这样 todo 就成了控制面的一部分。它帮模型在长任务里维持任务分解，不用用户在脑子里记。

### Worker 是受限 child runtime

我的 worker 不是“另一个完全自由的 agent”，它是 parent runtime 构造出来的 child `Pico`，带着更窄的工具 profile 和写边界。

现在有两类：

| 类型 | 权限 | 适合场景 |
| --- | --- | --- |
| Explore | readonly，approval never | 计划阶段调查代码、找入口、读文档 |
| worker | 可写但有 write_scope，不暴露 shell | 执行明确的子任务 |

这条边界防住两个问题：

- plan mode 通过子 agent 绕过写限制。
- worker 在没有 scope 的情况下变成第二个主 agent。

### notification 回流是协作的关键

子 agent 的结果如果只写在自己上下文里，主 agent 不会自然知道。我用 notification queue 把 worker 完成消息注入主 history 和 prompt。

这看起来像 UI 细节，其实是多 agent 协作的核心。主 agent 必须看到：

- 哪个 worker 完成了。
- 做了什么。
- 是否成功。
- 有哪些 artifact。
- 需不需要继续追问。

否则 worker 只是后台脚本，不是协作单元。

### 和成熟任务系统的对应

成熟 coding agent 的任务系统一般有更完整的 task lifecycle：

```text
created -> queued -> running -> waiting -> completed/failed/canceled/archived
```

任务对象可能带 output file、owner、parent tool use id、remote/local type、UI progress、kill handler、resume handle。我现在只有轻量的 WorkerTask 和 TodoLedger，但最小必要边界已经覆盖：spawn、continue、stop、notification、write_scope。

### 失败模式和防线

| 失败模式 | 当前防线 | 缺口 |
| --- | --- | --- |
| plan mode 下直接改源码 | plan profile + permission | plan artifact 质量检查弱 |
| worker 越权写文件 | write_scope | scope diff/report 可以更强 |
| 子任务完成主 agent 不知道 | notification drain | prompt budget 紧张时 notification 可能被挤压 |
| worker 卡住 | stop flag + shutdown | 没有完整的 task timeout 和 heartbeat |
| todo 变成形式主义 | prompt 注入 + task_state | 没有 done/blocked exit gate |
| 多 worker 互相冲突 | write_scope 约束 | 没有 worktree 隔离 |

### 改进路线

1. **Task lifecycle**：把 WorkerTask 状态扩展成 queued/running/waiting/completed/failed/canceled。
2. **Worker output contract**：每个 worker 必须产出 summary、changed_paths、artifacts、follow-up question。
3. **Plan exit gate**：退出 plan mode 前检查计划结构、scope、verification path。
4. **Todo completion gate**：final 之前 todo 没完成的话，要求 blocked reason 或 continuation plan。
5. **Worktree worker**：大改动的 worker 用隔离 worktree，减少并行冲突。

### 最小验收清单

这一层的改动至少要验证：

- plan mode 能读、能写 plan artifact，但不能写普通源码。
- Explore 在 plan mode 可用且只读。
- worker 必须受 write_scope 限制。
- `send_message` 续接同一个 child runtime，不是重开任务。
- `task_stop` 能让 worker 进入停止流程。
- worker 完成通知会进主循环可见的上下文。
- todo changes 会写进 task_state 和 report。
