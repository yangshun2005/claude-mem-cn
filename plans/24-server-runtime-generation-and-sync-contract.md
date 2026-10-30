<!-- GitHub Issue #3618 的镜像。Issue 是公开跟踪器；本文档是设计文档。
     若两者不一致，设计以文档为准，状态以 Issue 为准（oh-my-issues 健康检查）。 -->

> **跟踪器：** https://github.com/thedotmack/claude-mem/issues/3618

# [plan-24] 服务端运行时与云同步契约 — worker 与服务端共用一条生成流水线，所有写入路径都做会话关联，同步有界

> **跟踪器：** #3618 · **设计文档：** `plans/24-server-runtime-generation-and-sync-contract.md` · **系列：** plan-14 #3602 · plan-15 #3603 · plan-16 #3604 · plan-17 #3605 · plan-18 #3606 · plan-19 #3607 · plan-20 #3608 · plan-21 #3609 · plan-22 #3610 · plan-23 #3611 · plan-24 #3618

## 缺陷

服务端运行时（`src/server/`，Postgres + BullMQ 生成任务，`/v1/*` 路由，`workers/sync-hub`）和客户端云同步（`CloudSync.ts`、`sync_outbox`）是 worker 生成与存储流水线的**第二份手工复制实现**，没有共享契约，也没有契约测试。两条路径本应一致的每一处都已漂移——而服务端在退出时把任务标记为 `completed`，所以漂移是静默的：

- **提示词/解析契约是复制的，不是共享的。** `buildServerGenerationPrompt` 同时用于事件任务和摘要任务，且总是请求 `<observation>` 块；`parseAgentXml` 对这种形状返回 `summary: null`；`processSessionSummaryResponse` 只读 `parsed.summary` → 每个摘要响应都被丢弃，任务却标记为 completed（4,836 个 completed 摘要任务，0 行）。与 SDK 路径中的 #1345 同类缺陷，在新运行时中再次出现。
- **摘要输入集合错误，计量单位也错误。** 摘要任务的数据来自 `listUnprocessedEvents`（恢复残留，按数量上限 500，排除已有 completed 单事件任务的事件），所以大多数会话交给模型的是空的或不完整的 `<agent_events>` 块；而且上限按事件*数量*而非渲染*字节数*，长会话会溢出上下文窗口并被归类为 `unrecoverable`。大小估算放在一个手工复制的辅助函数里，已与真实的块构建器漂移。
- **任务标识在幂等 upsert 后不稳定。** `ON CONFLICT (idempotency_key)` 保留现有行的 `id`，却把新生成的 id 以 JSONB 合并进 `payload.generation_job_id`；随后 `lockOutbox` 找不到任何东西，向 BullMQ 报告 `completed`。只有按会话的幂等键（摘要）会冲突，所以看起来像摘要 bug 而非队列 bug。
- **会话关联/项目元数据只加在一条写入路径上。** `/v1/events` 解析 `contentSessionId → server_sessions.id`；`/v1/memories` 从未这样做（2,948/2,948 行 `server_session_id NULL`）；生成的 observation/摘要从不把 `server_sessions.metadata.project` 复制到行上，所以按项目注入/搜索无法限定服务端生成的数据；注入面板打印完整 UUID，无法用作抓取句柄。
- **Hook 和上下文假定本地 worker。** `hooks.json` 的 `start` 以及每次 `hook <platform> <event>` 调用都无视 `CLAUDE_MEM_RUNTIME` 直接调用 `ensureWorkerStarted()`；在服务端运行时中，MCP 搜索服务器占着 worker 端口，hook 会为一个永远不应答的健康检查等待约 180 秒；SessionStart 上下文处理器没有 `/v1/context` 分支。
- **客户端同步没有边界。** `sync_outbox` 的生产者无条件写入，而唯一的消费者（`CloudSync.drainMutations`）在没有凭据时从不构造，默认安装下每个 `DELETE FROM sync_outbox` 位置都不可达；v2 投影重写丢掉了 `substr(prompt_text,1,200000)` 截断（数 MB 提示词 → `SQLiteError: out of memory`，10 名用户 682 次崩溃），并在循环内按提示词逐个准备语句；sync-hub 的投影修复在单个请求内、持有每用户租约的情况下排空无界积压。

统一机制：**worker 路径与服务端/同步路径之间没有共享契约**，所以对一条路径的每个修复都得在另一条上重新发现，而服务端路径的 "completed" 并不能证明写了任何东西。

## 子项

由 2026-08-16 PR 合并整理归入此处的开放 PR（撰写时无开放 Issue——症状由服务端运行时运维者通过 PR 报告）：

- #3587 — 摘要任务拿到 observation 形状的提示词；每个摘要被静默丢弃（#1345 复发）；输入集是未处理残留，不是会话
- #3584 — 摘要输入按事件数量而非渲染字节数设界 → 上下文溢出 → `unrecoverable`，无摘要
- #3583 — 幂等任务 upsert 让 `payload.generation_job_id` 指向幽灵行；`lockOutbox` 返回 null → BullMQ `completed` 但无写入
- #3586 — `/v1/memories` 从不解析 `contentSessionId`/平台范围（所有行 `server_session_id NULL`）；`/v1/events` 会
- #2671 — 服务端生成的 observation/摘要写入时 `metadata.project = null`；无法按文件夹限定
- #2867 — 注入面板打印完整 Postgres UUID（每个约 17 token），无法用作抓取句柄
- #3227 — 服务端运行时中 hook 调用 `ensureWorkerStarted()`（停顿 180 秒）；SessionStart 上下文没有服务端分支
- #3537 — CloudSync 投影丢掉了 200 KB 的 `substr` 截断（OOM 崩溃）；重新入队循环中语句句柄泄漏
- #3555 — sync-hub 修复在单个租约/请求下排空无界积压；调用方把任何 2xx 当作完成
- #3228 — 自定义服务端 provider 模块（扩展点；单独决定）
- #3498 — CMEM Pro 的 `cloud connect` CLI（功能；依赖私有应用改动）

