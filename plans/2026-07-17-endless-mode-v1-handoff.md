# 交接文档：在新的 claude-mem worktree 中实现 Endless Mode v1

由上一会话于 2026-07-17 编写。本文档自包含：说明要构建什么、规格文档在哪、如何搭建 worktree，以及研究阶段产出的所有经代码核实的事实，你无需重新发现。

## 任务

在 claude-mem 中实现 **Endless Mode v1**（“the bottle, simply”）：当 Claude Code 自动压缩或恢复会话时，claude-mem 渲染一个“bottle”——由 transcript JSONL 和 observations DB 合并而成的会话 markdown 重建——并注入一个小指针，让压缩后的模型读取它并带着完整上下文继续，而不是患上压缩失忆症。

## 第 0 步 — Worktree 搭建

主检出：`/Users/alexnewman/Scripts/claude-mem`（分支 `main`，v13.11.0，当前**落后 origin/main 1 个提交**——先 fetch）。

```bash
cd /Users/alexnewman/Scripts/claude-mem
git fetch origin
git worktree add ../claude-mem-endless-mode-v1 -b feat/endless-mode-v1 origin/main
```

**关键：** 计划文件在主检出中是未跟踪的——新 worktree 不会包含它们。把它们复制进去（或从下面的绝对路径读取）：

```bash
cp /Users/alexnewman/Scripts/claude-mem/plans/2026-07-17-endless-mode-v1.md <worktree>/plans/
cp /Users/alexnewman/Scripts/claude-mem/plans/2026-07-16-endless-mode-message-in-a-bottle.md <worktree>/plans/
```

## 第 1 步 — 必读材料

1. **规格文档（据此构建）：** `/Users/alexnewman/Scripts/claude-mem/plans/2026-07-17-endless-mode-v1.md` — 自包含，约 10 个编号章节，包括机制入门（§4）、要构建什么（§5）、bottle 格式（§6）、注入文本（§7）、刻意不做的非目标（§8）、测试（§9）。无需预先阅读。
2. **可选背景：** `/Users/alexnewman/Scripts/claude-mem/plans/2026-07-16-endless-mode-message-in-a-bottle.md` — rev 2 研究文档，为下文每项论断提供 file:line 凭据。已被 v1 计划取代，但需要凭据时有用。
3. 仓库中的 `docs/architecture-overview.md` — 系统地图不错，但见下文“过时文档”警告。

## 核心思路（v1 为何很小）

压缩只摧毁**上下文窗口**。两个规范数据源——磁盘上的 transcript JSONL 和 observations SQLite DB——在爆炸半径之外且不变。所以压缩前无需保护任何东西：没有 PreCompact hook，没有 seal/drain 机制，没有压缩前状态捕获。bottle 是事后按需渲染的**派生视图**：

```
bottle = merge_by_position(transcript.jsonl, observations.db)
```

用户和助手消息逐字保留；工具流量替换为一行 observation 引用（带 `#id` 便于搜索）。比原始 transcript 小 10–20 倍。整体渲染，原子写入（临时文件 + 重命名），从不追加，从不是事实来源。

## 要构建什么（4 个组件，来自规格 §5）

1. **`BottleRenderer`** — 新的 worker 服务。合并 transcript + observations，通过原子写入输出到 `~/.claude-mem/bottles/<contentSessionId>.md`。必须对每条渲染的消息应用 `stripMemoryTags`（见下文隐私说明）。
2. **`POST /api/sessions/render-bottle`** — `SessionRoutes.setupRoutes()` 中的新 worker 路由。
3. **SessionStart hook 按 `source` 分支** — source 为 `startup | compact | resume | clear`。在 `compact`/`resume` 时：POST render-bottle，`wait:true`，10s 超时，然后注入指针；若 worker 宕机则回退到现有 timeline 上下文。在 `clear` 时：归档 bottle，不注入任何内容。
4. **可选：render-on-Stop** — 修复/实时日记环节。在 `src/cli/handlers/summarize.ts` 中现有 summarize 调用之后触发。必须使用 await 的 queue-and-return，**而非**未 await 的 POST（未 await 的 POST 会在 `process.exit` 时死亡）。

另外：在 `SettingsDefaultsManager` 中新增设置 `CLAUDE_MEM_ENDLESS_MODE_ENABLED`，默认 `'true'`（见下文设置说明），在 worker 侧强制执行。

## 经代码核实的事实（来自研究阶段——信任这些，它们已对照源码核实）

**会话身份 / 数据访问**
- `contentSessionId`（Claude Code 的 ID）在压缩前后不变；`memorySessionId`（SDK agent 的）每次 worker 重启都会变。单一 join 路径可到达一切：`content_session_id → sdk_sessions → memory_session_id → observations + session_summaries`。外键为 `ON UPDATE CASCADE`，因此多段会话自动级联。**始终使用 sdk_sessions join**——一个社区迁移（v36–38，不在本仓库）添加了部分填充的 `observations.content_session_id` 列；绝不要依赖它。
- `user_prompts` 以 `content_session_id` 为键存储逐字用户提示，截断于 4000 字符（`src/services/sqlite/prompt-storage.ts` 中的 `MAX_STORED_PROMPT_CHARS`）。持久化于 SQLite。
- `session_summaries` 存储结构化字段（request/investigated/learned/completed/next_steps）——**不是**逐字助手文本。**绝不要在 `**Assistant**` 标题下渲染摘要**——恢复后的模型会“记得”说过它从未说过的话。明确标记来源。

**降级模式（约 97% 的会话 transcript 已被修剪）**
- 阶梯：完整 bottle（transcript + observations）→ 重建（逐字用户提示 + observation 行 + 摘要，明确标记来源，无逐字助手轮次）→ 若 transcript 和存储数据都不存在，渲染器报告失败，hook 回退到当前的 timeline 上下文行为。

