# 无尽模式 — 漂流瓶

**日期：** 2026-07-16（rev 2 — 同日：§13 的四个待定问题均已通过代码调查解决；§4/§5/§6/§8/§9 已对照实际代码库修正）
**状态：** 草案规格，已通过代码验证
**功能名称：** 无尽模式
**一句话说明：** 会话通过渲染一份平行记录来挺过压缩（以及 `/clear`、崩溃、重启）——逐字对话 + 以观察替代工具流量——并用一条指令把压缩后的模型指向它：*读取漂流瓶并继续。*

---

## 1. 问题

Claude Code 会话不会在上下文上限处死亡——它们会失忆。自动压缩用一个有损的、批量生成的摘要替换对话，而写摘要的总结器并不知道哪些细节重要。模型丢失了它的计划、承诺、语气，以及已经尝试过什么的知识。claude-mem 在 SessionStart 注入的时间线能部分恢复长期记忆，但目前没有任何东西能恢复*对话位置*——模型实际所处的位置，工作中途，用它自己的话。

我们不依赖 Claude Code 生成的摘要。我们无法抑制它（PreCompact Hook 只能观察），所以我们绕过它，并把我们的注入标记为权威。

## 2. 核心洞察

会话记录按 token 计 80–95% 是工具流量。对话本身——用户消息和助手消息——才是稀薄而珍贵的部分。claude-mem **已经实时压缩了工具流量**：观察就是压缩结果。因此，一份近乎无损、小 10–20 倍的会话记录已经在生成时以碎片形式存在：

```
bottle = merge_by_position( transcript.jsonl , observations.db )
```

- **用户消息** — 逐字，来自记录（权威来源）。
- **助手消息** — 逐字，来自记录（完整且有序；系统中没有其他东西持久化它们——见 §10）。
- **工具调用 + 工具结果** — *替换*为它们产生的观察，作为带 ID 的单行条目，可通过 search/`get_observations` 无损展开。

漂流瓶是**派生视图，不是维护的文件**。不捕获任何新内容；实时流水线保持不变。磁盘上的文件是渲染缓存——整体重新生成，原子写入，从不追加，从不是权威来源。

### 为何派生优于维护

1. **无双写漂移。** 持续追加的文件是第二个权威来源，会分叉（漏掉的 Hook、崩溃的追加、撕裂的行）。渲染在封存时始终与权威来源一致。
2. **异步观察的排序问题反转。** 观察相对对话晚几秒落地。实时追加意味着乱序条目或跨 Hook 进程的读-改-写竞争。渲染*回看已稳定的数据*——按位置归并排序——滞后问题基本消失（尾部见 §5）。
3. **两个来源均只读。** 记录文件读取 + WAL SQLite 读取对现有流水线零写竞争。
4. **一切可追溯。** 格式升级重新生成历史。任何*过去*的会话都可装瓶——包括 Claude Code 已修剪记录的会话（§10）。

## 3. 架构位置

渲染发生在 **worker 侧**（worker 拥有 DB、SDK agent 和每会话状态；按现有优雅降级契约，Hook 保持轻薄）。Hook 只触发和注入。

```
PreCompact hook ──POST /api/sessions/seal──▶ Worker
                                               ├── bounded drain-wait on SessionMessageBuffer
                                               ├── render bottle (transcript + observations + buffer stubs)
                                               ├── write atomically to bottles dir
                                               └── arm injection marker (consumed_at=NULL)

SessionStart(compact|resume) hook ── claim marker ──▶ inject one-line pointer
SessionStart(clear)                ── archive bottle, no injection
SessionStart(startup)              ── existing behavior, unchanged
Stop hook                          ── await render-bottle queue-and-return (fire-and-forget effect)
```

新增表面（所有位置均已对照代码库验证）：

