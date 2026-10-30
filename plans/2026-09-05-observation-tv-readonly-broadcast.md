# Observation TV — 只读广播 + 共享密钥令牌

**日期：** 2026-09-05
**工作树 / 分支：** `.claude/worktrees/observation-tv` / `worktree-observation-tv`
**基于：** 提交 `8489f78b` — *feat(ui): observation TV — fullscreen fading titles off the existing SSE stream*
**实时状态文档：** `/workspace/obs-broadcast/STATUS.md`
**与以下无关：** Pepper / Booth X Live。本计划不涉及它们。

---

## 首要目标

**Alex 可以在局域网内的第二台设备上打开 Observation TV —— 手机、iPad、备用显示器 —— 该设备能看到
observation 标题流式滚动，并且对 worker 做不了任何其他事。** 不能重启它，不能读取
`~/.claude-mem/settings.json`，不能删除 observation，不能导入行，不能切换 MCP。

以下所有内容都以这句话为衡量标准。如果一个任务不能把请求从"整个 worker API 都可访问"推进到
"恰好四条只读路径可访问，且必须携带密钥"，那它就不属于本计划。

**已锁定决策（Prioritizer，2026-09-05）：**
- 目前仅限本地。本切片不通过互联网暴露手机访问。
- 构建**方案 (b)**：只读广播面 + 共享密钥令牌。
- **拒绝在未修改的 worker 前面加 cloudflared / 任何隧道。** 它会把
  `POST /api/admin/restart`、`POST /api/settings` 和 `GET /api/settings`（会以明文返回
  用户的 Gemini 和 OpenRouter API 密钥）发布到开放互联网。不要在本计划的各阶段、注释或文档中
  再次提出。

---

## 一段话说明问题

worker 的整个 HTTP 面 —— **13 个路由类中的 45+ 条路由** —— *没有任何请求认证*。它唯一的防御是
回环绑定。代码库自己就明说了：`src/server/runtime/ServerService.ts:129-131` 为 server 运行时
开启加固头，注释写道 *"server runtime is reachable over the network in Docker, so it emits hardening headers
(the worker, loopback-only, does not)."* `requireLocalhost`（`src/services/worker/http/middleware.ts:64-86`）
只保护三条路由：`/api/admin/restart`、`/api/admin/shutdown`、`/api/admin/doctor`
（`src/services/server/Server.ts:291,305,326`）。其余一切 —— `POST /api/settings`、
`GET /api/settings`、`DELETE /api/observation/:id`、`POST /api/import`、`POST /api/logs/clear`、
`DELETE /api/corpus/:name`、`ALL /api/auth/*splat` —— 对任何能到达该端口的东西完全开放。
所以今天把 `CLAUDE_MEM_WORKER_HOST=0.0.0.0` 翻过来并不是"把 TV 放到局域网上"，而是把
**整个记忆数据库、设置写入器和 provider API 密钥**放到局域网上。

**修复形态：** 一个守卫中间件，最先挂载，对回环请求是*空操作*，对非回环请求是*默认拒绝，仅四路径
白名单，且需共享密钥*。

---

## 约束条件

1. **白名单，绝不用黑名单。** 路由数量每个版本都在增长。黑名单是某人记得的路由列表；下一个 PR
   加上第 46 条路由，它默认就可被远程读取。
2. **不新增依赖。** 不用 `helmet`，不用 `express-rate-limit`，不用 `passport`，不用 `jsonwebtoken`。
   仓库有明确政策反对这类做法（`Server.ts:96-105`：helmet 被拒绝，改为手写头以保持 esbuild
   打包产物不变）。
3. **默认行为与今天逐字节一致。** 令牌未设置 ⇒ 守卫不挂载 ⇒ 对任何现有安装都没有变化，包括
   文档中记录的 Docker `0.0.0.0` 配置（`docs/docker.md:12`）。
4. **`CLAUDE_MEM_WORKER_HOST` 默认值保持 `127.0.0.1`。** 本计划从不改变它。它记录如何改变它，
   以及改变后令牌起什么作用。
5. 差异与缺陷规模相称。不要重构 `Server.ts`，不要重组路由类。
6. 不要编辑 `CHANGELOG.md`（自动生成）。

