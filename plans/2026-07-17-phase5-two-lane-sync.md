# 阶段 5：双通道多设备同步（每用户 Durable Object 日志）

**日期：** 2026-07-17
**状态：** 已规划 — 阶段 0 调研完成（三个子代理，实时文档 + 仓库提取）
**决策记录：** 四架构对抗性竞赛（纯 HTTP / 双通道混合 / 原生 socket / 现成引擎）→ 结论：**双通道混合** — HTTP 承载所有持久化数据，WebSocket 仅作为下游建议性加速层。四位倡导者独立收敛到同一存储核心（每用户有序日志、单调 seq、游标续传、幂等应用、变更用 rev、epoch 守卫），因此日志方案已定，争议只在传输层。现成引擎被其自身倡导者否决（ElectricSQL 仅读路径，Zero 拒绝离线写入，cr-sqlite 已停更 30 个月）。托管：Cloudflare Workers 付费版 + 每用户一个 SQLite 后端 DO — 已验证 $5/月（100 用户）、约 $15（1k）、约 $230（10k）；休眠用户成本约 $0。

**继承自决策记录的首要原则：**
1. 开发中期的批量通道不是底线。幸存下来的是四位对抗性倡导者独立保留的部分：DB 即队列的 outbox、失败关闭的设备身份、大小钳制、防抖提示。被淘汰的：Vercel 按类型端点、`toCloud` 映射器的覆盖式 upsert 竞态、`stampGuardSql` 机制、`live|backfill` 通道启发式。
2. 任何持久化数据都不走 socket。HTTP 游标是唯一事实来源；WS 可能出错、断开或被禁用，且零数据丢失。
3. 会话启动时始终立即拉取（内存注入的那一刻是延迟预算中唯一必须花费的部分）。
4. 绝不对 DO 长轮询（挂起的请求会破坏休眠 ≈ $4/设备/月 — 同样的陷阱，换个马甲）。
5. 成本护栏是结构性的：看门狗 → 轮询模式，绝不"停止工作"。

---

## 阶段 0：文档调研 — 汇总发现

三个子代理于 2026-07-17 完成（通过原始 `<url>index.md` 获取实时 Cloudflare 文档；在 `release/v13.11.0` 分支上提取仓库；Bun/vitest 工具链）。执行者：不要重新调研；从下方锚点复制。完整代理报告在会话记录中；下方每条论断均附来源。

### 0.1 允许使用的 Cloudflare API（实时获取；不要发明其他 API）

| API | 确切形式 | 来源 |
|---|---|---|
| DO 类 | `import { DurableObject } from "cloudflare:workers"`；`class SyncHub extends DurableObject<Env>`；`constructor(ctx, env) { super(ctx, env) }` | durable-objects/api/base/ |
| 路由 | `env.SYNC_HUB.getByName(userId)`（一步到位；替代 idFromName+get）。数据端点用 RPC；DO 的 `fetch()` 仅用于 WS 升级（`return stub.fetch(request)`） | best-practices/create-durable-object-stubs-and-send-requests/ |
| SQL | `this.ctx.storage.sql.exec(query, ...bindings)` → 游标，含 `.toArray()`、`.one()`、`.raw()`、`rowsRead/rowsWritten`。**同步消费游标 — 绝不跨 `await`。** 不能通过 exec 执行 `BEGIN`；用 `ctx.storage.transactionSync(cb)`（同步回调）。构造函数中通过 `ctx.blockConcurrencyWhile()` 初始化 Schema | api/sqlite-storage-api/ |
| 休眠 WS | `this.ctx.acceptWebSocket(server, tags?)`；类方法 `webSocketMessage(ws, msg)`、`webSocketClose(ws, code, reason, wasClean)`、`webSocketError(ws, err)`；`this.ctx.getWebSockets(tag?)`；`ws.serializeAttachment(obj)`（**最大 16 KB** — 存键，不存状态）/ `deserializeAttachment()`；升级：`new WebSocketPair()` → `acceptWebSocket(server)` → `new Response(null, { status: 101, webSocket: client })` | best-practices/websockets/ + examples/websocket-hibernation-server/ |
| 保活 | `this.ctx.setWebSocketAutoResponse(new WebSocketRequestResponsePair("ping", "pong"))`（各 ≤2,048 字符；无需唤醒 DO 即可应答）。协议级 ping 由运行时自动 pong，同样无需唤醒 | api/state/ + best-practices/websockets/ |
| 闹钟 | `await this.ctx.storage.setAlarm(msEpoch)`（每 DO 一个；**每次 setAlarm = 1 次计费行写入**）；处理器 `alarm(info?: {retryCount, isRetry})` — 至少一次，必须幂等，必须自我重排；**构造函数中设置前始终先 `getAlarm()` 检查** | api/alarms/ |
| Wrangler | 最小键 `name`/`main`/`compatibility_date`。DO 绑定 + 二选一：`migrations: [{tag:"v1", new_sqlite_classes:["SyncHub"]}]`（示例页流程）或较新的 `exports: {"SyncHub": {type:"durable-object", storage:"sqlite"}}` — 只选一种流程。脚手架：`npm create cloudflare@latest -- durable-object-starter` | reference/durable-objects-migrations/ + workers/wrangler/configuration/ |