| 组件 | 变更 |
|---|---|
| `plugin` Hook | 新增 PreCompact 处理器；SessionStart 处理器按 `source` 分支；summarize（Stop）处理器新增一个 await 调用 |
| 插件归一化 | 贯穿传递 `tool_use_id`：`src/cli/types.ts`（`NormalizedHookInput`）、`src/cli/adapters/claude-code.ts`（`normalizeInput` 当前丢弃它）、`src/cli/handlers/observation.ts`（POST body——worker 路由已接受 `tool_use_id`，`SessionRoutes.ts:288-299`） |
| Worker HTTP | 新增 `BottleRoutes`（`POST /api/sessions/seal`、`POST /api/sessions/render-bottle`、`GET /api/sessions/drain-status`）——`BaseRouteHandler` 子类，构造注入，zod 校验，在 `worker-service.ts registerRoutes()` 中注册 |
| `SessionMessageBuffer` | 在 `confirm()`/`clear()` 时发出 `'mutate'`（与入队时现有的 `'message'` 并列）；新增 `waitForDrain()` 原语（镜像现有 `waitForMessage()` `SessionMessageBuffer.ts:196-226`）；新增 `peekStubs()`（更丰富的 `peekTypes`） |
| Worker 服务 | 新增 `BottleRenderer`（记录解析 + 合并 + markdown 输出 + `stripMemoryTags`），带每会话渲染互斥锁（复制 `ChromaSync.backfillInProgress` 守卫，`ChromaSync.ts:913-931`） |
| SQLite | 新增 `session_bottles` 表（注入标记 + 记录字节游标）。**v1 无需更改观察路径**（§5、§6） |
| 设置 | 可选总开关 `CLAUDE_MEM_ENDLESS_MODE_ENABLED` 默认 `'true'`，位于 `SettingsDefaultsManager`（接口 + `DEFAULTS`），worker 侧检查（§8） |

其余一切——SDK agent、ResponseProcessor、ChromaSync、时间线、搜索——不变。

> **过时文档说明：** `docs/architecture-overview.md` 列出了一个 `SessionEnd | session-complete` Hook；它不存在（`plugin/hooks/hooks.json` 无条目，`src/cli/handlers/index.ts` 无处理器）。§8 的会话结束归档步骤是**新接线**，不是扩展——决策见 §13 R5。

## 4. 会话标识

漂流瓶以 **`contentSessionId`** 为键（Claude Code 的 ID——会话生命周期内不变，在每个 Hook 的 stdin 中传递，且挺过压缩）。

**rev 1 修正：** 不存在"跨所有 memorySessionId 收集"的问题。`sdk_sessions` 每个 `(platform_source, content_session_id)` 一行；worker 重启时*同一行的* `memory_session_id` 原地 UPDATE（`SessionStore.updateMemorySessionId` :1475-1482），且 `observations` 和 `session_summaries` 的外键均为 `ON UPDATE CASCADE`（迁移 v21，`SessionStore.ts:1040-1180`；`PRAGMA foreign_keys = ON` 于 `connection.ts:45`）。所有先前腿的观察自动级联到当前 `memory_session_id`。渲染只需一次连接：

```
contentSessionId → sdk_sessions row → memory_session_id → observations + session_summaries
```

**社区边缘注意事项：** 线上 DB 可能带有来自社区迁移 v36–38 的 `observations.content_session_id`，这些迁移*不在本仓库中*（缺口已在 `SessionStore.ts:447-451` 承认），且仅部分填充。主线代码必须始终使用 `sdk_sessions` 连接，绝不使用 `observations.content_session_id`。

## 5. 未观察的尾部（难点）

### 问题

PreCompact 触发时——通常是**回合中途**，因为自动压缩在长时间工作回合中触发——最新的工具事件可能尚未有观察。每个响应的观察数量*按设计*可变（agent 决定什么值得观察——这种可变性是产品的魔力，不能被约束）。因此完整性永远无法通过用观察数对比工具调用来确定。

### 解决方案：覆盖，而非计数

我们从不需要知道"应该存在多少观察"。我们需要知道**哪些输入已被完全处理**。这是一个排空问题，机制已存在——rev 1 有两处修正，均已在代码中验证：

> **修正：** (1) `clearPendingForSession` 是死代码——零调用点（`SessionManager.ts:207-209` 存在但从未被调用）。实际的完成信号是 **`confirmClaimedMessages`**（`SessionManager.ts:219-231`），由 `ResponseProcessor.ts:210` 在 `storeObservations()` 提交后（:126）调用，以及在 :87 的散文/无操作路径调用。(2) 待处理队列是**内存中**的——`SessionMessageBuffer` 明确取代了持久化的 `pending_messages` SQLite 表（文档注释 `SessionMessageBuffer.ts:21-41`：*"如果 worker 死亡，缓冲区消失，恢复是记录重放"*）。该表仅作为迁移/清理残留存在。rev 1 声称"待处理行跨 worker 重启持久化"是错误的；下面的诚实尾部机制正是围绕这一点设计的。

