# 活性优先于截止时间

**日期：** 2026-10-03 · **分支：** `feat/liveness-over-deadlines`（基于 `origin/main` a1951f2ad）

## 原因

claude-mem 有约 238 个猜测出来的截止时间（`setTimeout` / `AbortSignal.timeout` / `*_TIMEOUT`）。单个截止时间同时衡量两件不同的事——*对方是否还活着？* 和 *这项工作应该花多长时间？*——所以每个值都是猜测：太短会杀掉真实工作（并重新计费），太长会在死掉的对方上阻塞。修复方案借鉴 xAI SDK 的“secret stream”，把两者分开：

- **活性**来自信号（响应头、心跳字节、进度事件、磁盘上的文件）。
- **截止时间**变成*空闲*超时（“N 秒内没有生命迹象”），或者因为没人等待而消失。

第二个驱动因素（xAI 报告，“绝不重复付费”）：只有在已知工作没有发生时，才重试付费请求。claude-mem 目前在至少 7 处重发付费 LLM 调用（下文 D1–D10）。

第三个驱动因素（用户 2026-10-03 决定）：**云同步全部在 Supabase 上重建**——操作日志、内容、pgvector、Realtime、认证。Turbopuffer、Neon 同步日志、Fly `sync-api` 和 cmem.ai projector 跳点退役。“投影”不再存在：一次推送在一个事务中写入操作和可搜索行。

## 执行顺序

| 轨道 | 阶段 | 交付形式 |
|---|---|---|
| A — 本地活性 + 绝不重复付费 | 1–7 | claude-mem PR #1 |
| B — Supabase 云同步 | 8–12 | Supabase 迁移 + Edge Function、claude-mem PR #2、Pro PR |
| 最终 | 13 | 验证、`/babysit`、`/version-bump` |

轨道 A 和轨道 B 相互独立；先运行 A（更小，完全在仓库内）。

---

## 阶段 0 — 文档发现（已完成，已整合）

### 允许的 API / 事实（使用前引用）

**运行时。** Hook 和 worker 运行在 **Bun** 下（`plugin/scripts/bun-runner.js`）；MCP server 和 npx-cli 运行在 **Node** 下（`plugin/.mcp.json`、`scripts/build-hooks.js:689-695`）。打包产物是 esbuild `platform:'node'`。只使用 WHATWG 流 API（`response.body.getReader()`、`AbortController`、`AbortSignal.any`）——不要用 Node 流。

**Worker HTTP。** Bun 下基于 `node:http` 的 Express 5（`src/services/server/Server.ts:190-213`）。任何地方都没有设置服务器超时。现有 SSE：`ViewerRoutes.ts:157` 路由 `/stream`，处理器 `handleSSEStream`（`ViewerRoutes.ts:220-262`），广播器 `src/services/worker/SSEBroadcaster.ts`（目前没有心跳）。初始化门禁允许列表：`worker-service.ts:441-464`（`/health`、`/readiness`、`/version`、……）。

**Worker 初始化。** `WorkerService.start()`（`worker-service.ts:519`）→ 即发即忘的 `initializeBackground()`（`:607-860`）。唯一信号：`initializationCompleteFlag` + `resolveInitialization()`（`:757-759`）。失败路径在 `:857-859` 只记录日志——标志永远保持 false（这就是“卡死”状态）。Chroma 预热在就绪*之后*运行（`:828`），所以 `worker-utils.ts:65-66` 处归咎于预热的注释已过时。

**Hook→worker。** `executeWithWorkerFallback`（`src/shared/worker-utils.ts:1611-1709`）、`workerHttpRequest`（`:341-366`）、`fetchWithTimeout`（`:161-175`，把 TimeoutError 改写为 `"Request timed out after ${ms}ms"`——被 `server-client.ts:419`、`worker-utils.ts:1441`、`npx-cli/utils/prune-cache.ts:218` 的正则匹配；**保留该措辞**）。就绪：`waitForWorkerReadiness`（`:588`）、`ensureWorkerRunning`（`:753`，卡死分支 `:821-846`）、`ensureWorkerReadyWithin`（`:1152`）。`WEDGED_WORKER_UPTIME_S` 来自 `hook-constants.ts:27-38`。