**塑造设计的硬限制**（platform/limits 页面）：每 DO 10 GB SQLite；**单行最大 2 MB**；单语句最大 100 KB；**每查询 100 个绑定参数**（多行 INSERT 按 ≤ ~12 行 × 8 列分块）；传入 WS 消息最大 32 MiB；128 MB isolate 内存（⇒ 重放必须分页，绝不缓冲 50 MB）；每 DO 约 1,000 请求/秒软限制；传入 WS 消息按 20:1 计费，传出免费。

### 0.2 仓库锚点（确切 file:line，已在本分支验证）

| 需求 | 锚点 |
|---|---|
| 迁移链追加点 | `src/services/sqlite/SessionStore.ts:111`（构造函数链中最后一次调用；**占用 v41** — v40 是当前最高） |
| 加列迁移模板 | `ensureSyncedAtColumns` SessionStore.ts:447-474（PRAGMA 检查 → ALTER → 部分索引 → 版本行作为记账） |
| 版本门控一次性模板 | `requeuePromptCloudSyncAfterMapperFix` SessionStore.ts:488-500 |
| 变更点 (a)：worktree 重映射 | WorktreeAdoption.ts:210-215（observations + session_summaries 上的 `merged_into_project`，在事务 224-225 内）。**打开自己的 DB 连接（第 185 行）— 无法触达 `notify()`；rev 递增必须在此用纯 SQL** |
| 变更点 (b)：cwd 重映射 | ProcessManager.ts:312-314（sdk_sessions/observations/session_summaries 上的 `project`，事务 317-326）。**同样是自己的连接（第 276 行）** |
| 变更点 (c)：自定义标题 | SessionStore.ts:1922-1927（`custom_title` 的唯一写入者，在 `createSDKSession` 内） |
| 变更点 (d)：prompt→session 修复 | `requeuePromptSync` SessionStore.ts:1492-1497；调用方在 1481（`updateMemorySessionId`）和 1526（`ensureMemorySessionIdRegistered`） |
| 应用机制 | `storeObservations` SessionStore.ts:2060-2176（事务，内容哈希 `ON CONFLICT ... DO NOTHING RETURNING id`，**`overrideTimestampEpoch` 参数已存在** — 用它保留远端 `created_at_epoch`）；`storeSummary` 2014-2058；`saveUserPrompt` 1940-1954 |
| FTS | 基于触发器 — 直接 INSERT 自动建索引。observations/summaries FTS + 触发器：SessionSearch.ts:76-152；user_prompts FTS：SessionStore.ts:867-895。无 FTS5 时优雅缺失 |
| Chroma 摄取 | `syncObservation` ChromaSync.ts:375-427，`syncSummary` 429-474，`syncUserPrompt` 492-530；即发即忘的 `.then().catch()` 模式见 ResponseProcessor.ts:296-318 |
| 路由类模板 | `CloudSyncRoutes` src/services/worker/http/routes/CloudSyncRoutes.ts:15-32；基类是 `src/services/worker/http/BaseRouteHandler.ts`（不在 routes/ 下）；延迟注册在 worker-service.ts:534 |
| 设置键 | 接口 SettingsDefaultsManager.ts:67-73 + DEFAULTS 158-163；扩展 CloudSync.ts:184-190 的 `CloudSyncSettingKeys` Pick |
| 定时器惯用法 | `notify()` CloudSync.ts:297-313（第 305 行 `.unref?.()`）；`scheduleRetry` 493-503；单飞 `flush` 320-349；`stopped` 复查 399/413/418/434/442 |
| 服务类形态 | `TranscriptWatcher` watcher.ts:86-111（constructor/start/stop）；启动触发点 worker-service.ts:612 |
| 会话启动拉取钩子 | `SearchRoutes.handleContextInject` SearchRoutes.ts:278 — 将拉取插入约 317 行（动态导入）与 332 行（`generateContextWithStats`）之间。会话初始化：SessionRoutes.ts:385-562，现有提示在 493 |
| 测试框架 | tests/worker/sync/cloud-sync.test.ts — `makeFetchMock` 24-40，`makeCloudSync` 59-73（注入 `fetchImpl`，快速防抖/退避），stampGuard 回归测试 370-409。运行器：`bun test`（package.json:98） |
| 构建注意事项 | build-hooks.js 基于入口点（从 `worker-service.ts` 可达的新 src/ 代码自动打包）。**sync-marketplace.cjs:78 会 rsync 仓库根目录 — 新的顶层 `workers/` 目录会被复制进已安装插件，除非加入排除列表** |
| WS 先例 | src/ 中无（零命中）— 客户端是全新开发；SSE + TranscriptWatcher 是最接近的类比 |