---

## 阶段 0 —— 合并式发现（请阅读；不要重新推导）

2026-09-05 通过直接读取文件，针对 `worktree-observation-tv` @ `8489f78b` 验证。以下每条断言
都带有 `file:line`。如果后续阶段的指令与此处内容不一致，停下来重新读文件 —— 不要猜。

### 0.1 express 应用如何组装

`src/services/server/Server.ts:119-134` —— 构造函数，按精确顺序：

```ts
127: this.app = express();
128: this.app.disable('x-powered-by');
129: this.setupSecurityHeaders();   // Server.ts:198-206 — opt-in via options.securityHeaders
130: this.setupCors();              // Server.ts:208-210
131: this.setupPreBodyParserRoutes(); // Server.ts:212-214 — mounts ALL /api/auth/*splat
132: this.setupMiddleware();        // Server.ts:193-196 — json parser, logger, express.static
133: this.setupCoreRoutes();        // Server.ts:216-366 — mounts /api/admin/*, /api/health, ...
```

然后，**在构造之后**，`src/services/worker-service.ts:299` 调用 `registerRoutes()`
（`worker-service.ts:314-366`），挂载 Chroma → 初始化门 → Viewer → Session → Data →
Settings → Logs → Memory → ServerV1。还有三个类稍后在 `initializeBackground()` 内挂载：
SearchRoutes（`worker-service.ts:564`）、CorpusRoutes（`:573`）、CloudSyncRoutes（`:580`）。

**决定设计的后果：** 唯一能覆盖 `/api/auth/*`（构造函数步骤 131 挂载）、`/api/admin/*`（步骤 133）、
静态文件挂载（步骤 132）*以及*之后添加的每条路由 —— 包括尚不存在的路由 —— 的位置是
**位置零，即构造函数最顶部。** 之后挂载的任何东西都会留下更早的路由未覆盖。

### 0.2 我们将复制的现有内容

| 内容 | 位置 | 此处为何重要 |
|---|---|---|
| `requireLocalhost(req,res,next)` | `src/services/worker/http/middleware.ts:64-86` | 本仓库 IP 检查的惯用写法：读取 `req.ip \|\| req.connection.remoteAddress`，与 `127.0.0.1` / `::1` / `::ffff:127.0.0.1` / `localhost` 比较，记录 `logger.warn('SECURITY', ...)`，返回 `403` JSON。**新守卫照此文件的形态复制。** |
| `isLocalhost(req)` | `src/server/middleware/request-auth-helpers.ts:15-21` | 相同的检查，已提取为纯辅助函数，已可单元测试。优先导入它而非重新实现。 |
| `parseBearerToken(header)` | `src/server/middleware/request-auth-helpers.ts:10-13` | `/^Bearer\s+(.+)$/i` → 去除首尾空白的令牌或 `null`。可直接复制。 |
| `hasForwardedClientHeaders(req)` | `src/server/middleware/request-auth-helpers.ts:44-51` | 检测 `forwarded` / `x-forwarded-for` / `x-forwarded-host` / `x-real-ip`。现有认证用它拒绝经由代理传来的"回环"声明。 |
| 头优先级 `Bearer` → `X-Api-Key` | `src/server/middleware/auth.ts:40-46` | 既定顺序和确切的 401 消息措辞。 |
| `safeEqualHex(a,b)` | `src/server/auth/sqlite-api-key-service.ts:83-97` | 先长度预检，再 `crypto.timingSafeEqual`，输入格式错误时 `logger.warn`。**逐字复制此形态**用于令牌比较。 |
| `createRawServerApiKey()` | `src/server/auth/sqlite-api-key-service.ts:155-157` | `` `cmem_${randomBytes(32).toString('base64url')}` `` —— 本仓库生成密钥的惯用写法。 |
| 可选 `ServerOptions` 标志 | `src/services/server/Server.ts:88-93`（`securityHeaders?: boolean`）+ `:198-206` | "worker 选择启用、另一运行时不用"的 Server 能力先例。**新选项完全照此建模。** |
| "功能开启当且仅当令牌设置为非空" | `src/services/worker/DatabaseManager.ts:39-47`（CloudSync） | 密钥门控可选子系统的本仓库模式。`:39-41` 的注释解释了理由。 |
| 设置声明 + 默认值 | `src/shared/SettingsDefaultsManager.ts:22-131`（接口）、`:134-238`（默认值） | 新 `CLAUDE_MEM_*` 键的声明位置。`:200` 的 `CLAUDE_MEM_CLOUD_SYNC_TOKEN: ''` 是最接近的类比。 |
| 设置读取 | `SettingsDefaultsManager.loadFromFile(USER_SETTINGS_PATH)` —— `worker-service.ts:453`、`DatabaseManager.ts:23` | 规范读取方式。**参见 0.5 中的陷阱。** |
| 设置写入白名单 | `src/services/worker/http/routes/SettingsRoutes.ts:77-108` | 不在此数组中的键**无法通过 `POST /api/settings` 写入**。Chroma / Telegram / CloudSync 键都刻意不在其中。 |
| 已启动 Server 的测试框架 | `tests/server/server-security-headers.test.ts:10-45` | 真实的 `new Server(baseOptions({...}))`、`await server.listen(randomPort, '127.0.0.1')`，然后 `fetch`。**照此文件作为守卫测试的骨架。** |
| 文档表格格式 | `docs/public/configuration.mdx:14-24` | `\| \`CLAUDE_MEM_X\` \| \`default\` \| one-line description \|` |

