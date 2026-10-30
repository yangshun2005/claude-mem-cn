# Claude-Mem 13 Server Beta：独立的 BullMQ 观测运行时

状态：实现计划
日期：2026-05-07
发布目标：claude-mem 13 Server（beta）
与先前计划的关系：

- 扩展 `plans/2026-05-07-claude-mem-server-apache-bullmq-team-auth.md`。
- 取代 `plans/2026-05-07-claude-mem-13-server-beta-full-worker-parity.md` 中允许 Server beta 包装/复制 `WorkerService` 的 worker 对等部分。
- 保留现有 worker，但让 Server beta 成为完全独立的运行时，而非 worker 内部的门面。

## 执行决策

Server beta 必须端到端拥有自己的运行时：

```text
REST/MCP/hooks -> Server beta HTTP/API 层 -> BullMQ 观测任务 -> provider 生成 -> 服务器存储/搜索
```

Worker 仍是稳定的旧版运行时，但 Server beta 不得依赖 `WorkerService`、worker HTTP 路由、worker 队列消费者或 worker 进程生命周期来生成观测。

Server beta 应使用 BullMQ/Valkey 作为其规范队列，使用 Postgres 作为其规范观测存储。SQLite 仅保留为旧版 worker/本地兼容存储。Redis/Valkey 是任务、重试、并发和可观测性的运行时基础设施，而非观测的事实来源。

## 术语决策

Claude-mem 的领域对象是 **observation**（观测）。Server beta 必须在面向用户的 API、文档、任务、存储名称、测试、日志和实现计划中保留该措辞。

仅在 worker 时代代码中已存在的旧版兼容名称，或无法干净重命名的外部库/API 概念中，才使用 "memory"。新的 Server beta/Postgres 概念应围绕观测命名：

- `observations`，而非 `memory_items`
- `observation_sources`，而非 `memory_sources`
- `ObservationRepository`，而非 `MemoryItemsRepository`
- `GenerateObservationsForEventJob`，而非通用的 memory 生成
- `/v1/observations` 和以观测为中心的 MCP 工具作为规范接口

如果任何兼容端点仍使用 `/v1/memories`，应将其视为观测之上的别名，而非规范的 Server beta 模型。

## 阶段 0：文档调研

### 已阅读的本地来源

- `plans/2026-05-07-claude-mem-server-apache-bullmq-team-auth.md`
- `plans/2026-05-07-claude-mem-13-server-beta-full-worker-parity.md`
- `/Users/alexnewman/Downloads/claude-mem-handoff-docs/claude-mem-server-plan.md`
- `src/server/routes/v1/ServerV1Routes.ts`
- `src/server/queue/BullMqObservationQueueEngine.ts`
- `src/server/queue/ObservationQueueEngine.ts`
- `src/services/worker-service.ts`
- `src/services/worker/SessionManager.ts`
- `src/services/worker/agents/ResponseProcessor.ts`
- `src/services/worker/ClaudeProvider.ts`
- `src/services/worker/GeminiProvider.ts`
- `src/services/worker/OpenRouterProvider.ts`
- `src/services/worker/http/shared.ts`
- `src/storage/sqlite/agent-events.ts`
- `src/storage/sqlite/memory-items.ts`
- `src/core/schemas/agent-event.ts`
- `src/core/schemas/memory-item.ts`
- `scripts/e2e-server-beta-docker.sh`
- `docker/e2e/server-beta-e2e.mjs`

### 已阅读的外部文档

- BullMQ Workers：https://docs.bullmq.io/guide/workers
- BullMQ Worker 并发：https://docs.bullmq.io/guide/workers/concurrency
- BullMQ 停滞任务：https://docs.bullmq.io/guide/jobs/stalled
- Better Auth Express 集成：https://better-auth.com/docs/integrations/express

### 具体发现

- 当前 `/v1` 服务器路由在旧版 "memory" 路由/仓库名称下存储提供的事件和直接观测记录：
  - `src/server/routes/v1/ServerV1Routes.ts` 注册了 `POST /v1/events`、`POST /v1/events/batch` 和 `POST /v1/memories`。
  - 这些路由调用 `AgentEventsRepository.create(...)` 和 `MemoryItemsRepository.create(...)`。
  - 它们当前不会将 provider 生成任务入队。