**写 Hook（可暂存）：** observation（`POST /api/sessions/observations`）、file-edit（同一路由）、summarize（`POST /api/sessions/summarize`）、advisor-calls（`POST /api/advisor-calls`）、session-end（`POST /api/sessions/session-end`）。都不使用响应体。**不可暂存：** session-init（使用 `sessionDbId`，私有跳过）、user-message（一次读取）、context/file-context/semantic（读取）。

**Worker 摄取函数：** `ingestObservation(payload)` 从 `src/services/worker/http/shared.ts:118-275` 导出（已被 `transcripts/processor.ts:411` 直接调用）。Summarize/session-end 逻辑在 `SessionRoutes.ts` 中是私有的（`handleSummarizeByClaudeId :700-767`、`handleSessionEnd :769-782`）——必须提取。Advisor：`AdvisorRoutes.handleIngestAdvisorCalls`（`AdvisorRoutes.ts:73`）→ `store.recordAdvisorCall`。

**要复制的暂存先例：** `src/shared/deferred-session-end.ts`（`DeferredSessionEndQueue`：通过 `writeJsonFileAtomic` 每条目一个文件，`drain(accept)` 删除已接受的）及其 worker 排空 `drainDeferredSessionEndQueue` / `startDeferredSessionEndReplay`（`worker-service.ts:377-410`，启动时排空在 `:655-656`）。数据目录：`resolveDataDir()`（`src/shared/paths.ts:21-42`）——在使用时调用，不要在导入时调用。原子写入：`writeJsonFileAtomic`（`src/shared/atomic-json.ts`）。

**顺序约束。** `SessionMessageBuffer` 是内存中的，持久重放被有意移除（重复 observation，`SessionMessageBuffer.ts:20-40`）。暂存必须按确定性键去重，不能盲目重放。如果 init 尚未落地，summarize 返回 `unknown_session` → 排空必须保留会话未知的条目（如 `drainDeferredSessionEndQueue` 所做：返回 false = 保留）。

**上下文渲染。** Hook：`src/cli/handlers/context.ts:91-141`。Worker：`SearchRoutes.handleContextInject`（`SearchRoutes.ts:302-418`）→ `generateContextWithStats`（`ContextBuilder.ts:551-577`）。易变部分：头部时间戳（`formatHeaderDateTime`，`timeline-formatting.ts:104`）、工作状态“N 前更新”（`WorkStateRenderer.ts:58-60`）、observer/sync 健康横幅（`ContextBuilder.ts:240-268`，在预算适配内测量 `:363-409`）、每次注入时的 `syncClient.pullOnce({timeoutMs:1500})`（`SearchRoutes.ts:375-380`）。要失效的写入点：`SessionStore.storeObservations :3650`、`storeSummary :3600`、`appendWorkStateEntry :2989`、`importObservation :4319`、`importSessionSummary :4256`、`DataRoutes` 删除/合并、`SyncApply.apply*`（`SyncApply.ts:898-1274`）。项目键：`getProjectContext(cwd).allProjects`（`src/utils/project-name.ts:358`），主项目 = 最后一个元素。

**LLM 重试。** `withRetry`（`src/services/worker/retry.ts`）：`maxRetries:2`（`:156-160`），未分类 ⇒ 瞬态（`:164-167`），每次尝试截止时间 `DEFAULT_LLM_TIMEOUT_MS=180_000`（`SettingsDefaultsManager.ts:66`，`retry.ts:203-233`），`retryAfterMs` 无上限（`:245-246`），100ms 429 基数（`:173-176`）。分类器：`OpenRouterProvider.ts:267-304`、`OpenAICompatProvider.ts:298-319`、`GeminiProvider.ts:121-132`、`CodexProvider.ts:95-200`、服务器 `providers/shared/error-classification.ts:118-169`。截止时间 → `transport:deadline_exceeded`（`OpenAICompatibleProvider.ts:687-690`）→ `scheduleTransportResume`（`GeneratorExitHandler.ts:90-104`），在网关外无限制恢复（`SessionManager.ts:29-37`）。`x-claude-mem-prior-request-id` 在 `OpenRouterProvider.ts:756`、`GeminiProvider.ts:382` 发送，仓库中无人读取。所有 HTTP provider 强制 `stream:false`（`OpenRouterProvider.ts:479`）；SSE 解析已存在于 `OpenRouterProvider.ts:476` 附近。

