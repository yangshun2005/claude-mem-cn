# 隔夜修复 — 单轮 (v13.12.4)

来自 v13.12.2 之后 Issue 批次的四个根因修复，作为一个发布版本交付：

| 阶段 | Issue | 缺陷 |
|-------|-------|--------|
| 1 | #3380 (+ #3378 日志) | 当 http server 句柄未监听时，关闭流程跳过所有清理（`ERR_SERVER_NOT_RUNNING` 被 reject 而非 resolve） |
| 2 | #3379 | 带前缀的概念标签（`"gotcha: …"`）被精确匹配 SQL 静默排除在上下文注入之外 |
| 3 | #3378（后半部分） | `FOREIGN KEY constraint failed` 中止后台初始化 → worker 永不就绪；外加 `errors=[object Object]` 日志问题 |
| 4 | #3381 | 根 CLAUDE.md 中仅维护者可见的 agent 指令通过 marketplace git clone 交付给最终用户 |
| 5 | — | 验证 + 发布 13.12.4 |

**执行模型：** 每个阶段自包含，可在全新上下文中运行。按顺序执行（阶段 3 不依赖 1–2，但阶段 5 需要 1–4 全部提交）。用 `/claude-mem:do` 或手动执行。

**约束性要求（仓库合并准则，`docs/merge-rubric.md`）：** 仅做根因修复。禁止重试循环、熔断器、回退路径、fail-open 模式、截断用户数据、新增后台进程/状态文件、环境变量逃生舱。鼓励 fail-fast 转换和删除机制。测试始终在范围内。Diff 大小必须与逻辑错误的大小匹配。不要编辑 `CHANGELOG.md`（自动生成）。

---

## 阶段 0 — 汇总发现（任何阶段前先读）

以下所有发现均已对照 v13.12.3（commit `68b077f7f` 时期）源码验证，附精确引用。不要重新推导；不要臆造 API。

### 允许的 API / 可直接复制的模式

| 模式 | 位置 | 用于 |
|---------|----------|------|
| 关闭时容忍 `ERR_SERVER_NOT_RUNNING`（幂等关闭的领域状态） | `src/server/runtime/ServerService.ts:230-236` | 阶段 1 |
| 容忍该错误的测试模式 | `tests/server/server.test.ts:141-147`, `:167-173` | 阶段 1 |
| 正确的句柄生命周期（`if (!this.server) return;` + 关闭后置 null） | `src/services/server/Server.ts:159-178`（`Server.close()`） | 阶段 1 |
| 概念清理点（唯一现有的概念规范化） | `src/sdk/parser.ts:118`（`concepts.filter(c => c !== finalType)`） | 阶段 2 |
| 版本门控的一次性迁移模板（不新增列） | `src/services/sqlite/SessionStore.ts:1537-1552`（`addSessionCustomTitleColumn`） | 阶段 2 |
| 带 UPDATE 的数据修复迁移 | `SessionStore.ts:1520-1535`（`addObservationContentHashColumn`，v22） | 阶段 2 |
| FK 安全的父先于子 upsert（`INSERT … ON CONFLICT … DO NOTHING RETURNING`） | `src/services/sync/SyncApply.ts:646-703`（`ensureSessionForMemoryId`） | 阶段 3 |
| 日志约定：将 `Error` 作为第 4 个 `data` 参数传入；上下文值必须是标量/字符串 | `src/utils/logger.ts:135-138`, `:252-255`；计数风格示例 `WorktreeAdoption.ts:342` | 阶段 3 |
| 迁移测试模板 | `tests/sqlite/session-store-migrations.test.ts` | 阶段 2–3 |

### 事实（附引用）

