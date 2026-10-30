<!-- Plan 26 尚无 GitHub plan 主文档。按修复形态最接近的现有主文档是 plan-23
     （#3611，宿主集成契约：Hook 注册及其处理程序）。若日后开设主文档，
     请将本文档镜像过去，并补上下方 Tracker 行（oh-my-issues 健康检查）。 -->

> **Tracker：** 暂无 · **设计文档：** `plans/26-telegram-session-wrapups.md` · **分支：** `work/telegram-wrapups`（worktree `.claude/worktrees/telegram-wrapups`，head = `origin/main` `07ba05ae6`） · **日期：** 2026-09-12

# [plan-26] Telegram 会话总结 — 每会话一条消息，取自 Stop Hook 摘要，按项目路由，绝不重复

Alex 的缺陷报告，原文（2026-09-12）：*"the 'telegram session wrap-ups' are not derived of the stop hook summary, they're coming at the end of all turns. ALSO NOT scoped that way"。*

来源审计：`~/.claude/context/endless/TELEGRAM-WRAPUPS-2026-09-12.md`（只读，2026-09-12）。其中每条引用都已对照本 worktree 复核；多数已偏移 60 到 200 行，还有一条指向了错误的函数。本文档中的行号是复核后的行号。

## 缺陷

`notifyTelegram()` 由 PostToolUse 路径按每个存储的**观察批次**调用一次，从不从摘要路径调用，且所有消息都发往同一个全局聊天。没有会话结束触发器，没有按项目路由，也没有已发送记录，因此繁忙会话会产生大量 Telegram 消息，却没有一条是总结。

## 期望行为（本计划的验收标准）

1. 每个会话恰好一条 Telegram 总结，在会话结束时投递。
2. 文本取自该会话最新存储的摘要，即 Stop Hook 排入的那条。不新增模型调用。
3. 发往该会话所属项目配置的路由（若配置了路由键，则按路由键）。
4. 未知路由拒绝并记日志，绝不回退到全局聊天。
5. 绝不重复，包括跨 worker 重启和重复的 SessionEnd 投递。
6. 按观察告警作为独立功能保留，默认**关闭**。

## 子项

无。Alex 上述口头报告是唯一子项。

## 相关 PR

无进行中的 PR。`plans/12-provider-and-extensibility-roadmap.md:53`（#3845 Brainbeats）计划复用 `TelegramNotifier` 实现观察触发的 webhook；本计划将该观察路径保留在新默认关闭开关之后，因此 #3845 不受影响。

---

## 阶段 0 — 汇总发现（必读；不要重新推导）

以下每条事实都带有本 worktree 的 `file:line`，或标注为 UNVERIFIABLE 并给出验证方法。若某阶段的指令与这里的事实冲突，停下来重读文件；不要猜。

### 0.1 当前消息从何而来

| 事实 | 证据 |
|---|---|
| Telegram 发送器是 `notifyTelegram(input)`，一个普通异步函数，被导入响应处理器。 | `src/services/worker/agents/ResponseProcessor.ts:13` |
| 它以 fire-and-forget 方式调用，传入当前响应的观察批次，调用点在批次确认之后、观察同步之前。摘要从不传入。 | `src/services/worker/agents/ResponseProcessor.ts:623-628`（调用），`:630-636`（紧随其后的 Grok Bot 同类调用） |
| 摘要路径是独立的：`normalizeSummaryForStorage` 构建 `{request, investigated, learned, completed, next_steps, notes}`，`syncAndBroadcastSummary` 同步 Chroma 并广播 SSE。两者都不涉及 Telegram。 | `src/services/worker/agents/ResponseProcessor.ts:661-682`、`:828-890` |
| 通知器每次调用都加载设置，除非 `CLAUDE_MEM_TELEGRAM_ENABLED === 'true'` 且存在 token 和 chat id，否则直接退出；然后为每条类型或概念匹配配置触发器的观察发送一条 MarkdownV2 消息。失败记录在 `'TELEGRAM'` 下。 | `src/services/integrations/TelegramNotifier.ts:67-109` |
| 传输层是 `postOne(botToken, chatId, text)`：`fetch` POST 到 `https://api.telegram.org/bot<token>/sendMessage`，载荷 `{chat_id, text, parse_mode: 'MarkdownV2'}`，非 2xx 时抛错。`escapeMarkdownV2` 和 `splitCsv` 是模块私有辅助函数。 | `src/services/integrations/TelegramNotifier.ts:14-32`、`:49-65` |
| 消息模板为 emoji、类型、标题、副标题、项目和观察编号。`memorySessionId` 被接收但不渲染。 | `src/services/integrations/TelegramNotifier.ts:34-47` |
| 上游触发器是 PostToolUse，运行 `hook claude-code observation`（异步，120 秒）。 | `plugin/hooks/hooks.json:48-61` |

