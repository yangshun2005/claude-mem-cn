# Worker 原生云同步 — 数据库即队列

**日期：** 2026-07-09
**取代：** `plans/inbox/2026-07-08-cloud-sync-skill.md`（技能管理的独立守护进程 — 已否决：它在照看一个本不该存在的进程）。

**一段话设计：** 每条记忆行新增 `synced_at` 列（NULL = 未上云）。worker — 它本就执行所有写入、且已被 Hook 保活 — 在每次写入后触发一个防抖的刷写器。刷写器分批排空 `WHERE synced_at IS NULL`，POST 到 cmem.ai，成功后给行打戳。这单一机制**就是**实时同步、回填、离线追赶和重试。没有第二个进程、没有轮询、没有 pid 文件、没有 JSON 游标文件。配置就是 settings.json 里的一个 token，和其他所有 API key 一样。技能缩减为：粘贴 token → 重启 worker → 显示状态。

**线上契约（固定 — cmem.ai 服务端已部署）：** POST `{observations,summaries,prompts}/batch`，使用 camelCase 行映射器，请求头 `Authorization: Bearer`、`X-User-Id`、`X-Device-Id`、`X-Device-Name`；服务端按 `(user_id, device_id, local_id)` upsert；请求体 ≤2MB，字段钳制 ≤200KB；`GET /pull?kind=&afterId=&limit=`。所有映射器/钳制的参考实现是 `~/.claude-mem/cloud-sync.mjs` 中经审查的独立客户端（2026-07-08 源码审查；已对生产环境实测 — 129,777 行）。**逐字移植其 `toCloud` 映射器、`selectCols` prompt 钳制和请求体大小分批；不要重新设计线上格式。**

---

## 阶段 0：文档发现（已完成 — 已验证 file:line 事实）

**允许的 API / 复制来源：**

| 关注点 | 复制自 | 证据 |
|---|---|---|
| 列迁移 | `ensureDiscoveryTokensColumn()` — `schema_versions` 门控 + `PRAGMA table_info` 守卫 + `ALTER TABLE ADD COLUMN` | `src/services/sqlite/SessionStore.ts:931–953` |
| 写入点 Hook 形态 | `dbManager.getChromaSync()?.syncObservation(...)` — 即发即忘、可选链 | `src/services/worker/agents/ResponseProcessor.ts:343`（obs）、`:430`（summaries）、`src/services/worker/http/routes/SessionRoutes.ts:587`（prompts） |
| 服务访问器 | `getChromaSync(): ChromaSync \| null` | `src/services/worker/DatabaseManager.ts:62` |
| 同类服务类 | `ChromaSync`（构造函数、逐写方法、`static backfillAllProjects`） | `src/services/sync/ChromaSync.ts:83,89,361,914` |
| 后台接线点 | `WorkerService.initializeBackground()` — 设置加载于 `:460`，Chroma 门控 `:504`，回填 `:604` | `src/services/worker-service.ts:452–622` |
| 设置键 | `SettingsDefaults` 接口 + `DEFAULTS`；Chroma 远程组（`CLAUDE_MEM_CHROMA_HOST/PORT/API_KEY/...`）是端点+凭据的先例；文件模式 0600 | `src/shared/SettingsDefaultsManager.ts:27–130, 133–236, 101–106, 9–25` |
| HTTP 路由 | `LogsRoutes` 继承 `BaseRouteHandler`；像 SearchRoutes 一样延迟注册 | `src/services/worker/http/routes/LogsRoutes.ts:70–137`、`src/services/worker-service.ts:528` |
| 测试 | `bun test` 跑 `tests/`（子目录：`tests/sqlite/`、`tests/worker/...`） | 根 `package.json:99–105` |
| 技能编写 | 从 `plugin/skills/<name>/SKILL.md` 自动发现；frontmatter `name/description/allowed-tools`；由 `scripts/sync-marketplace.cjs:126–167` 原样发布 | `plugin/skills/standup/SKILL.md` |
| 待移植的线上映射器 | `KINDS[].toCloud`、prompts `selectCols`（SQL 侧 `substr(prompt_text,1,200000)` — 7MB 的 prompt 若读后钳制会让 node OOM）、`MAX_BODY_BYTES`/`clampRow` | `~/.claude-mem/cloud-sync.mjs:132–281` |