**关闭（#3380）：**
- `"Server is not running."` 是 **Node 原生的 `ERR_SERVER_NOT_RUNNING`**，来自对未监听句柄调用 `http.Server.close(cb)` — 它不是仓库字符串（`ServerService.ts:360/:388` 的 console 打印是无关的 CLI 状态行）。
- Reject 位置：`src/services/infrastructure/GracefulShutdown.ts:60-69`（`closeHttpServer`）— `server.close(err => err ? reject(err) : resolve())`，无 code 检查。它是 `performGracefulShutdown` 的**第一步**（`GracefulShutdown.ts:33-36`），因此其 reject 会跳过会话排空、MCP 关闭、chroma 停止、db 关闭和 supervisor 停止。
- 该 reject 在 `src/services/worker-shutdown.ts:96-113` 被捕获并记录（"Graceful shutdown failed — proceeding"），进程仍通过 `flushResponseThen.ts:8-13` 退出 — 因此 socket 可能永远不会被优雅关闭（#3380/#2111 中的 Windows 端口占用）。
- 促成状态 bug：`Server.listen()`（`src/services/server/Server.ts:140-157`）在第 143 行、socket 绑定**之前**就赋值 `this.server = server`，且 `listen` reject（EADDRINUSE）时从不清除 — 留下非 null、未监听的句柄。由 `tests/server/server.test.ts:113-127` 演示。
- Worker 接线：单个 `Server` 实例（`worker-service.ts:272`），单次 `listen`（`:421`）；优雅关闭在 `worker-service.ts:813-819` 消费原始句柄（`server: this.server.getHttpServer()`）。
- 现有关闭测试 mock 绕过了此路径：`tests/services/worker-shutdown-sequence.test.ts` mock 了 `performGracefulShutdown`；`tests/infrastructure/graceful-shutdown.test.ts:104-107` 中 `mockServer.close` 总是成功。未监听场景目前未测试。

**概念（#3379）：**
- 注入查询：`src/services/context/ObservationCompiler.ts:56-59` — `AND EXISTS (SELECT 1 FROM json_each(o.concepts) WHERE value IN (…))`，占位符来自 `config.observationConcepts`（`:28-29`），源自 `src/services/context/ContextConfigLoader.ts:13` 的激活模式。
- 有效的 `code` 模式标签（`plugin/modes/code.json:63-101`）：`how-it-works, why-it-exists, what-changed, problem-solution, gotcha, pattern, trade-off`。带 `observation_concepts` 的模式：`code.json`、`email-investigation.json`、`law-study.json`、`meme-tokens.json`。
- 持久化路径：`src/sdk/parser.ts:101` 提取概念；唯一的规范化是第 118 行（丢弃等于 observation type 的概念）。SQLite 写入 `SessionStore.ts:2517-2540`（`:2540` 处 `JSON.stringify(observation.concepts)`）— 无验证。次级原始插入路径在 `SessionStore.ts:3032`。
- 生产者提示：自由文本 XML（`src/sdk/prompts.ts:43-49`）；约束文本是各模式 JSON 中的 `prompts.concept_guidance`，其自身格式（`gotcha: traps or edge cases`）诱导模型回显 `keyword: description`。并行模板存在于 `src/server/generation/providers/shared/prompt-builder.ts:155`。
- 回填也会修复的精确匹配读取方：`SessionSearch.ts:207`、`SessionStore.ts:2146`。
- 迁移引擎：私有方法在 `SessionStore` **构造函数**中顺序调用（`SessionStore.ts:89-121`），记录在 `schema_versions`；当前最大版本 **48**；迁移在 worker 启动时自动运行（`DatabaseManager.ts:28`）。FTS `AFTER UPDATE` 触发器存在（`SessionSearch.ts:107-112`）— 回填 UPDATE 运行时验证触发器存在/顺序（见阶段 2 清单）。
- 注意：入站云同步（`SyncApply.ts:782`、`CloudSync.ts:205`）可能从其他客户端重新导入畸形概念；在修复提交中注明，超出解决范围。