### 0.3 路由，按远程请求绝不能到达的内容分组

完整清单来自对 `src/services/worker/http/`、`src/services/server/Server.ts`、`src/server/auth/BetterAuthRoutes.ts`、
`src/server/routes/v1/ServerV1Routes.ts` 的 `.get(|.post(|.put(|.patch(|.delete(|.all(` 扫描。

**远程访问后果灾难性（密钥 / 进程控制 / 破坏）：**

| 路由 | 文件:行 | 当前守卫 | 给攻击者什么 |
|---|---|---|---|
| `GET /api/settings` | `SettingsRoutes.ts:29` | **无** | 返回整个设置文件：`CLAUDE_MEM_GEMINI_API_KEY`、`CLAUDE_MEM_OPENROUTER_API_KEY`、`CLAUDE_MEM_CLOUD_SYNC_TOKEN`、`CLAUDE_MEM_CHROMA_API_KEY` |
| `POST /api/settings` | `SettingsRoutes.ts:30` | **无** | 重写 provider/model/host/port 配置 |
| `POST /api/admin/restart` | `Server.ts:291` | `requireLocalhost` | 杀死 worker |
| `POST /api/admin/shutdown` | `Server.ts:305` | `requireLocalhost` | 杀死 worker |
| `GET /api/admin/doctor` | `Server.ts:326` | `requireLocalhost` | pid、环境清洁状态、依赖转储 |
| `DELETE /api/observation\|summary\|prompt/:id` | `DataRoutes.ts:93,94,95` | **无** | 不可逆的行删除 |
| `POST /api/import` | `DataRoutes.ts:102` | **无** | 批量写入记忆数据库 |
| `DELETE /api/corpus/:name` | `CorpusRoutes.ts:76` | **无** | 销毁已构建的知识语料库 |
| `POST /api/logs/clear` | `LogsRoutes.ts:85` | **无** | 抹除审计轨迹 |
| `POST /api/mcp/toggle` | `SettingsRoutes.ts:34` | **无** | 禁用 MCP 子系统 |
| `ALL /api/auth/*splat` | `BetterAuthRoutes.ts:31` | **无** | better-auth：会话、组织、**API 密钥签发** |

**批量数据读取（机器上每个项目，全文）：** `GET /api/observations`、`/api/summaries`、
`/api/prompts`、`/api/observation/:id`、`/api/observations/by-file`、`/api/session/:id`、
`/api/prompt/:id`、`/api/stats`、`/api/projects`、`/api/logs`、`/api/search*`、`/api/timeline*`、
`/api/context/*` —— `DataRoutes.ts:83-100`、`LogsRoutes.ts:84`、`SearchRoutes.ts:146-158`。