**反模式（不要做）：** 在 plugin.json 里发明 `skills` 键（自动发现）；使用 `${CLAUDE_PLUGIN_ROOT}`（真正的变量是 `${CLAUDE_SKILL_DIR}`，而且此技能已不再捆绑脚本）；使用 `node:sqlite`/`better-sqlite3`（仓库用的是 `bun:sqlite`）；在写入路径上因网络 I/O 阻塞（Chroma 调用是本地快速；云不是 — 触发即可，不要 await）；编辑 CHANGELOG；添加任何 pm2/launchd 相关；**在存在旧设备 id 时铸造新设备 id**（服务端以 device_id 为键 — 新 id 会把每条云行分叉成重复）。

---

## 阶段 1：Schema — `synced_at`

**做什么：** 一次迁移，复制 `ensureDiscoveryTokensColumn` 模板（`SessionStore.ts:931–953`），取下一个空闲的 `schema_versions` 编号（执行器：先 `SELECT MAX(version) FROM schema_versions`）：

1. `ALTER TABLE observations ADD COLUMN synced_at INTEGER` — `session_summaries`、`user_prompts` 同理。（NULL = 未同步；值 = 成功上传的 epoch 毫秒。）
2. 部分索引，让排空查询保持 O(pending)：`CREATE INDEX IF NOT EXISTS idx_<table>_unsynced ON <table>(id) WHERE synced_at IS NULL`。

**旧数据接管（同一迁移，只跑一次）：** 若 `~/.claude-mem/cloud-sync-state.json` 存在（独立客户端的状态 — 格式：`{deviceId, lastId, lastSummaryId, lastPromptId, ...}`）：
- 给已推送的行打戳：`UPDATE observations SET synced_at = <now> WHERE id <= lastId AND synced_at IS NULL`（另两张表同理）。跳过此步是*安全的*（服务端会 upsert），但会重传约 13 万行 — 给它们打戳。
- 保留状态文件；阶段 2 从中接管 `deviceId`，技能（阶段 4）将其退役。

**验证：**
- [ ] `bun test tests/sqlite/` 通过；新测试：在 fixture DB 上迁移幂等地添加列（跑两次），存在 fixture 状态文件时给 ≤ 游标的行打戳，> 游标的行保持 NULL。
- [ ] `PRAGMA table_info(observations)` 显示 `synced_at`；`EXPLAIN QUERY PLAN SELECT id FROM observations WHERE synced_at IS NULL` 使用部分索引。

**反模式守卫：** 不要动现有列或重建表；无旧状态文件时不要打戳。

## 阶段 2：`CloudSync` 服务 — 刷写器

**做什么：** 新建 `src/services/sync/CloudSync.ts`，结构上是 `ChromaSync.ts` 的同类。核心：