**后台初始化 FK 中止（#3378 后半部分）：**
- **Adoption 本身无法引发 FK 错误**：`adoptMergedWorktrees`（`src/services/infrastructure/WorktreeAdoption.ts:122`）仅 UPDATE `observations`/`session_summaries` 的非键列（`project`、`merged_into_project`、`sync_rev`、`synced_at`），并 INSERT 到无 FK 的 `sync_outbox`（`remap-outbox.ts:173-186`）。这些表上唯一的 FK 是 `memory_session_id → sdk_sessions(memory_session_id)`（`SessionStore.ts:903`、`:925`），SQLite 立即 FK 仅在子**键**列变化时重新检查。FK 强制按连接开启（`connection.ts:45`）。
- 自 #2122 起 adoption 是 **fire-and-forget**（`worker-service.ts:480-496`，commit `94d592f21`），在当前 HEAD 上无法中止 init。中止 init 的 FK 错误必须来自 `initializeBackground` 中的**被 await 的**步骤 — 主要嫌疑：`await this.dbManager.initialize()`（`worker-service.ts:507`）→ SessionStore 构造函数迁移**重建**（`SessionStore.ts:211-224`、`:1085-1129` — 若存在孤立的 `observations.memory_session_id` 行，`INSERT … SELECT` 复制会出错），以及 `runOneTimeV12_4_3Cleanup()`（`:509`，在 `CleanupV12_4_3.ts:253` 从 `sdk_sessions` 删除，FK 开启 → CASCADE）。
- 就绪：`initializationCompleteFlag = true; resolveInitialization();` 在 `worker-service.ts:579-580`。此前任何 throw 落入 `:668-670` 的 catch，记录并返回 — worker 永不报告就绪。（此门控是正确的 fail-fast；不要改。）
- `errors=[object Object]`：`worker-service.ts:487-490` 将 `errors: adoption.errors`（`Array<{worktree, error}>`）传入 logger **上下文**位置；`logger.ts:268-274` 用模板字面量渲染上下文值 → `[object Object]`。
- adoption 期间 `database is locked`：adoption 自己的连接（`WorktreeAdoption.ts:187`）与启动时迁移写入者竞争，因为它在 `worker-service.ts:480` 被触发，**早于** `:507` 的 `dbManager.initialize()`；WAL 单写者 + 5 秒 busy_timeout（`connection.ts:4,:44`）。

**CLAUDE.md 交付（#3381）：**
- 泄漏载体是 **marketplace git clone**：`known_marketplaces.json` 显示 `source: github, repo: thedotmack/claude-mem` 克隆到 `~/.claude/plugins/marketplaces/thedotmack/` — 每个被跟踪文件都会交付，包括根 `CLAUDE.md`（字节一致，shasum 验证）、`docs/`、`plans/`。插件**缓存**（`~/.claude/plugins/cache/…/13.12.3/`）是干净的，因为 `marketplace.json:9-16` 设置插件 `source: "./plugin"`。
- #2537 修复是 `.npmignore:9-14`（`/CLAUDE.md` 等）— 它只管辖 npm tarball，这就是 GitHub 源安装绕过它的原因（#3359）。仓库内无任何 "#2688" 修复的证据。
- 维护者 rsync（`scripts/sync-marketplace.cjs:77-80`）仅按 `.gitignore` 过滤，也会复制 CLAUDE.md。
- 根 `CLAUDE.md` 中仅维护者可见的章节：`## Local Status Notes`（第 35-37 行）和 `## Daily Maintenance`（第 39-47 行，自主升级+提交指令）。第 1-33 行（Build、File Locations、Requirements、Documentation、Important）是合法的贡献者内容。
- `.gitignore:43` 已列出 `CONTRIB_NOTES.md`（未跟踪 → 永不交付）。`plugin/` 没有自己的 CLAUDE.md（只有功能模式模板 `plugin/modes/law-study-CLAUDE.md` — 不要动它）。

### 反模式（全仓库，所有阶段）

- 任何地方都不要添加笼统的 `try/catch`-并继续。容忍**特定**领域状态（关闭时 `ERR_SERVER_NOT_RUNNING`、kill 时 `ESRCH`）并显式检查 code 是正确的；吞掉未知错误则不是。
- 不要通过在 SQL 查询中前缀匹配来修复 #3379 — 那会永远容忍畸形数据。修复写入方 + 回填。
- 不要让后台初始化在失败时"照样就绪" — `worker-service.ts:668-670` 的 fail-fast 门控保留。
- 不要在阶段 3 删除孤立的 observation 行 — 孤立行仍是注入所服务的用户数据；修复父侧。
- 不要臆造 logger API；logger 签名为 `(component, message, context?, data?)`，见 `src/utils/logger.ts`。

---