### 0.3 工具链事实（实时获取）

- **Bun WS 客户端原生支持认证头**（Bun 特有扩展）：`new WebSocket(url, { headers: { Authorization, "X-Device-Id" } })`；还有 `ping()`、`pong()`、`terminate()`、`bufferedAmount`、`tls`、`proxy` 选项。来源：bun.com/docs/runtime/http/websockets + bun.com/reference/bun/WebSocketOptions。无需 `ws` npm 依赖。
- **已验证的保活配对：** Bun 客户端每 30–45 秒发送协议 `ws.ping()` → CF 运行时自动 pong **且不唤醒 DO**（有文档）。另外为未来浏览器客户端设置 `setWebSocketAutoResponse("ping","pong")`（浏览器无法发送协议 ping）。调研标记的缺口：Bun 对服务端 ping 的*自动 pong* 在文档中未验证 — 此处无关，因为由客户端发起。
- **测试（2026 形态 — 训练数据片段已过时）：** `@cloudflare/vitest-pool-workers` ≥0.18 使用 Vite 插件：`plugins: [cloudflareTest({ wrangler: { configPath: "./wrangler.jsonc" } })]`，配 **Vitest ≥4.1**（`defineWorkersConfig` 在 v0.13 中已移除）。测试 API 来自 `"cloudflare:test"`：`runInDurableObject(stub, (instance, state) => ...)`、`runDurableObjectAlarm(stub)`、`evictDurableObject(stub, { webSockets: "hibernate" })`。**已知问题：WS + DO 测试需要 `--max-workers=1 --no-isolate`** — 将 WS 测试放在单独的 vitest 调用中。`wrangler dev` 本地提供 `ws://localhost:8787`（SQLite DO 不支持 `--remote`）。
- **Lint 守卫：** ESLint flat config `no-restricted-globals`（setTimeout/setInterval）+ `no-restricted-syntax`（选择器 `CallExpression[callee.property.name='accept']`、裸 `fetch(`），作用于 DO 源文件 glob — 外加一个简单的 `grep -rn` CI 步骤（捕获 `globalThis.setTimeout` 规避）。

### 0.4 反模式清单（已对照实时文档验证 — 这些是 $4.11/用户/月 + $34k 的陷阱）