**队列 / 缓冲现实（rev 2 中发现并修复了两个规格 bug——不要回归）**
- 待处理队列**仅在内存中**：`SessionMessageBuffer`（数组 Map + 每会话 `EventEmitter`）取代了持久化 SQLite `pending_messages` 表。`SessionStore.ts:951` 的 `CREATE TABLE` 是迁移遗留；源码中零个 `INSERT INTO pending_messages`。
- `clearPendingForSession`（`SessionManager.ts:207`）是**死代码**——零调用点。真正的“agent 处理完毕”信号是 `confirmClaimedMessages`（`SessionManager.ts:219-231`），在 `storeObservations()` 提交后由 `ResponseProcessor.ts` 调用。
- `GeneratorExitHandler` 在非配额 generator 退出时丢弃所有缓冲工作（缓冲通过 `removeSessionImmediate` 释放）。缓冲的 observations 会**丢失**，而非延迟。（主要是 v2/honest-tail 关注点——v1 不 seal——但不要写声称待处理 observations“稍后会落地”的文案。）
- 排空谓词，若你需要：`SessionMessageBuffer.getPendingCount(sessionDbId) === 0`。

**隐私契约**
- `stripMemoryTags` 必须对**每条**渲染的消息运行。逐字 transcript 渲染路径是系统中唯一可能绕过 `<private>` 契约的路径。这是硬性要求，且有测试覆盖。

**Hooks**
- 注册于 `plugin/hooks/hooks.json`：Setup、SessionStart（`startup|clear|compact`——超时 60，同步）、UserPromptSubmit、PostToolUse（`*`，异步，120s）、PreToolUse（Read）、Stop（异步，120s）。**没有 PreCompact hook**——v1 刻意不添加。
- Hook 处理器位于 `src/cli/handlers/`，是纯函数（返回 `HookResult`，无 stdout/process.exit），注册于 `src/cli/handlers/index.ts`。传输错误总是退出 0（绝不阻塞 Claude Code）；只有客户端 bug 退出 2。
- **过时文档警告：** `docs/architecture-overview.md` 列出了一个 `SessionEnd | session-complete` hook，它在 `hooks.json` 或 `handlers/index.ts` 中不存在。

**Worker / HTTP 约定**
- 新路由放在 `SessionRoutes.setupRoutes()`（`src/services/worker/http/routes/SessionRoutes.ts`），用 `BaseRouteHandler` 的 `this.wrapHandler()` 包装，zod `validateBody`，依赖通过构造函数 DI（`this.sessionManager`、`this.dbManager` 等）。
- 即发即忘的惯用模式：`void (async () => { ... })()`（见 `broadcastProcessingStatus`、`worker-service.ts`）。但记住：从 hook 进程发起时，未 await 的 POST 会在退出时死亡——hook 必须 await。

**设置**
- 两套独立的设置系统。功能开关放在 **`SettingsDefaultsManager`**（`src/shared/SettingsDefaultsManager.ts`，约 100 个 `CLAUDE_MEM_*` 键，扁平 JSON 位于 `~/.claude-mem/settings.json`，字符串 `'true'`/`'false'`，每次调用通过 `loadFromFile(USER_SETTINGS_PATH)` 重新读取——无需重启）。添加一个约需 3 行：interface 键、DEFAULTS 条目、读取点。不要碰 `SettingsManager`（`src/services/worker/SettingsManager.ts`）——那只是 SQLite 支撑的 UI 查看器状态。

**注入（规格 §7——两个承重细节）**
- 它是**指针**（约 20 行），不是 bottle 载荷。模型通过自己的 Read 工具读取 bottle；结果作为工具结果落地，而*下一次*压缩会最先驱逐它，因此 bottle 不会跨段累积。设计上自清理。
- **“do not repeat”** 行是承重的：没有它，模型会可靠地重发其最终助手消息或重做已完成的工作。保留它。

## v1 刻意不构建（规格 §8——范围守卫）

PreCompact hook、seal/drain 端点、`session_bottles` 表、transcript 字节游标 / 渲染互斥锁、`tool_use_id` 锚列 / 线程化。全部推迟到 v2，理由见规格。render-bottle 的双重触发无害（幂等文件、指针注入），因此无需互斥锁。

## 测试（规格 §9——7 个用例）

轮次中途压缩；从 bottle 中剥离 `<private>` 内容；降级渲染（无 transcript）；worker 宕机回退到 timeline 上下文；`/clear` 归档 bottle；双重触发幂等性；端到端 compact→inject→continue。测试布局：`bun test`，套件位于 `tests/`（`tests/sqlite/`、`tests/worker/` 等）。

## 开发工作流

```bash
npm run typecheck        # tsc --noEmit (root + viewer)
bun test                 # or npm run test:sqlite / test:server etc.
npm run build            # sync manifests + build hooks + lockfile
npm run dev              # build + sync marketplace + restart live worker — touches the LIVE plugin; use deliberately
npm run worker:logs      # today's worker log
```

注意：`npm run dev` / `build-and-sync` 会重启本机上的**实时** claude-mem worker（即记录你自己会话的那个）。迭代时优先用 `typecheck` + `bun test`；只有打算 dogfood 时才同步到实时插件。

## 历史记录在哪

所有研究都在 claude-mem 记忆中（项目 `context-mode`，2026 年 7 月 16–17 日）。需要凭据时的关键 observation ID：`102441`（rev 2 规格，经代码核实，bug 列表）、`102442`/`102443`（v1 计划 + 扩展）、`102432`–`102439`（逐文件代码调查：buffer、events、hooks、settings、ingestion path、generator exit、user_prompts、routes）。通过 `get_observations([ids])` 或 mem-search 技能获取。