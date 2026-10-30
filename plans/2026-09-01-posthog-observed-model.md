# PostHog：上报观测模型、来源与计费类型

**分支：** `posthog-observed-model`（worktree 位于本文件所在仓库根目录）。**基线：** `main`。

## 目标

PostHog 目前只上报*观测者*模型（claude-mem 用于写入观测的模型：`model` / `top_model`）。我们还需要按被观测会话了解：

1. **`observed_model`** —— 用户在 IDE 会话中运行的模型（例如 `claude-fable-5-1`）。
2. **`ide`**（每轮已存在，但汇总时被丢弃）—— 来源平台（`claude`、`codex`、`cursor` 等）。
3. **`observed_billing`** —— Max / Pro / Team / API 额度 / Bedrock / Vertex / Foundry / 未知，在可判定时上报。

这些是**新增的独立**属性。不要改动 `model` / `top_model` / `provider` 的语义——它们仍属于观测者侧。

## 阶段 0：文档调研（已完成——汇总结论）

### 0.1 观测模型可从何处获取

| 来源 | 事实 | 证据 |
|---|---|---|
| Claude Code hook stdin | `model`（规范 id 字符串）**仅在 SessionStart** 出现；UserPromptSubmit / PostToolUse / Stop 中不存在。 | https://code.claude.com/docs/en/hooks.md（SessionStart 输入表） |
| 转录 JSONL | 每个 assistant 条目都带有 `message.model`（已在真实转录中验证：`"model":"claude-fable-5-1"`），同时还有 `message.role`、`message.content`。行角色为 `line.type`（Claude Code）或 `line.role`（Cursor）。 | `~/.claude/projects/…/*.jsonl` 下的真实转录；`src/shared/transcript-parser.ts:61` |
| 我们的 hook 类型 | `NormalizedHookInput` 已声明 `model?`、`permissionMode?`、`sessionSource?`（`src/cli/types.ts:13-15`）。只有 codex 适配器填充它们（`src/cli/adapters/codex.ts:99-101`）。claude-code 适配器丢弃了它们（`src/cli/adapters/claude-code.ts:15-25`）。`src/` 中没有任何下游消费者读取 `input.model`。 | grep |
| Stop hook 路径 | `summarize` 处理器已读取转录：`src/cli/handlers/summarize.ts:93` 处的 `extractLastMessage(transcriptPath, 'assistant', true)`，然后向 `/api/sessions/summarize` POST `{ contentSessionId, last_assistant_message, agentId?, platformSource }`（`summarize.ts:138-147`）。 | 阅读 |

**决策：** 在 Stop hook（`summarize` 处理器）中从转录读取观测模型。它与平台无关，无需新增 hook 接线，且每轮刷新（覆盖 `/model` 切换）。SessionStart 的 `model` 字段*不*使用（它需要经由 `context` 处理器新增传输，而该处理器只 GET `/api/context/inject`）。

### 0.2 计费 / 套餐可从何处获取

| 信号 | 位置 | 证据 |
|---|---|---|
| 订阅账户 | `~/.claude.json` → `oauthAccount.organizationType`（观测到：`"claude_max"`）、`oauthAccount.billingType`（`"stripe_subscription"`）、`oauthAccount.organizationRateLimitTier`（`"default_claude_max_20x"`）。这些键的顶层副本在本机为 `null`——应从 `oauthAccount` 读取。 | `jq '.oauthAccount' ~/.claude.json`（2026-09-01 验证） |
| API key 计费 | **hook 进程环境**中的 `ANTHROPIC_API_KEY` / `ANTHROPIC_AUTH_TOKEN`（hook 继承 Claude Code 的环境；worker 守护进程没有用户的 shell 环境）。当已登录用户同时导出 key 时，Claude Code 会请求批准，并将批准记录在 `~/.claude.json` 的 `customApiKeyResponses.approved`（key 的后 20 个字符）下。 | https://code.claude.com/docs/en/hooks.md（hook 执行环境）；`~/.claude.json` 键清单 |
| 云提供商 | `CLAUDE_CODE_USE_BEDROCK`、`CLAUDE_CODE_USE_VERTEX`、`CLAUDE_CODE_USE_FOUNDRY` 环境变量。 | https://code.claude.com/docs/en/settings-reference.md |
| 长期 OAuth token | `CLAUDE_CODE_OAUTH_TOKEN` 环境变量 → 未知层级的订阅。 | 同上 |
| 配置目录 | 设置 `CLAUDE_CONFIG_DIR` 时，`.claude.json` 位于 `$CLAUDE_CONFIG_DIR/.claude.json`，否则位于 `~/.claude.json`。现有常量：`src/shared/paths.ts:43` 中的 `CLAUDE_CONFIG_DIR`。 | 阅读 |