**`/v1/*` 下的一切**（`ServerV1Routes.ts:73-260`）—— 已在 `requireServerAuth` 之后，并保持如此；
新守卫也会远程拒绝它，双保险。

**TV 实际需要的四条：** `GET /tv`（`ViewerRoutes.ts:162`）、`GET /tv.html`（由
`middleware.ts:36-38` 的 `express.static(plugin/ui)` 提供）、`GET /stream`（`ViewerRoutes.ts:164`，
处理器 `:224-266`）、`GET /api/observations`（`DataRoutes.ts:83`，处理器 `:104-109`）。

从页面本身确认 —— `src/ui/tv.html` 恰好发起两次网络调用：
`tv.html:265` 的 `fetch('/api/observations?' + qs)` 和 `tv.html:277` 的 `new EventSource('/stream')`。
没有别的。tv.html 无依赖、无外部资源，因此需要**零**个静态资源。

### 0.4 现有中间件及其作用与不作用

- **CORS** —— `middleware.ts:43-62`，在 `Server.ts:208-210` 全局挂载。对不是 `http://localhost:*` /
  `http://127.0.0.1:*` 的 `Origin` 调用 `next(new Error('CORS not allowed'))` 拒绝。
  两点须知：(a) 它只在存在 `Origin` 头时触发，所以 `curl` 和同源页面的 `EventSource` 会直接通过；
  (b) 它**不是**安全边界 —— CORS 限制的是*来自另一源的浏览器页面*可以读取什么，而不是局域网设备
  可以请求什么。
- **没有 `trust proxy`。** 已验证：`app.set('trust proxy', ...)` 在 `Server.ts` 或
  `worker-service.ts` 中均未出现。所以 `req.ip` 是真实 socket 对端，Express 忽略 `X-Forwarded-For`
  —— 攻击者**无法**用头伪造回环 `req.ip`。很好。本计划中不要添加 `trust proxy`；那会破坏该性质。
- **请求日志器** —— `middleware.ts:12-34`。记录 `req.path`，在 Express 中**不含查询字符串**。
  所以 `?token=` 永远不会通过此路径进入日志文件。这对阶段 2 的查询参数决策至关重要 —— 不要
  "改进"日志器去记录 `req.originalUrl`。
- **`finalizeRoutes()` 在 worker 上从未被调用。** 已验证：`grep -rn finalizeRoutes src/`
  返回 `Server.ts:187-191` 的定义和恰好一个调用点 `src/server/runtime/ServerService.ts:217`
  （*另一个*运行时）。因此 worker **没有** `notFoundHandler`，也**没有**终端 `errorHandler`。
  **后果：** 新守卫必须自己写响应。绝不要从中 `next(err)` —— 那会落入 Express 默认处理器并返回
  HTML 堆栈页。
- **HTTP 没有速率限制。** `globalRateLimitStore`（`src/services/worker/RateLimitStore.ts:101`）
  跟踪的是 *LLM provider* 配额，仅在 `ClaudeProvider.ts:303,317` 消费，在 `Server.ts:239` 报告。
  `src/server/middleware/rate-limit.ts` 是 Postgres 支撑的，只接入 Server-Beta Postgres 路由。
  **两者在此都不可复用。** 不要尝试。

### 0.5 陷阱

1. **`SettingsDefaultsManager.get(key)` 不读取 `settings.json`。** `SettingsDefaultsManager.ts:245-247`
   是 `return process.env[key] ?? this.DEFAULTS[key]`。写入 `~/.claude-mem/settings.json` 的令牌
   对 `.get()` 不可见。令牌必须用 `SettingsDefaultsManager.loadFromFile(USER_SETTINGS_PATH)` 读取
   —— 如 CloudSync 所做（`DatabaseManager.ts:23`、`worker-service.ts:453`）。优先级是
   env > settings.json > DEFAULTS（`applyEnvOverrides`，`SettingsDefaultsManager.ts:253-261`）。
2. **`SettingsManager` ≠ `SettingsDefaultsManager`。** `src/services/worker/SettingsManager.ts` 是
   SQLite `viewer_settings` 表，存 `sidebarOpen` / `selectedProject` / `theme`。无关。
   不要把