**语料库。** `callWorker`（`src/servers/mcp-server.ts:82-130`，默认 30s）→ `CorpusRoutes.ts:97-231`（末尾单个 `res.json()`，没有 `res.on('close')`）。`KnowledgeAgent.isSessionResumeError`（`KnowledgeAgent.ts:124-127`）正则 `/session|resume|expired|invalid.*session|not found/i` → 几乎任何错误都会重新初始化。

### 反模式（适用于每个阶段）
- **不要**发明 Express/Bun API；SSE = `res.setHeader` + `res.write('event: …\ndata: …\n\n')`，与 `handleSSEStream` 完全一样。
- **不要**重新引入 `SessionMessageBuffer` 的持久重放；暂存条目按键去重。
- **不要**添加吞掉错误的 try/catch；每个捕获的错误都带上下文记录，并重新抛出或转为显式状态。
- **不要**把 `x-client-request-id`/尝试 id 当作服务器端幂等性——它们仅用于追踪。
- **不要**在任何响应字节到达后重试付费 POST。
- **不要**添加新的固定截止时间来替换旧的；使用空闲超时 + 活性信号，只在宿主强制要求的地方保留一个绝对安全上限（Claude Code hook 上限）。

---

## 阶段 1 — 绝不重复付费：重试规则 + 每批一个预算

**内容。** 将 `src/services/worker/retry.ts` 中的重试决策重写为 xAI 规则表（报告第 7 部分）：

| 结果 | 循环内重试？ |
|---|---|
| 429 / 我们自己的发送前拒绝（工作前被拒绝） | 是，Retry-After 上限 60 秒；无 Retry-After ⇒ 1 秒→30 秒退避；抖动 ×(0.5–1) |
| 任何响应前的网络错误、非流式 POST 上的 5xx（模糊） | **否**——返回类型化的 `ambiguous` 错误；由会话传输暂停决定，计入批次预算 |
| 收到响应后正文读取/解析失败、200 带内嵌错误（litellm） | **绝不**——输出失败 |
| 未分类 | **否**（翻转 `retry.ts:164-167`） |

- 为每个认领的批次添加 `PaidSendBudget`（默认总共 2 次付费发送），由 `withRetry`、`scheduleTransportResume`（`GeneratorExitHandler.ts:90-104`）、停滞恢复（`response-pacer.ts`）和 Codex `maxRetries` 消耗。耗尽 ⇒ 以显式记录状态停放该批次（不静默丢弃）。
- Codex：“完成但没有最终 agent 消息”和完成轮次后格式错误的结构化输出 ⇒ 不可重试（`CodexProvider.ts:118-122`）。
- 删除 `x-claude-mem-prior-request-id` 头和 `retry.ts:8-9` 的去重声明；添加每批 `clientAttemptId`（UUID），作为 `x-client-request-id` 发送，随用量/错误记录，携带在 `ClassifiedProviderError`（`provider-errors.ts:43,59`）上，并由 `describeProviderError`（`:118-120`）显示。
- 错误正文读取上限 64 KiB（`OpenRouterProvider.ts:814` 及同类）。

**文档。** xAI 报告第 7 部分表；`retry.ts` 整个文件；`provider-errors.ts`。
**验证。** 新增 `tests/worker/retry-never-pay-twice.test.ts`：表中每行一个用例，断言确切的 fetch 调用次数；预算在 withRetry + 传输恢复间共享；`grep -rn "prior-request-id" src` ⇒ 0。现有 provider 测试通过。
**防护。** 观察到 `response.ok` 后不重试。除预算大小外不新增设置键。

## 阶段 2 — 共享 SSE 读取器 + 空闲超时 fetch + 测试工具