### 0.2 Stop Hook 做什么，以及为何是按轮次

| 事实 | 证据 |
|---|---|
| Stop 运行 `hook claude-code summarize`（异步，120 秒）。插件中注册的事件恰好是 `Setup, SessionStart, UserPromptSubmit, PostToolUse, PreToolUse, Stop`。**没有 SessionEnd 注册。** | `plugin/hooks/hooks.json:76-88` 及整个文件 |
| summarize 处理程序取最后一条 assistant 消息（或从 transcript 读取），POST `/api/sessions/summarize`，载荷 `{contentSessionId, last_assistant_message, platformSource, observedModel, observedBilling}`。 | `src/cli/handlers/summarize.ts:76-114`、`:150-160` |
| 处理程序注册键为 `context, session-init, observation, summarize, user-message, file-edit, file-context`。`src/cli/handlers/` 中不存在 `session-end` 处理程序文件。 | `src/cli/handlers/index.ts:13-20`、`:22-30` |
| 路由用 `summarizeByClaudeIdSchema` 校验，用 `createSDKSession` upsert 会话，然后调用 `sessionManager.queueSummarize(sessionDbId, cleaned)`。仅注册了三个 POST 路由：`/api/sessions/init`、`/api/sessions/observations`、`/api/sessions/summarize`。 | `src/services/worker/http/routes/SessionRoutes.ts:569-576`、`:623-663`、`:522-537` |
| `queueSummarize` 将 `PendingMessage {type:'summarize', last_assistant_message}` 入队到内存中的每会话 FIFO 缓冲区。缓冲区不持久化。 | `src/services/worker/SessionManager.ts:211-232`；`src/services/worker/SessionMessageBuffer.ts:21-40`（内存中文档注释）、`:56-71`（入队）、`:210-218`（FIFO 取用） |
| `PendingMessage.type` 是联合类型 `'observation' | 'summarize'`。 | `src/services/worker-types.ts:87-98` |
| 消费者：`ClaudeProvider` 将摘要提示注入实时 SDK 查询，随后用 `activeResponseContext.current` 调用 `processAgentResponse`；`OpenAICompatibleProvider` 构建提示、查询，并内联调用 `processAgentResponse`。 | `src/services/worker/ClaudeProvider.ts:762-770`、`:472`、`:554`；`src/services/worker/OpenAICompatibleProvider.ts:206-210`、`:316-364` |
| `ResponseContext` 是 `{project, promptNumber, pendingAgentId, pendingAgentType}`，从 `ActiveSession` 快照而来。除通过 `ActiveSession` 字段外，没有任何按消息的数据能到达存储点。 | `src/services/worker/agents/ResponseProcessor.ts:266-280` |
| Claude Code 文档：Stop 是**按轮次**事件（"per turn: UserPromptSubmit, Stop, StopFailure"）；SessionEnd 是**按会话**，生命周期描述中 Stop 在 SessionEnd 之前。 | https://code.claude.com/docs/en/hooks.md（2026-09-12 获取） |

所以"所有轮次结束时"正是 Stop 的字面含义。总结需要 SessionEnd。

### 0.3 SessionEnd Hook 契约（来自文档，2026-09-12 获取）

