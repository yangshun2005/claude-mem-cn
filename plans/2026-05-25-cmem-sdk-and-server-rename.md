# cmem-sdk：在服务器（Postgres）运行时上可嵌入的 claude-mem I/O + `server-beta` → `server` 重命名

状态：实施计划
日期：2026-05-25
发布目标：claude-mem 13.x — `claude-mem/sdk` 导出
与先前计划的关系：

- 基于 `plans/2026-05-07-server-beta-independent-bullmq-observation-runtime.md` 和 `plans/2026-05-07-claude-mem-server-apache-bullmq-team-auth.md`（Postgres“服务器”运行时）。
- 完成提交 `ae454cfc`（“feat: add SDK exports for consumer app integration”）中推迟的 SDK 导出槽位——该提交向 `package.json` 添加了 `exports["."]` 和 `exports["./sdk"]`，但从未添加 `src/index.ts`、从未添加 `src/sdk/index.ts`，也从未添加生成它们的构建步骤。
- 从服务器运行时中移除“beta”标签，因为字面字符串 `server-beta` 是静默运行时回归的根源（见阶段 1）。

## 执行决策

cmem-sdk **不是新系统**。它是现有的进程内 **服务器** 运行时（`src/server/*` + `src/storage/postgres/*` + 现有 `src/services/sync/*` Chroma 引擎）**以可导入库的形式暴露，并移除 HTTP/守护进程/Redis 外壳**。SDK 所需的一切都已存在，且其核心已无守护进程依赖；SDK 是组合 + 打包，外加一次谨慎的重命名。

```
consumer app
  └─ import { createCmemClient } from 'claude-mem/sdk'
       ├─ Postgres (pg)         ← 记录系统（捕获、观测、会话、任务）  [src/storage/postgres/*]
       ├─ in-process generation ← provider.generate() (fetch) → parseAgentXml → processGeneratedResponse  [src/server/generation/*]
       ├─ Chroma (REQUIRED)     ← 对同一批观测的语义索引，通过 uvx chroma-mcp 子进程  [src/services/sync/*]
       └─ search                ← Chroma 语义（主路径）。Postgres FTS 仅在 Chroma 瞬时失败时作为运行时安全网存在——它不是功能开关。  [src/storage/postgres/observations.ts]
```

**Chroma 不是可选的。** 没有语义搜索的 claude-mem 是坏的——观测无法以用户实际搜索的方式被搜索。SDK 必须在构造时初始化并验证 Chroma；如果 `uvx chroma-mcp` 无法启动，`createCmemClient(...)` 会拒绝。保留 Postgres FTS 路径仅为了镜像 `SearchManager.ts:255` 的运行时韧性模式（会话中途 Chroma 瞬时死亡）；使用时大声记录日志，且不暴露为用户可配置模式。

SDK **不得** 引入：Express、BullMQ、ioredis/Redis、better-auth、HTTP 路由、守护进程/pidfile 生命周期、worker 的 `bun:sqlite` 存储，或 Claude Code 子进程生成路径。这些都是可复用核心周围的 *外壳*。

要研究的接线枢纽是 `createServerBetaService()`（`src/server/runtime/create-server-beta-service.ts:156`）。它已经构建了 SDK 想要的确切对象图（连接池 → Schema 引导 → 仓库），然后附加 SDK 丢弃的部分（HTTP 服务、队列管理器、生成 worker）。SDK 复现的是 *图*，不是 *服务*。

## 术语决策（继承，强制执行）

- 领域对象是 **observation**，绝不是“memory”。保留 `observations`、`observation_sources`、`PostgresObservationRepository`、`/v1/observations`。`/v1/memories` 和 `memory_*` MCP 工具仅为别名。
- 运行时是 **`server`**（本计划），绝不是“server-beta”。`worker` 仍是旧版 SQLite 运行时。
- 公共客户端是 **`CmemClient`**，由 **`createCmemClient(...)`** 构造，从 **`claude-mem/sdk`** 导入。

---

## 阶段 0：文档发现

### 已阅读的本地来源（及各自确立的内容）