- 当前 AI 观测生成路径由 worker 拥有：
  - `src/services/worker/SessionManager.ts` 通过 `getMessageIterator(...)` 消费排队消息。
  - `src/services/worker-service.ts` 通过 `startSessionProcessor(...)` 启动 provider 会话。
  - `src/services/worker/agents/ResponseProcessor.ts` 使用 `parseAgentXml(...)` 解析 provider XML，并通过 `sessionStore.storeObservations(...)` 写入观测。
- 现有的 v2 对等计划将 `Claude/Gemini/OpenRouter providers`、会话摄取路由、队列语义和 hook 路由列为对等要求，但未明确要求 `/v1/events` 生成观测。
- BullMQ 官方文档确立了 Server beta 应直接使用的原语：
  - `Worker` 处理任务，将成功的任务移至 completed，将抛出的任务移至 failed。
  - BullMQ worker 应附加 `error` 监听器。
  - Worker 支持 `autorun: false`。
  - Worker 通过 worker 选项对象支持并发。
  - 多个 worker 是提高可用性的推荐方式。
  - 当 worker 停止续订锁时，活动任务可能停滞并被重试。
- Better Auth Express 文档要求 auth handler 在 `express.json()` 之前挂载，并对 Express 5 使用 `/api/auth/*splat`。

### 允许的 API 和模式

- 从 `src/services/server/Server.ts` 和 Better Auth 文档复制 Express body 前置路由挂载。
- 从 `src/server/middleware/auth.ts` 和 `src/server/auth/api-key-service.ts` 复制 API-key 认证。
- 在有用处复制仓库行为，但针对 Postgres 实现 Server beta 仓库；不要将 worker 旧版 `SessionStore` 复用为服务器观测模型。
- 从 `src/services/worker/ClaudeProvider.ts`、`GeminiProvider.ts` 和 `OpenRouterProvider.ts` 复制 provider 请求构造，然后将共享逻辑移至 `src/server/generation` 或 `src/core/generation`。
- 从 `src/sdk/parser.ts` 复制 XML 解析，从 `src/services/worker/agents/ResponseProcessor.ts` 复制当前后处理规则。
- 直接使用 BullMQ `Queue`、`Worker` 和 `QueueEvents` 作为 Server beta 生成队列。
- 保留 `src/server/queue/redis-config.ts` 中的 Valkey/Redis 健康检查和现有 Docker E2E 设置。

### 反模式防护

- 不要让 Server beta 调用 `new WorkerService()`。
- 不要让 Server beta 依赖 worker HTTP 路由类进行生成。
- 不要让 `/v1` 成为只写事件归档，却声称 Server beta 生成观测。
- 不要将旧版 SQLite 待处理消息队列用于 Server beta 生成。
- 不要将规范观测记录存储在 Redis 中。
- 不要移除或破坏现有 worker 的稳定性。
- 不要从显式 Server beta BullMQ 模式静默回退到 SQLite。
- 不要在 `express.json()` 之后挂载 Better Auth。

## 目标架构

### 运行时分离

```text
src/services/worker-service.ts
  旧版 worker 运行时。稳定的兼容路径。以后可以导入共享核心组件。

src/server/runtime/ServerBetaService.ts
  独立的服务器运行时。拥有 HTTP 服务器、BullMQ 队列、provider 生成 worker、
  服务器存储仓库、认证、健康检查和 Docker 部署。
```

### Server Beta 流程

```text
POST /v1/events
POST /v1/events/batch
路由到 Server beta 的 Claude Code hook
MCP observation_record_* 工具
        |
        v
AgentEventsRepository 事务
        |
        v
ObservationGenerationJobRepository outbox 行
        |
        v
BullMQ Queue.add(...)
        |
        v
BullMQ Worker 处理器
        |
        v
ProviderObservationGenerator
        |
        v
parseAgentXml / 结构化解析器
        |
        v
ObservationRepository.create(...) + ObservationSourcesRepository.addSource(...)
        |
        v
QueueEvents/SSE/审计/搜索索引更新
```

## 阶段 1：Postgres 观测存储基础

### 需要实现的内容

- 添加 Server beta Postgres 配置：
  - 将包依赖 `pg` 和 `@types/pg` 添加到本仓库使用的 Node/Bun TypeScript 包清单；
  - 将 Postgres 存储代码集中到：
    - `src/storage/postgres/config.ts` 用于环境解析、连接池大小、超时和 SSL 设置；
    - `src/storage/postgres/pool.ts` 用于共享的 `pg.Pool` 工厂、健康检查、事务和优雅关闭；
    - `src/storage/postgres/schema.ts` 用于迁移/引导 SQL 和 schema 版本常量；
    - `src/storage/postgres/index.ts` 用于 Server beta 运行时装配使用的导出；
  - `CLAUDE_MEM_SERVER_DATABASE_URL`；
  - 连接池大小和超时设置；
  - 启动验证：当需要 Postgres 但不可用时使 Server beta 失败；
  - 优雅关闭：排空并关闭 Postgres 连接池。