**排空谓词是一个数字。** `claimNext()` 只把消息标记为 `claimed: true`（`SessionMessageBuffer.ts:186-194`）；只有 `confirm()` 移除它们（:74-84）。一个批次在 SDK 为其生成响应的整个期间都留在缓冲区中。因此：

```
buffer.getPendingCount(sessionDbId) === 0  ⟺  queue empty AND no batch in flight
```

排空测试不需要单独的"生成中"布尔值。（对于 `drain-status` 报告，`generating` = `session.claimedMessageIds.length > 0` 或 `session.generatorPromise !== null`，`worker-types.ts:17,23`。）

### 封存操作

```
POST /api/sessions/seal   { contentSessionId, transcriptPath, timeoutMs? }   // default 20_000, max 60_000
```

Worker 行为（完整调查见 §13 R2）：

1. 通过 `createSDKSession` upsert 解析 `sessionDbId`（惯用模式，`SessionRoutes.ts:358`）。
2. `buffer.waitForDrain(sessionDbId, deadline, generatorDead)` — **混合等待**：在缓冲区的每会话 `EventEmitter` 上唤醒（`'mutate'` 在 `confirm`/`clear` 时发出），每次唤醒重新检查谓词，带 250ms 安全滴答和硬截止。纯事件等待不安全，因为**生成器死亡不改变任何缓冲区状态**；纯轮询浪费了现有 emitter 基础设施免费提供的即时唤醒。
3. **绝不中断或强制生成。** 特别是，在配额暂停期间不要从 seal 调用 `ensureGeneratorRunning`——提供方已被限流，强制启动会在注定失败的生成上烧预算。
4. 生成器死亡检测：`session && session.generatorPromise === null && pendingCount > 0`——此状态的持久形式是**配额暂停**（`GeneratorExitHandler.ts:28-65`：配额退出保留 session + buffer，`generatorPromise = null`；其他所有退出都会终结并完全处置缓冲区）。因为 `ingestObservation` 在 `ensureGeneratorRunning` *之前*入队（`http/shared.ts:121,138`），该条件在每次新入队时短暂为真——要求它在**连续两次唤醒（约 500ms 宽限）**内保持，才提前返回。
5. 排空后 → 渲染，返回 `{ sealed: true, gap: 0 }`。超时或生成器死亡 → 带诚实尾部渲染，返回 `{ sealed: true, gap: pendingCount }`。

### 诚实尾部——两个来源，因为缓冲区在 RAM

**在单个 worker 生命周期内**，缓冲区本身提供存根：每个排队*和飞行中*的消息都保留 `tool_name`、完整可 JSON 解析的 `tool_input`（`stripMemoryTags(JSON.stringify(toolInput))`，`http/shared.ts:114-116`）、`enqueuedAt` 和 `toolUseId`（`worker-types.ts:63-74`，`SessionMessageBuffer.ts:7-12`）。唯一缺失的是读取表面：添加 `peekStubs(sessionDbId)`，返回 `{message_type, tool_name, tool_input, enqueuedAt, toolUseId, claimed}`（约 10 行；当前 `peekTypes` :138-143 只投影 `{message_type, tool_name}`）。**无 schema 变更，无迁移。** 简要切片在渲染时从解析的输入派生：Bash→`command`，Edit/Write/Read→`file_path`，Grep→`pattern`，否则取第一个键；跳过 `summarize` 类型消息（无工具字段）。

**跨 worker 重启**，缓冲区消失，队列派生的缺口不可知。因此渲染器*也*从记录派生尾部：比最新观察锚点位置更新的 tool_use 条目渲染为存根。队列缺口在单个 worker 生命周期内是权威的；基于记录的比较是始终有效的底线。

渲染形式：

```markdown
*Recent activity (observations pending — will appear in the timeline shortly):*
- ⏳ Bash — `npm test` (14:32:11)
- ⏳ Edit — src/services/worker/SessionManager.ts (14:32:40)
```

此缺口*按构造*可接受的三个原因：