| 来源 | 确立内容 |
|---|---|
| `src/server/runtime/create-server-beta-service.ts` | 接线枢纽。图 = 连接池 + `bootstrapServerBetaPostgresSchema` + `createPostgresStorageRepositories`；队列管理器 **禁用**，除非 `CLAUDE_MEM_QUEUE_ENGINE=bullmq`（`:255-263`）；生成 worker **禁用**，除非队列激活 + provider 已配置（`:195-216`）。环境驱动的 provider 构建在 `:218-247`。 |
| `src/storage/postgres/index.ts` | `createPostgresStorageRepositories(client: PostgresQueryable)` `:39` 返回所有仓库。 |
| `src/storage/postgres/pool.ts` | `createPostgresPool(config)` `:13`、`getSharedPostgresPool({requireDatabaseUrl})` `:24`、`withPostgresTransaction(pool, fn)` `:45`。 |
| `src/storage/postgres/config.ts` | `parsePostgresConfig()` `:26`，读取 `CLAUDE_MEM_SERVER_DATABASE_URL`（唯一连接变量）+ 连接池/SSL 调优。 |
| `src/storage/postgres/schema.ts` | `bootstrapServerBetaPostgresSchema(client)` `:22`——幂等的进程内迁移运行器，无扩展，无 pgvector。`observations` 表 DDL `:212-227`：`content TEXT`、`content_search TSVECTOR GENERATED ALWAYS`、`embedding JSONB`（可空，**未使用**）、GIN 索引 `:274`。 |
| `src/storage/postgres/observations.ts` | `create(...)` `:72`、`search({projectId, teamId, query, limit})` `:153`（通过 `websearch_to_tsquery` + `ts_rank` 的 FTS）、`getByIdForScope` `:120`、`listByProject` `:133`。**无向量搜索。** `embedding` 仅在传入时写入，从不读取。 |
| `src/storage/postgres/agent-events.ts` | `PostgresAgentEventsRepository.create(input)` `:63`、`CreatePostgresAgentEventInput` `:31`。 |
| `src/storage/postgres/generation-jobs.ts` | `create(...)` `:101`、`transitionStatus({status})` `:164`（仅合法转换——`queued→processing→completed`；**`queued→completed` 非法**，`:390`）。 |
| `src/storage/postgres/server-sessions.ts` / `projects.ts` / `teams.ts` | 会话/项目/团队仓库。项目有 `create({teamId,name})` `:27` + `getByIdForTeam(id,teamId)` `:46`——**无按名称查找**（租户引导影响，见阶段 3）。 |
| `src/server/services/IngestEventsService.ts` | `ingestOne(input,{generate})` `:96` 在一个事务中写入 agent_event + generation_job 发件箱；`resolveEventQueue: () => null`（`IngestEventsServiceOptions` `:64`）使 BullMQ 入队成为空操作（`enqueueState='queued_only'`）。 |
| `src/server/generation/providers/shared/types.ts` | `ServerGenerationProvider.generate(context, signal?)` `:30`；`ServerGenerationContext` `:9` = `{job, events, project:{projectId,teamId,serverSessionId,projectName}}`；`ServerGenerationResult` `:23` = `{rawText, modelId?, providerLabel, tokensUsed?}`。 |
| `src/server/generation/providers/ClaudeObservationProvider.ts` | `constructor({apiKey, model?, maxOutputTokens?, fetchImpl?})` `:32`；普通 `fetch` 到 `https://api.anthropic.com/v1/messages` `:69`。**无 `@anthropic-ai/claude-agent-sdk`，无子进程。** 默认模型 `claude-3-5-sonnet-latest` `:17`。（Gemini/OpenRouter 同类：形状相同。） |
| `src/server/generation/processGeneratedResponse.ts` | `processGeneratedResponse({pool, job, rawText, modelId, providerLabel, ...})` `:62` 和 `processSessionSummaryResponse` `:332`——包装 `withPostgresTransaction`，通过 `parseAgentXml` 解析，写入 `observations` + `observation_sources`，完成任务。**从不写入 `embedding`。** |
| `src/server/generation/ProviderObservationGenerator.ts` | 可复用的内联核心是 `:196-254`（加载事件 → 加载项目 → `provider.generate` → `processGeneratedResponse`）；`:72-194` 是要 **跳过** 的 BullMQ 仪式。 |
| `src/sdk/parser.ts` | `parseAgentXml(raw, correlationId?)` `:41`。**在 `:105` 调用 `ModeManager.getInstance().getActiveMode()` 且无回退**——SDK 必须初始化一个模式。 |
| `src/sdk/prompts.ts` | `buildObservationPrompt`/`buildSummaryPrompt`/`buildInitPrompt`/`buildContinuationPrompt`（模式驱动）。 |
| `src/server/generation/providers/shared/prompt-builder.ts` | `buildServerGenerationPrompt(context)` `:42`——有 `loadActiveModeOrFallback()` `:46`（优雅），与 `parser.ts` 不同。 |
| `src/services/sync/ChromaSync.ts` | `constructor(project)` `:69`，集合 `cm__<project>` `:74`；文档层 `addDocuments(ChromaDocument[])` `:234`（id/document/metadata → `chroma_add_documents`）是 **存储无关的**；`syncObservation(observationId:number, …, obs:ParsedObservation, …)` `:306` 是 **SQLite 形状的**（整数 id、`StoredObservation`）；`queryChroma(...)` `:855`；`close()` `:1096`。 |
| `src/services/sync/ChromaMcpManager.ts` | 单例 `getInstance()` `:56`；生成 `uvx chroma-mcp` 子进程；`callTool(name, args)`。本地 all-MiniLM 嵌入，**无 API key**。 |
| `src/services/worker/DatabaseManager.ts` | 参考组合：`new ChromaSync('claude-mem')` `:26`。worker 读取 `CLAUDE_MEM_CHROMA_ENABLED !== 'false'` 环境门；**SDK 故意不遵守该门**——Chroma 是必需的（见执行决策）。该门是 worker 的脚枪，不应传播。 |
| `src/services/worker/SearchManager.ts` | 参考搜索语义：`search()` `:140` 做 Chroma 语义，**Chroma 失败时回退 FTS**（`:255`）。SDK 针对 Postgres 镜像此分支逻辑。 |
| `src/services/hooks/runtime-selector.ts` | **回归源。** `selectRuntime()` `:35` 要求 `CLAUDE_MEM_RUNTIME === 'server-beta'` 精确匹配；否则静默回退 worker（`:71-78`）。设置键 `CLAUDE_MEM_SERVER_BETA_{URL,API_KEY,PROJECT_ID}` `:41-43`。 |
| `src/shared/SettingsDefaultsManager.ts` | 键 `CLAUDE_MEM_SERVER_BETA_*` `:76-78`，默认值 `:151-154`；`CLAUDE_MEM_RUNTIME` 默认 `'worker'` `:151`。 |
| `src/services/worker-service.ts` | 分发 `runServerBetaServiceCli` `:850`，查找 `server-beta-service.cjs` `:851`；`server <cmd>` 子命令 `:1040`。 |
| `scripts/build-hooks.js` | 构建目标 `name:'server-beta-service'` `:16`；仅生成 `dist/npx-cli` + `dist/opencode-plugin`——**从不生成 `dist/index.js` 或 `dist/sdk/`**。在 `:262` 有 `bun:sqlite` 导入守卫先例。 |
| `package.json` | `exports["."]` → `dist/index.js` 和 `exports["./sdk"]` → `dist/sdk/index.js`（两者当前都解析到不存在的文件）。`pg` 是生产依赖且为纯 Node。 |