**内容。**
- `src/shared/sse-reader.ts`：约 100 行，xAI 第 5 部分步骤 4 规则——CR/LF/CRLF，`:` 注释行算作活性，否则忽略，每事件上限 1 MiB，产出 `{event, data}`；流结束无终止事件 ⇒ 抛出 `StreamEndedEarlyError`。
- `fetchWithIdleTimeout(url, init, { idleTimeoutMs, absoluteCapMs })` 放在 `fetchWithTimeout`（`worker-utils.ts:161`）旁边：计时器在响应头前启动，在响应头和每个正文块时重置；拥有正文读取并返回文本或块的异步可迭代对象（绝不是裸 `Response`，因为空闲计时器必须覆盖正文）。到期时抛出 `Error("Request timed out after ${idleTimeoutMs}ms idle")`——保留“timed out”措辞以匹配现有正则。
- `workerHttpRequest` 增加 `idleTimeoutMs` 选项。
- 测试工具 `tests/helpers/stream-fetch-mock.ts`：尊重中止的 mock fetch，脚本化流（响应头前挂起、正文中途挂起、首字节前/后失败、`:` ping）；假计时器断言助手“N 次轮询后零活跃计时器”（xAI 第 18 部分）。

**验证。** `tests/shared/sse-reader.test.ts`、`tests/shared/fetch-idle-timeout.test.ts`（ping 使连接在空闲窗口后保持存活；静默触发；绝对上限触发）。类型检查干净。
**防护。** 不导入 Node `stream`。不改变 `fetchWithTimeout` 语义。

## 阶段 3 — Provider “secret stream”（移除 180 秒放弃并重发）

**内容。** OpenRouter + OpenAI 兼容 provider 内部发送 `stream: true`（`OpenRouterProvider.ts:479` 和 OpenAICompat 请求构建器），通过阶段 2 的 `sse-reader` + `fetchWithIdleTimeout` 读取（默认空闲 90 秒，绝对上限 = 现有 `MAX_LLM_TIMEOUT_MS` 300 秒，`retry.ts:65`），组装最终文本，从最终块获取 `usage`（请求 `stream_options: {include_usage: true}`）。保持公共 provider 返回形状不变。`retryBeforeOutput`：仅当流在第一个内容增量前失败时允许一次重试，计入阶段 1 预算。Gemini：此阶段保持非流式（不同端点），但获得阶段 1 规则。移除流式 provider 的 `transport:deadline_exceeded` 重发：空闲到期现在是阶段 1 下的 `ambiguous` 结果。

**文档。** `OpenRouterProvider.ts:476` 附近的现有 SSE 处理；xAI 第 6 部分。
**验证。** 使用工具的 provider 测试：慢但活着的流（ping 400 秒）成功；静默 91 秒触发一次且不在循环内重发；捕获用量。在这些 provider 中 grep `stream: false` ⇒ 0。
**防护。** 不改变提示内容或组装文本的解析。

## 阶段 4 — `GET /api/ready` 进度流；删除卡死启发式

**内容。**
- `WorkerService` 获得 `initPhase` 状态：`starting → db_ready → routes_ready → ready | failed{message}`。在 `initializeBackground()` 的现有步骤设置（`worker-service.ts:649` db、`:727-755` 路由、`:757-759` 就绪）。`:857-859` 的 catch 除记录外设置 `failed`（显式状态）。
- `Server.ts` `setupCoreRoutes`：`GET /api/ready` SSE（复制 `handleSSEStream` 头）。立即发出当前阶段、每次转换、每 5 秒 `: ping` 注释，并在 `ready` 或 `failed` 后**结束**。将 `/ready` 添加到初始化门禁允许列表（`worker-service.ts:441-464`）。`/api/health` 和 `/api/readiness` 对旧客户端保持不变。
- Hook/CLI：用一次 `/api/ready` 读取（使用 `fetchWithIdleTimeout`，空闲 5 秒，绝对上限 = hook 现有预算）替换 `waitForWorkerReadiness` 轮询（`worker-utils.ts:588`）和 `ensureWorkerReadyWithin` 探测循环（`:1152`）。`failed` ⇒ 立即回收；空闲到期 ⇒ 卡死 ⇒ 回收。删除基于运行时间的卡死分支（