`~/.claude.json` 是 Claude Code 特有的 → 仅当 `input.platform === 'claude-code'` 时运行计费检测。

**决策：** 在 hook 进程（拥有环境变量）中、在同一个 `summarize` 处理器内计算计费，并与观测模型一起发送。

检测顺序（封闭枚举，低基数）：

```
1. CLAUDE_CODE_USE_BEDROCK 已设置（非空，不是 '0'/'false'）→ 'bedrock'
   CLAUDE_CODE_USE_VERTEX  → 'vertex'
   CLAUDE_CODE_USE_FOUNDRY → 'foundry'
2. key = ANTHROPIC_API_KEY || ANTHROPIC_AUTH_TOKEN（非空）
   若 key &&（无 oauthAccount || customApiKeyResponses.approved 包含 key.slice(-20)）→ 'api_key'
3. oauthAccount.organizationType 是字符串 → 去掉前导 'claude_' → 值
   （例如 'max'、'pro'、'team'、'enterprise'）；清洗为 /^[a-z0-9_]{1,40}$/，否则 'subscription'
4. oauthAccount 存在 || CLAUDE_CODE_OAUTH_TOKEN 已设置 → 'subscription'
5. 否则 → 'unknown'
```

### 0.3 遥测如何流转（属性要到达 PostHog 必须改什么）

| 事实 | 证据 |
|---|---|
| 每轮 `session_compressed` 记录**从不发送**；它们由 `computeSessionCompressedRollup`（`src/services/telemetry/buffer.ts:124-241`）聚合进 `observer_turn_rollup`。只有数值聚合和 `top_model` 保留；`ide`、`provider`、`hook` 被丢弃。文档（`docs/public/telemetry.mdx:132`）声称会携带 `provider`、`ide`、`hook`——目前不实。 | 阅读 |
| 每轮记录创建于：`src/services/worker/agents/ResponseProcessor.ts:490-511`（`compressionProps`，成功）、`src/services/worker/http/routes/SessionRoutes.ts:328-336`（错误）、`SessionRoutes.ts:374-381`（中止）。Claude 路径在 `src/services/worker/ClaudeProvider.ts:448-460` 和 `:469` 重新发出 `...pending`——展开暂存的 props，因此那里无需改动。 | 阅读 |
| 每个属性都必须在 `ALLOWED_PROPERTY_KEYS`（`src/services/telemetry/scrub.ts:8-184`）中，否则清洗器会静默丢弃。`ide`、`provider`、`model`、`top_model` 已在白名单中（`scrub.ts:30,73,162`）。 | 阅读 |
| 内存会话状态类型：`src/services/worker-types.ts:9-…` 中的 `ActiveSession`（有 `platformSource`、`lastModelId?`、`lastGeneratorSource?`）。在 `SessionManager.initializeSession`（`src/services/worker/SessionManager.ts:23-130`）中由 `dbManager.getSessionById(sessionDbId)`（`src/services/worker/DatabaseManager.ts:97-113` → `SessionStore.getSessionById`，`src/services/sqlite/SessionStore.ts:2321-2332`，行类型 `SdkSessionDetailRow` `SessionStore.ts:63-72`）构建。 | 阅读 |
| 汇总刷新点：`SessionManager.ts:260` 和 `:324`（`flushSession(sessionDbId, 'session_end')`），以及 `buffer.ts` 中的安全扫描和关闭。 | 阅读 |
| Summarize 路由：`handleSummarizeByClaudeId` `SessionRoutes.ts:504-539`；请求体 schema `summarizeByClaudeIdSchema` `SessionRoutes.ts:456-461`（`.passthrough()`）；它通过 `store.createSDKSession(contentSessionId, '', '', undefined, platformSource)`（`:514`）upsert 会话，然后 `queueSummarize` → `ensureGeneratorRunning`（`:532-534`）。 | 阅读 |
| SQLite 迁移模式：`SessionStore` 上的幂等私有方法，`PRAGMA table_info`、`ALTER TABLE … ADD COLUMN`、`INSERT OR IGNORE INTO schema_versions`，追加到构造函数链 `SessionStore.ts:102-122`。可直接复制的模板：`addObservationModelColumns` `SessionStore.ts:1688-1704`。现有最高版本戳：49（`normalizeConceptTags`）→ 使用 **50**（构造函数链并非按戳顺序；34 已是 `ensureUserPromptsSessionDbId`）。 | 阅读 |