- **配置**（`SettingsDefaultsManager` 接口 + DEFAULTS 中的新键，仿照 `:101–106` 的 Chroma 组）：`CLAUDE_MEM_CLOUD_SYNC_TOKEN`（默认 `''`）、`CLAUDE_MEM_CLOUD_SYNC_USER_ID`（`''`）、`CLAUDE_MEM_CLOUD_SYNC_URL`（`https://cmem.ai/api/pro/sync`）、`CLAUDE_MEM_CLOUD_SYNC_DEVICE_ID`（`''` — 首次启动时解析：若存在旧 `cloud-sync-state.json` 则从中接管，否则 `randomUUID()`，然后持久化回设置）、`CLAUDE_MEM_CLOUD_SYNC_DEVICE_NAME`（默认 `hostname()`）。**激活 ⇔ token 和 user id 均非空。** 无独立启用标志 — 有凭据即开，token 为空即关。
- **`notify()`** — 由写入点调用；约 1500ms 尾随防抖；把写入突发合并为一次刷写。
- **`flush()`** — 单飞（已在运行则跳过）。按类型：`SELECT ... WHERE synced_at IS NULL ORDER BY id LIMIT 200`（prompts 用移植的 SQL 钳制），用移植的 `toCloud` 映射器映射，打包成 ≤2MB 请求体，带 `AbortSignal.timeout(30_000)` POST（**修复独立客户端无超时挂起的问题**），2xx 时给该批打戳：`UPDATE <table> SET synced_at = ? WHERE id IN (...)`。循环直到排空。失败时：经仓库 logger 记录，行保持 NULL，下次 notify 时重试 + 有上限的指数退避定时器（30s → 10min，`.unref()`）。
- **`start()`** — 触发一次 `flush()`（非阻塞）。这**就是**回填：从未同步过的安装自然全是 NULL。
- **`status()`** — `{configured, deviceId, pending: {observations, summaries, prompts}, lastFlushAt, lastError}`（pending = `COUNT(*) WHERE synced_at IS NULL`）。
- **可测试性：** 构造函数接收 `fetchImpl = globalThis.fetch`。

**接线（3 + 2 行）：**
- `DatabaseManager`：持有实例 + `getCloudSync(): CloudSync | null` 访问器（复制 `:62`）。
- `initializeBackground()`（`worker-service.ts:452–622`）：设置加载（`:460`）后，若激活 → 构造 + `start()`，放在 Chroma 门控（`:504`）/回填（`:604`）附近。
- 写入点 — 各一行，紧挨现有 Chroma 调用：`dbManager.getCloudSync()?.notify()` 于 `ResponseProcessor.ts:343` 附近、`:430` 附近、`SessionRoutes.ts:587` 附近。

**验证：**
- [ ] 新建 `tests/worker/sync/cloud-sync.test.ts`（mock fetch）：notify 突发 → 恰好一次刷写；排空给行打戳并清空 pending；POST 失败保持 NULL + 重试；请求体打包遵守 2MB；>200KB 的 prompt 到达时被钳制并带截断标记；空 token → `start()` 空操作。
- [ ] 载荷形态黄金测试：fixture 行的映射器输出与独立客户端对同一行的映射器输出深度相等（守卫线上契约）。
- [ ] 完整 `bun test` 通过。

**反模式守卫：** 绝不在写入路径 `await` 同步；无新依赖；不要记录 token（只记录其长度或末 4 位）；不要重推拉取来的/外来的行（阶段 5 前无意义，但届时排空查询会加上 `AND origin_device_id IS NULL`）。

## 阶段 3：状态端点

**做什么：** `GET /api/sync/status` → `cloudSync.status()`（未激活时外加 `{configured: false}`）。新建 `CloudSyncRoutes extends BaseRouteHandler`，复制自 `LogsRoutes.ts:70–137`；在服务存在后于 `initializeBackground()` 中注册（SearchRoutes 模式，`worker-service.ts:528`）。

**验证：** [ ] `curl 127.0.0.1:<port>/api/sync/status` 返回 pending 计数；未配置的 worker 返回 `configured: false`，而非 500。

## 阶段 4：技能 — 仅薄前端

**做什么：** `plugin/skills/cloud-sync/SKILL.md`（自动发现；无捆绑脚本）。Frontmatter：`name: cloud-sync`、`allowed-tools: [Bash, Read, AskUserQuestion]`，description 在"设置云同步 / 同步我的记忆 / cmem pro / 云备份 / 同步状态"时触发。