- 为 Server beta 存储添加迁移/引导辅助程序：
  - 创建所需的 schema/表/索引；
  - 记录已应用的迁移版本；
  - 在启动和测试中可安全重复运行。
- 定义规范的 Postgres 表：
  - `teams`；
  - `projects`；
  - `team_members`；
  - `api_keys`；
  - `audit_log`；
  - `server_sessions`；
  - `agent_events`；
  - `observations`；
  - `observation_sources`；
  - `observation_generation_jobs`；
  - `observation_generation_job_events`。
- 在阶段 1 迁移中显式实现初始 schema 契约。仅当所有仓库契约和测试在同一阶段更新时，才可细化列名：

```sql
CREATE TABLE teams (
  id TEXT PRIMARY KEY,
  name TEXT NOT NULL,
  metadata JSONB NOT NULL DEFAULT '{}'::jsonb,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE projects (
  id TEXT PRIMARY KEY,
  team_id TEXT NOT NULL REFERENCES teams(id) ON DELETE CASCADE,
  name TEXT NOT NULL,
  metadata JSONB NOT NULL DEFAULT '{}'::jsonb,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE (id, team_id)
);

CREATE TABLE team_members (
  team_id TEXT NOT NULL REFERENCES teams(id) ON DELETE CASCADE,
  user_id TEXT NOT NULL,
  role TEXT NOT NULL,
  metadata JSONB NOT NULL DEFAULT '{}'::jsonb,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  PRIMARY KEY (team_id, user_id)
);

CREATE TABLE api_keys (
  id TEXT PRIMARY KEY,
  key_hash TEXT NOT NULL UNIQUE,
  team_id TEXT REFERENCES teams(id) ON DELETE CASCADE,
  project_id TEXT REFERENCES projects(id) ON DELETE CASCADE,
  actor_id TEXT NOT NULL,
  scopes JSONB NOT NULL DEFAULT '[]'::jsonb,
  revoked_at TIMESTAMPTZ,
  expires_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  CHECK (project_id IS NULL OR team_id IS NOT NULL),
  FOREIGN KEY (project_id, team_id) REFERENCES projects(id, team_id) ON DELETE CASCADE
);

CREATE TABLE audit_log (
  id TEXT PRIMARY KEY,
  team_id TEXT REFERENCES teams(id) ON DELETE SET NULL,
  project_id TEXT REFERENCES projects(id) ON DELETE SET NULL,
  actor_id TEXT,
  api_key_id TEXT REFERENCES api_keys(id) ON DELETE SET NULL,
  action TEXT NOT NULL,
  resource_type TEXT NOT NULL,
  resource_id TEXT,
  details JSONB NOT NULL DEFAULT '{}'::jsonb,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  CHECK (project_id IS NULL OR team_id IS NOT NULL),
  FOREIGN KEY (project_id, team_id) REFERENCES projects(id, team_id) ON DELETE SET NULL
);

CREATE TABLE server_sessions (
  id TEXT PRIMARY KEY,
  project_id TEXT NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
  team_id TEXT NOT NULL REFERENCES teams(id) ON DELETE CASCADE,
  external_session_id TEXT,
  content_session_id TEXT,
  agent_id TEXT,
  agent_type TEXT,
  platform_source TEXT,
  generation_status TEXT NOT NULL DEFAULT 'idle',
  metadata JSONB NOT NULL DEFAULT '{}'::jsonb,
  started_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  ended_at TIMESTAMPTZ,
  last_generated_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE (project_id, external_session_id),
  FOREIGN KEY (project_id, team_id) REFERENCES projects(id, team_id) ON DELETE CASCADE
);

CREATE TABLE agent_events (
  id TEXT PRIMARY KEY,
  project_id TEXT NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
  team_id TEXT NOT NULL REFERENCES teams(id) ON DELETE CASCADE,
  server_session_id TEXT REFERENCES server_sessions(id) ON DELETE SET NULL,
  source_adapter TEXT NOT NULL,
  source_event_id TEXT,
  idempotency_key TEXT NOT NULL,
  event_type TEXT NOT NULL,
  payload JSONB NOT NULL,
  metadata JSONB NOT NULL DEFAULT '{}'::jsonb,
  occurred_at TIMESTAMPTZ NOT NULL,
  received_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE (idempotency_key),
  UNIQUE (id, project_id, team_id),
  FOREIGN KEY (project_id, team_id) REFERENCES projects(id, team_id) ON DELETE CASCADE
);

CREATE TABLE observation_generation_jobs (
  id TEXT PRIMARY KEY,
  project_id TEXT NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
  team_id TEXT NOT NULL REFERENCES teams(id) ON DELETE CASCADE,
  agent_event_id TEXT REFERENCES agent_events(id) ON DELETE CASCADE,
  source_type TEXT NOT NULL CHECK (source_type IN ('agent_event', 'session_summary', 'observation_reindex')),
  source_id TEXT NOT NULL,
  server_session_id TEXT REFERENCES server_sessions(id) ON DELETE SET NULL,
  job_type TEXT NOT NULL,
  status TEXT NOT NULL CHECK (status IN ('queued', 'processing', 'completed', 'failed', 'cancelled')),
  idempotency_key TEXT NOT NULL UNIQUE,
  bullmq_job_id TEXT UNIQUE,
  attempts INTEGER NOT NULL DEFAULT 0,
  max_attempts INTEGER NOT NULL DEFAULT 3,
  next_attempt_at TIMESTAMPTZ,
  locked_at TIMESTAMPTZ,
  locked_by TEXT,
  completed_at TIMESTAMPTZ,
  failed_at TIMESTAMPTZ,
  cancelled_at TIMESTAMPTZ,
  last_error JSONB,
  payload JSONB NOT NULL DEFAULT '{}'::jsonb,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE (team_id, project_id, source_type, source_id, job_type),
  CHECK (
    (source_type = 'agent_event' AND agent_event_id IS NOT NULL AND source_id = agent_event_id)
    OR
    (source_type = 'session_summary' AND agent_event_id IS NULL AND server_session_id IS NOT NULL AND source_id = server_session_id)
    OR
    (source_type = 'observation_reindex' AND agent_event_id IS NULL)
  ),
  FOREIGN KEY (agent_event_id, project_id, team_id) REFERENCES agent_events(id, project_id, team_id) ON DELETE CASCADE,
  FOREIGN KEY (project_id, team_id) REFERENCES projects(id, team_id) ON DELETE CASCADE
);

CREATE TABLE observations (
  id TEXT PRIMARY KEY,
  project_id TEXT NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
  team_id TEXT NOT NULL REFERENCES teams(id) ON DELETE CASCADE,
  server_session_id TEXT REFERENCES server_sessions(id) ON DELETE SET NULL,
  kind TEXT NOT NULL DEFAULT 'observation',
  content TEXT NOT NULL,
  content_search TSVECTOR GENERATED ALWAYS AS (to_tsvector('english', content)) STORED,
  generation_key TEXT,
  metadata JSONB NOT NULL DEFAULT '{}'::jsonb,
  embedding JSONB,
  created_by_job_id TEXT REFERENCES observation_generation_jobs(id) ON DELETE SET NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE (team_id, project_id, generation_key),
  FOREIGN KEY (project_id, team_id) REFERENCES projects(id, team_id) ON DELETE CASCADE
);

CREATE TABLE observation_sources (
  id TEXT PRIMARY KEY,
  observation_id TEXT NOT NULL REFERENCES observations(id) ON DELETE CASCADE,
  agent_event_id TEXT REFERENCES agent_events(id) ON DELETE CASCADE,
  generation_job_id TEXT REFERENCES observation_generation_jobs(id) ON DELETE SET NULL,
  source_type TEXT NOT NULL CHECK (source_type IN ('agent_event', 'session_summary', 'observation_reindex', 'manual')),
  source_id TEXT NOT NULL,
  metadata JSONB NOT NULL DEFAULT '{}'::jsonb,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE (observation_id, source_type, source_id),
  UNIQUE (source_type, source_id, generation_job_id, observation_id),
  CHECK (
    (source_type = 'agent_event' AND agent_event_id IS NOT NULL AND source_id = agent_event_id)
    OR
    (source_type <> 'agent_event' AND agent_event_id IS NULL)
  )
);

CREATE TABLE observation_generation_job_events (
  id TEXT PRIMARY KEY,
  generation_job_id TEXT NOT NULL REFERENCES observation_generation_jobs(id) ON DELETE
```