1. 逐字的最终助手消息覆盖尾部叙述——模型自己刚描述了它在做什么。
2. 丢失的缓冲工作的恢复是**记录重放**（缓冲区自己文档化的契约）——观察被重新生成，而非丢失。
3. 漂流瓶是派生视图——**下一次渲染自动愈合缺口**（Stop 时渲染，§8）。

seal 从不阻塞压缩，从不阻塞用户：任何 worker 失败时 Hook 遵循现有契约（传输错误 → exit 0），注入回退到当前仅时间线的上下文块。无尽模式降级为今天的行为，绝不更差。

## 6. 交错：将观察锚定到记录位置

为按时间顺序合并，观察需要一个*源位置*，而不仅仅是生成时间戳。

**状态（已验证）：锚点已半交付。** 排空产生 `_originalTimestamp = enqueuedAt`（`SessionMessageBuffer.ts:160-164`）；`SessionManager` 将批次最小值跟踪为 `earliestPendingTimestamp`（:362-367）；`ClaudeProvider.ts:355` 将其作为 `originalTimestamp` 传递；`ResponseProcessor.ts:126-135` 将其作为 `overrideTimestampEpoch` 传给 `storeObservations` → 存储为 `observations.created_at_epoch`（`SessionStore.ts:2088-2100`）。已存储的观察**已用批次最小工具时间戳标记，而非生成时间。**

剩余工作，分两部分：

1. **v1，插件侧，无迁移：** 贯穿传递 `tool_use_id` 通过 Claude Code Hook 路径——它到达 Hook stdin（`PostToolUseHookInput.tool_use_id`，SDK 类型）但 `claude-code.ts normalizeInput`（:15-25）丢弃了它。在 `NormalizedHookInput`（`src/cli/types.ts`）中添加 `toolUseId?`，在适配器中映射，包含在观察 POST body 中（`observation.ts:18-31`）——worker 路由已接受并转发它（`SessionRoutes.ts:288-299,308-332`；记录重放路径已发送它，`transcripts/processor.ts:205,233,254`）。可选地也从 Hook 发送 `occurred_at_epoch: Date.now()`，使锚点是工具完成时间而非 worker 接收时间。两个偏移方向的向后兼容都很简单：路由 schema 是 `.passthrough()` 带可选字段。
2. **v2，单独变更：** `observations` 上的每观察锚点列（`anchor_ts_min/max`、`tool_use_id`）——需要可空列迁移加上 `ResponseProcessor`/`storeObservations` 签名变更。v1 渲染不需要。

**零成本的定位回退：** `observations` 和 `user_prompts` 都携带 **`prompt_number`**。按提示编号分组观察（组内按 `created_at_epoch` 排序）给出*精确*的对话定位，无需记录也无需新列——这也使降级漂流瓶（§10）正确排序。回合内的批次范围精度本质上是近似的（SDK agent 对批次写自由形式）；漂流瓶需要的是时间上合理的放置，而非精确因果。

## 7. 漂流瓶：格式与渲染

### 位置

```
~/.claude-mem/bottles/<contentSessionId>.md        # active
~/.claude-mem/bottles/archive/<ts>-<id>.md          # archived on /clear or session end
```

每会话文件名——同一项目中的并发会话不得共享文件。

### 写入纪律

整体渲染，写入临时同级文件，`rename()` 覆盖目标（POSIX 上原子；先例：`src/shared/atomic-json.ts:50,100`）。从不追加。**每会话渲染互斥锁**（复制 `ChromaSync.backfillInProgress` 静态守卫模式，`ChromaSync.ts:913-931`）：同一会话的 Stop 渲染和 seal 渲染不得交错——没有互斥锁，两个并发渲染是最后写入者胜出，游标不同。

渲染器维护每会话**字节偏移游标**指向记录（`session_bottles.transcript_cursor`），使常规重渲染只解析新字节——先例是工作代码：`JsonlTailer`（`src/services/transcripts/watcher.ts:10-72`，偏移高水位标记带截断重置）。调查中的两个诚实注意事项：

- 游标使*解析*增量，而非*写入*——每次渲染重写整个漂流瓶并重新查询会话的观察。漂流瓶比记录小 10–20 倍，所以这是毫秒级；只是不要把游标归功于写入侧。
- "整体渲染" + "只解析新字节"只有在每会话保留