技能编码的操作手册：
1. `GET /api/sync/status`（worker 端口取自 `~/.claude-mem/settings.json` → `CLAUDE_MEM_WORKER_PORT`）。
2. **已配置** → 报告 pending 计数 + 最后错误，完成。
3. **未配置** → 获取凭据，按优先级：(a) 存在旧 `~/.claude-mem/.cloud-sync.env` → 从中读取 token/user-id，告知用户正在迁移；(b) 否则请用户从 **cmem.ai → Connect** 粘贴 token + user id。两者写入 `~/.claude-mem/settings.json`（保持 0600；绝不回显 token，绝不把它放到命令行上）。
4. **旧守护进程退役：** 若 `~/.claude-mem/cloud-sync.pid` 持有存活 pid → 杀掉它；然后归档独立产物（`cloud-sync.mjs`、`.cloud-sync.env`、`cloud-sync.pid` → 加 `.retired` 后缀重命名；保持 `cloud-sync-state.json` 不动 — 阶段 1 迁移和阶段 2 设备 id 接管会读它）。
5. `POST /api/admin/restart`，然后轮询 `/api/sync/status` 直到 pending 计数排空；报告。
6. 仅首次设置：一行隐私说明（同步会把 observation 叙述 + 完整 prompt 文本上传到用户的 cmem.ai 账户）。

**验证：**
- [ ] 全新机器路径：无旧文件、无配置 → 提示输入凭据 → 状态排空到 0。
- [ ] **本机（实时迁移冒烟）：** 2026-07-08 起的旧守护进程在运行 → 技能读取 `.cloud-sync.env`，杀掉守护进程，worker 接管设备 id `ee1b7637-…`（在设置 + 状态中验证），pending 立即 ≈ 0（行已由阶段 1 按游标打戳），新 observation 在写入后数秒内落到云端 — **cmem.ai 仪表盘不出现重复行**（设备 id 连续性证明）。
- [ ] `grep -rn "CLAUDE_PLUGIN_ROOT\|nohup\|\.pid" plugin/skills/cloud-sync/` → 只有退役步骤触及 `.pid`。

## 阶段 5（单独发布）：拉取 — 多设备入站

**做什么：** 三张表新增列 `origin_device_id TEXT`、`origin_local_id TEXT`（+ 该对的 `UNIQUE` 部分索引 `WHERE origin_device_id IS NOT NULL` — 取代独立客户端的 JSON"导入账本"）；worker 在启动时和每次刷写后按类型 `GET /pull`，只插入来自其他设备的行（对唯一索引 `INSERT OR IGNORE`；跳过 `device_id IN (mine, 'legacy')`）；推送排空加上 `AND origin_device_id IS NULL`，使外来行永不回传。拉取游标（`pullAfter`）放在小型 `sync_meta` KV 表中，而非文件。

**验证：** [ ] 双 fixture 设备测试：设备 B 的行在本地出现一次（重拉 = 无重复），绝不以 A 的 id 重传。

## 阶段 6：文档 + 清理 + E2E

1. `docs/public/cloud-sync.mdx`（以 `hosted-server.mdx` 为范本）：同步什么、隐私说明、`/cloud-sync` 快速开始、设置键表、状态端点、"无守护进程 — worker 在写入时同步"架构说明。注册到 `docs/public/docs.json` 导航（Hosted Server 组，第 102–108 行）。README 指针放在 `README.md:428` 附近；不要动 i18n README。
2. `npm run build-and-sync`；确认 worker 重启干净，技能在市场 + 缓存目录中可见。
3. E2E grep：任何地方无真实 token（`grep -rn "cm_pro_" — 仓库必须干净`）；src 中无 `node:sqlite`；写入点显示成对的 chroma+cloud 单行。
4. 完整 `bun test` + 实时冒烟（阶段 4 的迁移清单）— 然后删除 `plans/inbox/2026-07-08-cloud-sync-skill.md` 的过时同类产物（若仍有残留）。

---

## 明确否决（来自被取代的计划）

- 技能中捆绑 `cloud-sync.mjs` + `${CLAUDE_SKILL_DIR}` 调用 — 已无脚本可捆绑。
- 专用 `.cloud-sync.env` — 用 settings.json，和其他所有凭据一样（Chroma 先例）。
- pid 文件 / 单实例守卫 / nohup / 重启注意事项 — 无进程可守卫。
- 10s 本地 DB 轮询 — 由写入点触发 + 启动排空取代。
- 独立客户端保持其本来面目：一个仓库外的工具，由 cmem.ai 提供给非插件用户。本仓库不再依赖它。