相关，因缺陷在本地队列而路由到 plan-21：#3616（默认安装下 `sync_outbox` 无界）、#3597（合成会话 id 重铸 × 提示词重新入队放大）。

## 修复顺序

1. **一份按任务类型键控的提示词/解析契约。** `buildGenerationPrompt({sourceType})` 和 `parseAgentXml` 在两个运行时上是*同一批模块*（`src/sdk/`），摘要变体请求 `<summary>`，解析器的 `skip`/`empty`/`prose_salvageable` 分类（plan-18 步骤 1）以相同方式应用；摘要输入是**完整会话事件集**（不是 `listUnprocessedEvents`），由块构建器自身测量的渲染字节数（`eventBlockBytes`）设界，头+尾选取，并计入信封。产生零行的摘要任务是 `failed:<reason>`，绝不是 `completed`。
2. **能挺过 upsert 的任务标识。** 生成任务 upsert 把 `payload.generation_job_id` 重新钉到存活行 id（或者 id 根本不存进载荷）；`lockOutbox` 返回 null 是 ERROR，记录幂等键并把任务标记为 `failed`，不是 `completed`。Postgres 门控的回归测试在 CI 中运行（服务容器）。
3. **每条写入路径解析相同的会话/项目身份。** `/v1/events`、`/v1/memories`、MCP `observation_add` 和生成的行都经过同一个 `resolveServerSession({contentSessionId, platformSource})`，并在写入时从会话（或 `resolveProjectIdentity()`，plan-20）打上 `metadata.project`；当 `fetchByIdSupported=false` 时注入面板渲染 8 字符前缀并附图例。现有 NULL 行的回填是一次性修复。
4. **运行时感知的 hook 和上下文。** `CLAUDE_MEM_RUNTIME=server` 在每个 hook 入口和 CLI `start` 中短路 `ensureWorkerStarted()`；SessionStart 上下文处理器增加 `/v1/context` 近期分支；Codex/OpenCode 适配器（plan-23）继承同一开关。服务端运行时中的 hook 必须在 worker 端口被无关监听器占用时于 5 秒内完成。
5. **两端同步都有界。** 客户端：恢复 `substr(prompt_text,1,N)` 并用 NUL 安全截断（`length()` 按字节而非字符），预编译语句缓存在循环外，`sync_outbox` 生产者在消费者存在时才写入，并在稍后启用同步时用 `set_title` 回填（plan-21 拥有 outbox 表）。Hub：修复端点每请求处理一页，在每个检查点释放租约，未完成时返回 202，调用方循环直到 200。
6. **契约测试，而非靠肉眼比对一致性。** 单一 fixture 语料（事件批次、摘要批次、记忆写入）在 CI 中通过*两个*运行时重放，结果行逐字段比较（`project`、`session_id`/`server_session_id`、`type`、`title`、`files_*`）；任何加到一条路径的新列都会让另一条的契约测试失败。

## 测试矩阵

| 运行时 | 任务 / 路径 | 输入 | 必需行为 |
|---|---|---|---|
| server | `session_summary` 任务 | 40 事件会话，所有事件已处理 | 一行 `session_summaries`；仅在行存在后任务才 `completed` |
| server | `session_summary` 任务 | 2,000 事件会话（> 上下文窗口） | 头+尾有界输入；写入行；无 `unrecoverable` |
| server | `session_summary` 任务重跑 | 相同幂等键 | upsert 重新钉住任务 id；`lockOutbox` 找到行；无幽灵 `completed` |
| server | `/v1/memories` | `contentSessionId` + `platformSource=codex` | 按平台范围解析 `server_session_id`；不匹配 Cursor 会话 |
| server | 生成的 observation | 带 `metadata.project` 的会话 | 行 `metadata.project` 已填充；按项目限定的 `/v1/context` 返回它 |
| server | SessionStart hook | `CLAUDE_MEM_RUNTIME=server`，端口被 mcp-search 占用 | 5 秒内返回上下文；不调用 `ensureWorkerStarted` |
| local | CloudSync 投影 | 4 MB 提示词，内嵌 NUL | 截断到 N 字节，无 OOM，不确认为完全同步 |
| local | 默认安装（无凭据） | 100 会话 × 20 提示词 | `sync_outbox` 行数有界（无消费者时零增长） |
| hub | 投影修复 | 10,000 行积压 | 每页 202，页间释放租约，调用方收敛到 200 |
| both | 契约重放 | fixture 语料 | worker 与服务端路径逐字段行相等 |

矩阵在 CI 中（服务端行用 Postgres 服务容器）。未来的回归必须在用户提交前让 CI 失败。

## 范围外

观察者输出分类和历史预算本身（plan-18——本计划只要求服务端路径*使用*相同模块）。本地 `sync_outbox` schema/状态机（plan-21）。项目键推导（plan-20）。宿主适配器 schema（plan-23）。新 Pro 功能（`cloud connect`、自定义 provider 模块）在此作为子项跟踪，但属于产品决策，不是缺陷修复。