| 事实 | 证据 |
|---|---|
| 输入 JSON：通用字段（`session_id`、`transcript_path`、`cwd`、`hook_event_name`、…）加上 `reason` ∈ `clear, resume, logout, prompt_input_exit, other`。 | https://code.claude.com/docs/en/hooks.md，"SessionEnd input" |
| 不能阻塞；JSON 输出被丢弃。 | 同上，"SessionEnd hooks have no decision control" |
| **默认预算为 1.5 秒。** *设置文件*中的每 Hook `timeout` 可将其提高到最多 60 秒，但"插件提供的 Hook 上设置的超时不会提高预算"。环境变量覆盖：`CLAUDE_CODE_SESSIONEND_HOOKS_TIMEOUT_MS`。 | 同上，SessionEnd 超时段落 |
| SessionEnd 是否支持 `async: true`：文档未说明。Ctrl+C 或 `claude -p` 退出是否触发：文档未说明（若触发则归入 `other`）。 | 同上 |
| 插件注册形态与其他事件相同；matcher 按 `reason` 过滤；`${CLAUDE_PLUGIN_ROOT}` 可用。 | https://code.claude.com/docs/en/plugins-reference.md |

结论：SessionEnd Hook 命令必须是快速 POST 后退出。阶段 2 测量 `worker-service.cjs hook claude-code session-end` 的冷启动并记录数值。实际工作全部由 worker 完成。

### 0.4 可依托的持久化

| 事实 | 证据 |
|---|---|
| 会话标识：`sdk_sessions(content_session_id, memory_session_id UNIQUE, project, platform_source, status …)`。 | `src/services/sqlite/SessionStore.ts:1021-1035` |
| 按 `(platform_source, content_session_id)` 查找是 `createSDKSession` 内部的 SELECT。没有独立的按 content-id 查找方法；`resolvePromptSessionDbId` 是最接近的辅助函数。 | `src/services/sqlite/SessionStore.ts:2744-2749`、`:227-255` |
| `getSessionById(id)` 返回 `content_session_id, memory_session_id, project, platform_source, user_prompt, custom_title, status, observed_model, observed_billing`。 | `src/services/sqlite/SessionStore.ts:2640-2651` |
| **最新摘要已有现成方法**：`getSummaryForSession(memorySessionId, platformSource?)` 返回最新的 `session_summaries` 行（`request … notes, prompt_number, created_at, created_at_epoch`）。其 SELECT 不含 `id`。 | `src/services/sqlite/SessionStore.ts:2629-2646` |
| `storeSummary(memorySessionId, project, summary, promptNumber?, discoveryTokens?, overrideTimestampEpoch?)` 返回 `{id, createdAtEpoch}`。`session_summaries` 没有 `platform_source` 列。 | `src/services/sqlite/SessionStore.ts:2900-2947`、`:1678-1696` |
| 迁移模式：构造函数调用列表以 `this.ensureToolUsesTable()` 结尾；每个 `ensureXxx` 执行 `CREATE TABLE IF NOT EXISTS` + `CREATE INDEX IF NOT EXISTS`，然后 `INSERT OR IGNORE INTO schema_versions (version, applied_at) VALUES (?, ?)`。最新版本为 **51**。UNIQUE 内联写在 CREATE 中。 | `src/services/sqlite/SessionStore.ts:164-212`、`:1909-1912`；`src/services/sqlite/tool-uses.ts:171-206`（复制模板，`UNIQUE(...)` 在 `:194`） |
| `SessionStore` 中不存在发送台账、`sent_at` 或幂等键。 | grep，2026-09-12 |
| 规范项目名是 `getProjectName(cwd, platform)`（git 顶层目录名，回退 `'unknown-project'`）；`getProjectContext()` 为 worktree 组合 `parent/child`。 | `src/utils/project-name.ts:53-88`、`:97-125` |
| worktree 会话中两个字符串哪个落入 `sdk_sessions.project`：**此处无法验证**。在阶段 1 通过阅读 `src/cli/handlers/session-init.ts` 和 `/api/sessions/init` 处理程序来证明，然后检查 `sqlite3 ~/.claude-mem/claude-mem.db "select distinct project from sdk_sessions where project like '%/%' limit 5"`。下方路由解析器同时处理两种形态。 |
| `platform_source` 是规范化的自由字符串：默认 `'claude'`，`codex`、`cursor`，其他原样透传。 | `src/shared/platform-source.ts:1-19` |