1. `server.accept()` + `addEventListener`（旧版 WS API）— 破坏休眠。只用 `ctx.acceptWebSocket()` + 类处理器方法。
2. DO 中任何位置的 `setTimeout`/`setInterval` — 使其保持唤醒。只用闹钟，且只在有工作时。
3. **DO 不得有任何出站 I/O — 完全禁止。** 2026 年 6 月 19 日变更：出站 `connect()`/WebSocket 每个连接使 DO 保持唤醒最多 15 分钟；被 await 的出站 `fetch()` 阻塞空闲。Token 验证、上游调用等一切都在前面的无状态 Worker 中完成。
4. 构造函数中 `setAlarm()` 未先 `getAlarm()` 检查 — 有文档记载的 $34k 失控背后的坑。
5. 用内存 map 作为 socket 状态 — 休眠时丢失。附件（≤16 KB）存键；SQLite 存状态；构造函数从 `getWebSockets()` 重建。
6. 在内存中缓冲完整重放（128 MB isolate）— 分页 `/changes`，分块 WS 推送。
7. 对 DO 长轮询 / 挂起 HTTP 请求 — 经济上与 #2 相同。

---

## 阶段 1：同步枢纽（Worker + 每用户 DO，仅 HTTP 通道）

**目标：** 全新 `workers/sync-hub/` wrangler 项目 — 每用户有序日志 + 两个 HTTP 端点。本阶段无 socket。

**任务：**
1. 用 `npm create cloudflare@latest -- durable-object-starter` 脚手架到 `workers/sync-hub/`（自有 package.json；不被 build-hooks.js 触达 — 这是正确的，它通过 wrangler 部署，不走插件包）。**将 `workers/` 加入 sync-marketplace.cjs 排除列表（sync-marketplace.cjs:78）**，以免已安装插件携带它。
2. `wrangler.jsonc`：复制休眠示例配置形态（阶段 0.1 wrangler 行）— 绑定 `SYNC_HUB` → 类 `SyncHub`，`migrations: [{tag:"v1", new_sqlite_classes:["SyncHub"]}]`，`compatibility_date` 用当前值。
3. `SyncHub extends DurableObject<Env>`：构造函数中通过 `blockConcurrencyWhile` 初始化 Schema（从 sqlite-storage-api "SQL API" 节复制形态）：
   ```sql
   CREATE TABLE IF NOT EXISTS ops (
     seq           INTEGER PRIMARY KEY AUTOINCREMENT,
     kind          TEXT NOT NULL,            -- 'observation'|'summary'|'prompt'|'mutation'
     origin_device TEXT NOT NULL,
     origin_id     TEXT NOT NULL,            -- device-local rowid; op UUID for mutations
     rev           INTEGER NOT NULL DEFAULT 1,
     body          TEXT NOT NULL,            -- canonical row JSON (opaque-able later for E2E)
     server_ts     INTEGER NOT NULL
   );
   CREATE UNIQUE INDEX IF NOT EXISTS ops_entity ON ops(origin_device, kind, origin_id, rev);
   CREATE TABLE IF NOT EXISTS devices (device_id TEXT PRIMARY KEY, name TEXT, last_ack_seq INTEGER DEFAULT 0, last_seen INTEGER);
   CREATE TABLE IF NOT EXISTS meta (k TEXT PRIMARY KEY, v TEXT);  -- epoch, counters
   ```
4. RPC 方法（不是 fetch — 阶段 0.1 路由行）：`pushOps(deviceId, ops[]) → {acked:[{origin_id, rev, seq}], head_seq}`（通过唯一索引幂等 — 重复推送返回现有 seq；INSERT 按 ≤100 绑定参数分块；每行 body ≤2 MB 强制，客户端钳制作为兜底）和 `getChanges(sinceSeq, limit≤500) → {epoch, ops[], head_seq, more}`（游标 `.toArray()` 同步消费 — 阶段 0.1 SQL 行）。
5. 前面的无状态 Worker：解析 `Authorization: Bearer` + `X-User-Id` + `X-Device-Id`（与今天相同的头 — CloudSync.ts:448-466），验证 token，路由 `env.SYNC_HUB.getByName(userId)`。**Token 验证在此处，绝不在 DO 中**（反模式 #3）：调用 cmem.ai 的验证端点，并将结果以短 TTL 缓存到 Workers KV。路由：`POST /v1/sync/ops`、`GET /v1/sync/changes`、`GET /v1/sync/status`。
6. 变更操作（阶段 3 写入它们；枢纽现在必须接受）：`kind='mutation'`，body `{op: 'set_title'|'set_prompt_session'|'remap_project', target|where, fields}`。谓词合理性上限：如果枢纽无法解析来自设备的 `remap_project` 操作则拒绝 — 仅验证；应用在客户端