### 0.4 允许的 API（仅这些——不要发明其他）

- 来自 `fs` 的 `readFileSync`、`existsSync`；来自 `path` 的 `join`；来自 `os` 的 `homedir`。
- `src/shared/transcript-parser.ts` 中的 `extractLastMessage` / `extractLastMessageFromJsonl` 模式（反向行扫描，try/catch 中逐行 `JSON.parse`，`line.type ?? line.role`）。
- `telemetryBuffer.record(event, sessionDbId, props)`（`buffer.ts:309`）。
- `SessionStore` 方法使用 `this.db.prepare(sql).run(...)` / `.get(...)`（bun:sqlite）。
- `sessionManager.getSession(sessionDbId)`（用于 `SessionRoutes.ts:621`）。
- 现有 `.passthrough()` schema 内的 Zod `z.string().optional()`。
- 日志：`logger.debug('HOOK', msg, ctx)` / `logger.warn(...)`。

### 0.5 要避免的反模式

- 不要把任何自由格式内容放进遥测 props（不要路径，不要提示词）。`observed_model` 是模型 id；`observed_billing` 是封闭枚举。
- 不要发送上述命名之外的 `~/.claude.json` 字段；绝不读取或记录 token。
- 不要复用 `observations.generated_by_model`、`session.lastModelId`、`model` 或 `top_model`——它们属于观测者侧。
- 不要在 PostToolUse（`observation`）处理器中读取转录——它每次工具调用都会触发；Stop 每轮一次。
- 除转录扫描中逐行 `JSON.parse`（现有、有文档的模式）外，不要为新正常路径代码添加 try/catch。错误必须暴露。
- 不要改动 `plugin/scripts/*.cjs`（构建产物在版本号提升时重建，不在功能 PR 中——见 `git log -- plugin/scripts/worker-service.cjs`）。
- `mock.module('../../../src/shared/transcript-parser.js', …)` 的测试（例如 `tests/cli/handlers/summarize-tag-stripping.test.ts:35-40`）必须扩展以导出新函数，否则处理器导入在调用时变为 `undefined`。
- 不要添加设置键；遥测同意仅通过环境变量/`telemetry.json`（`src/services/telemetry/consent.ts`）。

---

## 阶段 1：Hook 侧——提取观测模型 + 计费，在 Stop 时发送

### 1.1 `src/shared/transcript-parser.ts` —— 添加 `extractLastAssistantModel`

复制 `extractLastMessage`（`transcript-parser.ts:5-22`）的结构和 `extractLastMessageFromJsonl`（`:40-58`）的反向扫描：

```ts
export function extractLastAssistantModel(transcriptPath: string): string | undefined {
  if (!transcriptPath || !existsSync(transcriptPath)) return undefined;
  const content = readFileSync(transcriptPath, 'utf-8').trim();
  if (!content) return undefined;
  return extractLastAssistantModelFromJsonl(content);
}

export function extractLastAssistantModelFromJsonl(content: string): string | undefined {
  const lines = content.split('\n');
  for (let i = lines.length - 1; i >= 0; i--) {
    const rawLine = lines[i];
    if (!rawLine) continue;
    let line: any;
    try { line = JSON.parse(rawLine); } catch { continue; } // same tolerated condition as extractLastMessageFromJsonl
    if ((line.type ?? line.role) !== 'assistant') continue;
    const model = line.message?.model;
    if (typeof model === 'string' && model) return model;
  }
  return undefined;
}
```