### 允许的 API（逐字签名——不要发明或扩展）

**连接 / 启动**
- `parsePostgresConfig(options?): PostgresConfig | null` — `config.ts:26`
- `createPostgresPool(config: PostgresConfig): PostgresPool` — `pool.ts:13`
- `getSharedPostgresPool(options?: { requireDatabaseUrl?: boolean }): PostgresPool` — `pool.ts:24`
- `bootstrapServerBetaPostgresSchema(client: PostgresQueryable): Promise<void>` — `schema.ts:22`（阶段 1 重命名为 → `bootstrapServerPostgresSchema`）
- `createPostgresStorageRepositories(client: PostgresQueryable): PostgresStorageRepositories` — `index.ts:39`
- `withPostgresTransaction<T>(pool, fn): Promise<T>` — `pool.ts:45`
- `PostgresQueryable = { query(text, values?) }` — `utils.ts:9`（`pg.Pool` 或 `pg.PoolClient` 满足）

**捕获**
- `new IngestEventsService({ pool, resolveEventQueue: () => null })` 然后 `ingestOne(input, { generate })` — `IngestEventsService.ts:93,96`
- `PostgresAgentEventsRepository.create(input: CreatePostgresAgentEventInput)` — `agent-events.ts:63`

**生成（内联，无 BullMQ）**
- `new ClaudeObservationProvider({ apiKey, model? })`（或 Gemini/OpenRouter）— `ClaudeObservationProvider.ts:32`
- `PostgresObservationGenerationJobRepository.transitionStatus({ id, projectId, teamId, status:'processing', lockedBy })` — `generation-jobs.ts:164`
- `provider.generate({ job, events, project }, signal?)` — `providers/shared/types.ts:30`
- `processGeneratedResponse({ pool, job, rawText, modelId, providerLabel, ... })` — `processGeneratedResponse.ts:62`
- `processSessionSummaryResponse({ ... })` — `processGeneratedResponse.ts:332`
- `parseAgentXml(raw, correlationId?)` — `parser.ts:41`（需要活动的 `ModeManager` 模式）