## 阶段 1 — 关闭流程必须在未监听的 server 句柄下存活（#3380）

### 要实现的内容

1. **`src/services/infrastructure/GracefulShutdown.ts`（`closeHttpServer`，第 60-69 行）：** 复制 `ServerService.ts:230-236` 的容忍模式 — 在 `server.close` 回调中，将 `err.code === 'ERR_SERVER_NOT_RUNNING'` 视为成功（resolve，并 `logger.warn` 镜像 ServerService 的 "Server was already stopped when close was requested"）；其他一律 reject。这是核心修复：已关闭的 server 不得中止其余清理（sessions、MCP、chroma、db、supervisor）。
2. **`src/services/server/Server.ts`（`listen`，第 140-157 行）：** 仅在 `onListening` 处理器中赋值 `this.server`；在 `onError` 中，保持 `this.server` 为 null（或清除它），使失败的绑定永不留下未监听句柄供关闭流程踩坑。其余 promise 语义保持不变。

### 文档参考

- 复制自：`src/server/runtime/ServerService.ts:227-238`（确切的 catch 形状）、`src/services/server/Server.ts:159-178`（句柄置 null 纪律）。
- 正在修复的出错即跳过后果：`GracefulShutdown.ts:33-36` + `worker-shutdown.ts:96-113`。

### 验证清单

- [ ] `tests/infrastructure/graceful-shutdown.test.ts` 新增测试：`mockServer.close` 以 `code = 'ERR_SERVER_NOT_RUNNING'` 的错误调用回调 → `performGracefulShutdown` **resolve**，且后续步骤（sessionManager.shutdownAll、dbManager.close、supervisor stop）仍全部被调用。第二个测试：close 以不同 code 出错 → 仍 reject。
- [ ] `tests/server/server.test.ts` 新增/更新测试：`listen` reject 后（EADDRINUSE 模式已在 `:113-127`），`getHttpServer()` 返回 `null`。
- [ ] `bun test tests/infrastructure/ tests/server/ tests/services/worker-shutdown-sequence.test.ts` 通过。
- [ ] Grep 守卫：`grep -n "ERR_SERVER_NOT_RUNNING" src/services/infrastructure/GracefulShutdown.ts` 返回显式 code 检查（不是裸 catch）。

### 反模式守卫

- 仅容忍 `ERR_SERVER_NOT_RUNNING`；不要扩大到通用 catch。
- 不要重排 `performGracefulShutdown` 步骤，也不要添加每步 try/catch "照样继续"包装 — 修复点是第 1 步不再误失败。
- 不要动重启后继/kill 逻辑（13.12.3 已修复）。

---

## 阶段 2 — 概念标签：写入时规范化、收紧生产者、回填（#3379）

### 要实现的内容

1. **在解析边界规范化 — `src/sdk/parser.ts:118`。** 扩展现有概念清理行：对每个提取的概念，在第一个 `:` 处截断并 trim（`"gotcha: WASM…"` → `"gotcha"`），然后应用现有 `!== finalType` 过滤并丢弃空值。不要按模式列表过滤（新标签目前会存储；保留该行为）。
2. **收紧生产者提示** — 在四个带 `observation_concepts` 的模式 JSON 中（主要是 `plugin/modes/code.json`；在 `email-investigation.json`、`law-study.json`、`meme-tokens.json` 中镜像），修改 `prompts.concept_guidance`，添加显式输出规则，例如："每个 `<concept>` 元素必须只包含裸关键词（如 `gotcha`）— 绝不要描述、句子或冒号。" 保留关键词列表；描述留在 guidance 中，规则禁止回显它们。
3. **一次性回填 — `src/services/sqlite/SessionStore.ts` 新增迁移 v49。** 复制 `addSessionCustomTitleColumn`（`:1537-1552`）的版本门控形状：若 `schema_versions` 有 49，返回；否则运行一次 UPDATE 重写每个 concepts 数组元素，在第一个 `:` 处截断：
   ```sql
   UPDATE observations
   SET concepts = (
     SELECT json_group_array(
       CASE WHEN instr(value, ':') > 0
            THEN trim(substr(value, 1, instr(value, ':') - 1))
```