### 1.2 新增 `src/shared/observed-billing.ts`

```ts
import { existsSync, readFileSync } from 'fs';
import { join } from 'path';
import { homedir } from 'os';

export type ObservedBilling = string; // closed set documented in docs/public/telemetry.mdx

export function claudeJsonPath(env: NodeJS.ProcessEnv = process.env): string {
  return env.CLAUDE_CONFIG_DIR ? join(env.CLAUDE_CONFIG_DIR, '.claude.json') : join(homedir(), '.claude.json');
}

export function detectObservedBilling(
  env: NodeJS.ProcessEnv = process.env,
  claudeJsonFile: string = claudeJsonPath(env),
): ObservedBilling { … order from §0.2 … }
```

- `isTruthyEnv(v)`：非空且不是 `'0'`/`'false'`（不区分大小写）。
- 用 `existsSync` + `readFileSync` + `JSON.parse` 读取 `.claude.json`。文件缺失 → 视为无账户。（JSON 格式错误：让 `JSON.parse` 抛出——快速失败；Stop 处理器现有的外层行为可接受。注意：`summarize.ts` 此段没有 try/catch；如果此处抛出会导致 `.claude.json` 损坏的用户摘要中止，则仅将文件读取包在返回 `null` 账户的保护中，并添加 `logger.debug`——实现时决定并在代码注释中记录。）
- 仅检查：`oauthAccount.organizationType`、`oauthAccount` 是否存在、`customApiKeyResponses.approved`（字符串数组）。绝不读取类似 token 的字段。

### 1.3 `src/cli/handlers/summarize.ts` —— 发送两个新字段

在 `lastAssistantMessage` 解析后（`:104` 之后）且 worker POST（`:138-147`）之前：

```ts
const observedModel = transcriptPath ? extractLastAssistantModel(transcriptPath) : undefined;
const observedBilling = input.platform === 'claude-code' ? detectObservedBilling() : undefined;
```

将 `observedModel`、`observedBilling` 添加到 `/api/sessions/summarize` 请求体（`:141-146`）。保持服务器运行时路径（`summarizeViaServer`）不变。

### 1.4 测试（阶段 1）

- `tests/transcripts/observed-model-extraction.test.ts` —— `extractLastAssistantModelFromJsonl`：返回最后一条 assistant 行的模型（Claude Code `type` 形式），当 Cursor `role` 形式携带 `message.model` 时返回模型，跳过用户行和格式错误行，当没有 assistant 行带模型时返回 `undefined`。
- `tests/shared/observed-billing.test.ts` —— 使用注入的 `env` 对象和写入 scratchpad/tmp 目录（`mkdtempSync`）的临时 `.claude.json`，覆盖 §0.2 的完整矩阵：bedrock/vertex/foundry；无账户的 api key；有账户但未批准的 api key → 账户层级；已批准的 api key（后 20 个字符）→ `api_key`；`organizationType: 'claude_max'` → `max`；无 organizationType 的账户 → `subscription`；仅 `CLAUDE_CODE_OAUTH_TOKEN` → `subscription`；什么都没有 → `unknown`；文件缺失 → `unknown`。
- 更新 `tests/cli/handlers/summarize-tag-stripping.test.ts:35-40` 的 mock，同时导出 `extractLastAssistantModel: () => 'claude-test-model'`，并 grep 其他 mock `transcript-parser.js` 的测试（`grep -rl "transcript-parser.js" tests`）——逐一扩展。
- 在 `tests/cli/handlers/` 中新增断言（添加到 `summarize-tag-stripping.test.ts` 或新建 `summarize-observed-fields.test.ts`）：发往 `/api/sessions/summarize` 的 POST 请求体包含 `observedModel`，且对于平台 `claude-code`，`observedBilling` 为字符串。

### 1.5 验证清单（阶段 1）

```
npm run typecheck
bun test tests/transcripts tests/shared tests/cli
npm run lint:hook-io
grep -n "observedModel\|
```