### 0.5 设置管道

| 事实 | 证据 |
|---|---|
| Telegram 键：接口 `:123-127`，默认值 `:253-257`（`ENABLED 'true'`、`BOT_TOKEN ''`、`CHAT_ID ''`、`TRIGGER_TYPES 'security_alert,sensitive'`、`TRIGGER_CONCEPTS ''`）。旧触发器迁移在 `:369-383`。 | `src/shared/SettingsDefaultsManager.ts` |
| 新增键 = 接口字段 + `DEFAULTS` 条目。`loadFromFile(path, applyEnvOverrides = true)` 对 `DEFAULTS` 中每个键做往返；环境变量按键覆盖。 | `src/shared/SettingsDefaultsManager.ts:22-153`、`:158-286`、`:316-324`、`:328-395` |
| JSON 值设置已存在（`CLAUDE_MEM_FOLDER_MD_EXCLUDE: '[]'`、`…_SKELETON_DENYLIST: '[]'`），因此 JSON 路由映射符合项目惯例。 | `src/shared/SettingsDefaultsManager.ts:213-214` |
| 设置 API 的密钥掩码允许列表包含 `CLAUDE_MEM_TELEGRAM_BOT_TOKEN`。 | `src/services/worker/http/routes/SettingsRoutes.ts:30` |
| 模式文件不含 Telegram 字段；路由完全由设置驱动。 | grep `plugin/modes/*.json`，2026-09-12 |

### 0.6 测试与工具

| 事实 | 证据 |
|---|---|
| 运行器是 `bun test`，预加载 `tests/preload.ts`。脚本：`test`、`test:sqlite`、`test:agents`、`test:search`、`test:context`、`test:infra`、`test:server`。 | `bunfig.toml`；`package.json:99-105` |
| 临时数据库模式：`beforeEach` 中 `new SessionStore(':memory:')`，`afterEach` 中 `store.close()`，辅助函数 `createSDKSession` + `updateMemorySessionId`。 | `tests/sqlite/session-store-summaries.test.ts:1-24` |
| `global.fetch` mock 模式并恢复。 | `tests/shared/worker-utils-fetch-verbose.test.ts:28-46`、`:205`、`:216-217` |
| 响应处理器测试框架：对 `worker-service.js`、`worker-utils.js`、`ModeManager.js` 做 `mock.module` 并快照/恢复；`beforeEach` 中 logger spy。 | `tests/worker/agents/response-processor.test.ts:12-24`、`:95-103`、`:125-134` |
| CLI 处理程序测试框架：对 `SettingsDefaultsManager.js`、`hook-settings.js`、`transcript-parser.js`、`observed-billing.js` 做 `mock.module`；`postedBody()` 辅助函数读取最后一次 worker POST。 | `tests/cli/handlers/summarize-observed-fields.test.ts:8-58`、`:79-91`、`:103-108` |
| 可参照的同类集成及其单元测试：`GrokBotAwarenessPusher`（从设置 `loadXxxConfig`，纯函数 `formatXxx`，整体函数体加保护）。 | `src/services/integrations/GrokBotAwarenessPusher.ts:18-52`、`:151-186`；`tests/integrations/grok-bot-awareness-pusher.test.ts` |
| **目前没有测试涉及 `TelegramNotifier`。** | grep `tests/`，2026-09-12 |
| 基线：`origin/main` 在 2026-09-12 有 4 个既有失败，包括一个 30 秒 worktree 测试超时（claude-mem 观察 #124832）。阶段 1 前重新运行 `bun test tests`，将失败名称粘贴到阶段 1 提交正文，以便后续阶段区分旧红与新红。 | 记忆，需重新验证 |

### 0.7 允许的 API（只用这些；其他先读文件）

- `SettingsDefaultsManager.loadFromFile(USER_SETTINGS_PATH)` 及 0.5 中列出的 `SettingsDefaults` 键。
- `SessionStore.getSessionById`、`getSummaryForSession`、`createSDKSession` 的 SELECT（复制，不要在 session-end 路径调用 `createSDKSession`：它会 upsert）。
- `SessionManager.getSession(sessionDbId)`、