**搜索**
- `PostgresObservationRepository.search({ projectId, teamId, query, limit? })` — `observations.ts:153`
- `PostgresObservationRepository.getByIdForScope({ id, projectId, teamId })` / `listByProject(...)` — `observations.ts:120,133`

**Chroma（必需的语义；复用，不要分叉）**
- `ChromaMcpManager.getInstance()` + `callTool('chroma_add_documents' | 'chroma_query_documents' | 'chroma_create_collection' | 'chroma_delete_documents', args)` — `ChromaMcpManager.ts:56`
- `new ChromaSync(project)` + `queryChroma(...)` `:855` + `close()` `:1096`；`ChromaDocument { id, document, metadata }` + `addDocuments` `:234` 层是可复用、存储无关的接缝。

### 需防范的反模式（本计划存在是因为这些已经咬过我们）

1. **不要“构建混合体”、“适配”、“迁移”或“分叉”任何东西。** 每个引擎都已存在。SDK 是胶水 + 打包。如果任务描述包含“新系统”或“重新实现”，那是错的。
2. **不要添加 pgvector / `vector` 列 / 嵌入 API 调用。** Postgres 语义搜索不存在且超出范围——语义搜索由现有 Chroma 引擎交付（按明确指示）。FTS 是 Postgres 侧搜索。
3. **不要把 Express、BullMQ、ioredis、better-auth、React 或 `bun:sqlite` 拉入 SDK 包。** 服务器生成 provider 使用普通 `fetch`（无 `@anthropic-ai/claude-agent-sdk`）。用构建时导入守卫强制执行（阶段 2/9）。
4. **不要调用 `transitionStatus(queued → completed)`**——它会抛错（`generation-jobs.ts:390`）。必须先转换 `queued → processing`（镜像 `lockOutbox`）。
5. **不要在无活动模式时调用 `parseAgentXml`**——`parser.ts:105` 否则会抛错。在阶段 5 初始化 `ModeManager`（或使用 `prompt-builder` 回退语义）。
6. **不要盲目字符串替换重命名。** 持久化值（DB 表 `server_beta_schema_migrations`、`job_type`/`source_type` 枚举字符串、用户的 settings.json 键、`CLAUDE_MEM_RUNTIME` 值）需要向后兼容。只有代码标识符可自由重命名。
7. **不要重新运行仅 grep 的子代理并跨片段综合。** 整体阅读接线枢纽和组合根。

---

## 阶段 1：重命名 `server-beta` → `server`（基础 + 回归修复）

**为何优先：** SDK 构建在服务器运行时之上；以干净命名发布。可独立发布——单独修复静默回退回归。

### 1a. 修复回归（最高价值，最小改动）

要实现的内容：
- `src/services/hooks/runtime-selector.ts:34-37`：接受 **`'server'`** 作为规范运行时值，并 **仍接受 `'server-beta'`