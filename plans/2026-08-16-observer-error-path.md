# Observer 错误路径 — 一次分类、携带消息、告诉用户该做什么

## 首要目标

**付费用户的 observer 停止工作时，应在一次会话内发现——并用大白话被告知究竟发生了什么、该做哪一件事。** 绝不能静默 502 循环。绝不能出现只有我们才看得懂的消息。每个错误、每个 provider、每一跳：*发生了什么 · 为什么 · 这样做 · 链接 + 请求 id*。

以下所有内容都以这句话为衡量标准。如果某项任务不能把用户从"它悄悄停了"推进到"我知道原因，也知道该做什么"，它就不属于本计划。

**问题（2026-08-14 → 16 观察到）：** 推理额度耗尽的 Pro 用户会看到 `OpenRouter upstream error (status 502)` ×3 次重试，永远如此，且悄无声息。根因是一连串小信息丢失：OpenRouter 返回 **403 `Key limit exceeded`** → cmem.ai 网关只特判 **402**，其余一律 502 包装 → worker 分类器把 5xx 当作 `transient`，重试，并丢弃上游响应体（其中包含补救办法）→ observer 健康台账（PR #3538）记录了一条无用的字符串。与此同时，6 位付费客户在不知原因的情况下失联了 3–5 天。

**设计（一条原则）：** 在源头（网关）**只分类一次**。后续每一跳都*携带*结构化错误——`{code, message, action, url, request_id}`——绝不重写，绝不重试不可能成功的请求，并在日志、会话启动上下文以及（后续）邮件中显示同样的措辞。

**分类法——6 个 code，这就是全部词汇：**

| `code` | HTTP | 重试？ | worker `kind` |
|---|---|---|---|
| `allowance_exhausted` | 402 | 否 | `quota_exhausted` |
| `key_invalid` | 401 | 否 | `auth_invalid` |
| `subscription_inactive` | 402 | 否 | `auth_invalid` |
| `rate_limited` | 429 | 是（Retry-After） | `rate_limit` |
| `upstream_unavailable` | 503 | 是 | `transient` |
| `bad_request` | 400 | 否 | `unrecoverable` |

**每条消息 = 4 部分：** *发生了什么* · *为什么* · *这样做*（一个具体动作）· 链接 + 请求 id。

**仓库：** 阶段 1 = `claude-mem-pro`（网关）。阶段 2–3 = `claude-mem`（worker）。每个阶段自包含；在新的上下文中运行。阶段 3 依赖 PR #3538 先合并。

**约束性要求：** 只做根因修复；不新增重试循环，不加回退，不加环境变量逃生口，唯一例外是下文点名的测试接缝；改动范围与缺陷规模相当。不要臆造 URL——只允许 `https://cmem.ai/dashboard`、`support@cmem.ai` 和 `https://github.com/thedotmack/claude-mem/issues`。不要编辑 `CHANGELOG.md`。

---

## 阶段 0 — 汇总勘察（任何阶段之前先读；不要重新推导）

**这如何服务于首要目标：** 你无法为没见过的故障写出诚实的错误消息。本阶段锁定*实际*的上游响应体（403 `Key limit exceeded`、401 `User not found.`）、今天消息丢失的确切代码行，以及警告将要渲染所经由的台账 API——这样后续每个阶段都改在真实的接缝上，而不是猜测的接缝上。

已对照 `claude-mem` @ `or-issue` HEAD（`d768ba364`）、`origin/observer-health-alerts`（PR #3538）和 `claude-mem-pro` @ main（2026-08-16）验证。经验性上游事实是对 OpenRouter 实时捕获的。

### OpenRouter 实测响应（2026-08-15/16 捕获，生产密钥）

| 情况 | HTTP | 响应体 |
|---|---|---|
| 子密钥达到上限（`limit_reset: null`，即试用） | **403** | `{"error":{"message":"Key limit exceeded (total limit). Manage it using https://openrouter.ai/workspaces/default/keys/<hash>","code":403}}` |
| 子密钥达到上限（`limit_reset: 'monthly'`） | 403（括号内不同，如 `(monthly limit)`）——匹配 `/key limit exceeded/i`，绝不要匹配括号内容 | |
| 无效密钥 | **401** | `{"error":{"message":"User not found.","code":401}}` |
| 文档声称的密钥耗尽 402 | 生产环境未观察到；保留该分支，成本为零 | |

### 网关（`claude-mem-pro`）— `src/app/api/inference/v1/chat/completions/route.ts`（432 行）

当前所有错误返回（行号 → 状态码 → 消息 → 条件）：

| 行号 | HTTP | 消息 | 条件 |
|---|---|---|---|
| L83（`unauthorized()` L60-65） | 401 | `Missing or malformed CMEM Pro key` | 没有 `Bearer cm_pro_…` |
| L92 | 401 | `Unrecognized CMEM Pro key` | 没有匹配的 `proUsers.setupToken` |
| L99-108 | 402 | `CMEM Pro subscription is not active. Manage billing at https://cmem.ai/dashboard` | `!isProActive(proUser)`——把 `none/pending/past_due/cancelled` 合并处理 |
| L113-118 | 429 | `Rate limit exceeded` + `Retry-After: 60` | `isRateLimited('cmem-inference', userId, 600, 60_000)` |
| L124-127 / L131-134 | 400 | `Invalid JSON body` / `messages[] is required` | 请求体解析 |
| L170-173 | 500 | `Inference credential is unreadable` | 解密失败（fail-closed） |
| L236-239 | 503 | `Could not provision inference access` | 签发失败 |
| **L279-301** | **402** | 试用 / 月度文案（见 L291-300） | **仅** `upstream.status === 402` |
| **L309-317** | `429→429 else 502` | `payload?.error?.message ?? 'Observer model request failed'`，带 `code: upstream.status` | `!upstream.ok \|\| !payload` 的兜底分支（L274）——**今天 403 就走到这里** |
| L420-428 | 504 / 502 | `Observer model timed out` / `unreachable` | fetch 抛出异常 |

影响阶段 1 的事实：
- **任何地方都没有请求 id**——不生成、不从 OpenRouter 读取、不记录。必须在服务端生成（`randomUUID()`），并贯穿到 JSON 响应体、`x-request-id` 头和每一行 `console.*` 日志。
- `PRIVATE_HEADERS = { 'Cache-Control': 'private, no-store' }` 在每个路由里复制粘贴（此处 L58）；`src/lib` 中**不存在共享错误辅助函数**（grep `errorResponse|jsonError|ApiError` = 0 命中）。阶段 1 创建一个。
- 在用的 `paymentStatus` 值：`'none' | 'pending' | 'active' | 'trialing' | 'past_due' | 'cancelled'`（schema 注释 `src/db/schema.ts:22` 漏了 `trialing`）。现有按状态区分的文案可对齐语气：`src/app/api/pro/status/route.ts:142-153`。
- 试用密钥是 `limit_reset: null`（累计制）——对它们来说"下个周期重置"是**假的**（`src/lib/pro/openrouter-child-keys.ts:63-69`）。`trialing` 在 L280 推导。
- **不往返 Stripe 就无法推导"于 <日期> 重置"**（`current_period_end` 不在 `proUsers` 上；仅在 `src/lib/pro/account-summary.ts:35`、`src/lib/admin/pro-users.ts:90-92` 实时获取）。**决定：消息中不放日期。** 说"在你下个计费周期开始时"。若日后新增反规范化列再重新考虑。
- `captureServer(distinctId, event, props)`——`src/lib/analytics/posthog-server.ts:41-45`，从不抛异常，等待 flush。规则 A3（props 中不得有 token/邮箱）和 A7（分析绝不能让推理失败）见 `plans/2026-08-08-posthog-funnel-and-admin.md:521,567`。现有事件：`pro_inference_cap_exhausted` 在 L287-290，带 `{limit_usd, trialing}`。
- `OpenRouterKeyError`（`src/lib/pro/openrouter-child-keys.ts:89-96`）携带 `.status`。
- `isRateLimited(route, key, max, windowMs)`——`src/lib/rate-limit.ts:34-39`；无法报告窗口剩余时间——保留 `Retry-After: 60`。
- 测试：**没有 runner**；惯例是独立 tsx 脚本 `scripts/test-<name>.ts`，注册为 `"test:<name>": "node --import tsx scripts/test-<name>.ts"`（`package.json:17-33`）。照抄 `scripts/test-mcp-auth.ts` 的骨架（`node:assert/strict`、`main().then(...).catch(...)`）。`bearerFrom`（路由 L72）为可测试性而导出——沿用该模式：**导出一个纯映射函数并测试它**，无需 mock HTTP。
- 部署：合并到 `main` 自动部署（`CLAUDE.md:4`）。环境变量在 Vercel 面板。`CMEM_PRO_MONTHLY_LIMIT_USD=30` 已在生产设置（2026-08-16）。
- 邮件（Resend）存在，但 `proUsers` 没有邮箱列，也没有任何去重——**本计划范围外**（记为后续事项）。

### Worker（`claude-mem`）

- `classifyOpenRouterError`——`src/services/worker/OpenRouterProvider.ts:30-95`。分支顺序：响应体标记 `quota exceeded|insufficient credits|insufficient_quota` → `quota_exhausted`（L44-53）；429 → `rate_limit`（L55-60）；401/403 → `auth_invalid`（L62-67）；400/404 → `unrecoverable`（L69-74）；5xx → `transient`（L76-81）；无状态码 → `transient`（L84-89）；兜底（含 402）→ `unrecoverable`，带 body[:200]（L91-94）。**`input.requestId` 被接收但从未使用。** 只有兜底分支保留了响应体的任何部分。
- `ClassifiedProviderError`——`src/services/worker/provider-errors.ts:2-33`：`{ kind, retryAfterMs?, cause }`，kind 是开放的字符串联合。今天没有结构化字段。
- 重试策略——`src/services/worker/retry.ts`：`isRetryableKind` L56-61 重试 `transient|rate_limit` 和**未分类错误**；`withRetry` L76；重试警告在 L116-119（`Retrying ${label} after …ms (attempt n/2)`）。
- 调用点——`queryOpenRouterMultiTurn` L227-329：请求 id 在 L256 捕获（`x-request-id` ?? `x-openrouter-request-id`）；`!response.ok` 抛出在 L263-272，传入 `status, bodyText, headers, cause, requestId`；200-带-`error` 抛出在 L276-284，把 `{code,message}` 压平成字符串，且**不传 requestId**。
- 每次失败的日志扇出（5 行）：重试警告 ×2（`retry.ts:116`）、`init query failed`（`OpenAICompatibleProvider.ts:114/116`）、`✗ <Provider> agent error`（`handleSessionError` L305）、`Generator failed`（`SessionRoutes.ts:211-215`，从 L179 带 `provider` + `errorMsg`）。
- 同类分类器：Gemini `GeminiProvider.ts:25-97`（在合成的 cause L37-39 中使用 requestId），Claude `ClaudeProvider.ts:54-164`（逐字保留消息）。**必须镜像的服务端文案：** `src/server/generation/providers/shared/error-classification.ts:73-135`（`classifyHttpProviderError`；头部 L3-6 禁止导入 worker 代码）。
- Base URL：`resolveOpenRouterChatCompletionsUrl` `src/shared/openrouter-base-url.ts:48-62`；Pro 用户的 URL 是 `https://cmem.ai/api/inference/v1/chat/completions`（`src/npx-cli/cmem-pro-costs.ts:197`）→ `endpointClass: 'custom'`（`OpenRouterProvider.ts:159`）。
- PR #3538 `src/shared/observer-health.ts`（在 `origin/observer-health-alerts` 上）：`ObserverHealthState` L21-34（`lastErrorMessage: string|null`——扁平），`scrubErrorMessage` L71-80，`readObserverHealth` L82-93（`{...EMPTY_STATE, ...parsed}` 合并 → 新增字段向后兼容），`recordObserverFailure(provider, errorMessage, filePath?)` L180-197，`recordObserverSuccess` L199-209，`renderObserverHealthWarning(state, nowMs?)` L232-254（输出 `Latest error: <message>` + 通用补救文案 L253-254，写着"检查 ~/.claude-mem/settings.json"——对 Pro 用户是错的）。Hook：`SessionRoutes.ts` 中紧接 `Generator failed` 日志之后 → `recordObserverFailure(provider, errorMsg)`。消费方：`ContextBuilder.ts:181-189`（`withObserverHealthWarning`）在 L208/221/235 应用。
- 测试：`tests/worker/provider-classifiers.test.ts`（`describe('classifyOpenRouterError')` L136-202，7 个用例，**只断言 `kind`/`retryAfterMs`，从不断言 `message`**）；PR 新增 `tests/observer-health.test.ts`（fixture 已使用 `Key limit exceeded (monthly limit). Manage it using https://openrouter.ai/keys/abc`）。Runner：`bun test tests`（`package.json:98`），模式 `import { describe, it, expect } from 'bun:test'`。`plugin/scripts/*.cjs` 是已提交的 bundle——源码改动后运行 `npm run build`。

### 反模式（不要做）
- 不要把任何用户可处理的问题 502 包装。不要在响应体中返回 `code: <upstream status>`。
- 不要把子密钥、setup token 或邮箱放进 PostHog props 或错误消息。
- 不要臆造 `cmem.ai/upgrade` 或 `/billing` URL——它们不存在。只用已批准的链接（见顶部）。
- 不要为了日期在错误路径上调用 Stripe。
- 不要新增重试层；不要让未分类错误变为不可重试（那是范围外的行为变更）。
- 不要从 `src/server/*` 导入 `src/services/worker/*`。

---

## 阶段 1 — 网关：一次分类、诚实状态码、结构化信封、请求 id（`claude-mem-pro`）

**这如何服务于首要目标：** 消息在这里*诞生*。网关是唯一知道用户套餐、试用状态和限额的一跳，因此也是唯一能说"你已用完 $30 额度——下个周期重置，如需更多请邮件联系支持"的地方。诚实的状态码意味着 worker 不再重试不可重试的请求；请求 id 意味着"邮件联系支持"是真实路径，而非死胡同。

**分支：** 从 `main` 切出 `feat/inference-error-taxonomy`。

### 1.1 新模块 `src/lib/http/gateway-error.ts`
从 `src/app/api/pro/trial/approve/route.ts:38` 复制单行响应形状并扩展：

```ts
import { NextResponse } from 'next/server';

export type GatewayErrorCode =
  | 'allowance_exhausted' | 'key_invalid' | 'subscription_inactive'
  | 'rate_limited' | 'upstream_unavailable' | 'bad_request';

export const GATEWAY_ERROR_STATUS: Record<GatewayErrorCode, number> = {
  allowance_exhausted: 402, key_invalid: 401, subscription_inactive: 402,
  rate_limited: 429, upstream_unavailable: 503, bad_request: 400,
};

export interface GatewayError {
  code: GatewayErrorCode;
  message: string;      // what happened + why, one sentence
  action: string;       // what the user should do, one sentence
  url?: string;         // approved links only
  request_id: string;
}

export const PRIVATE_HEADERS = { 'Cache-Control': 'private, no-store' } as const;

export function gatewayErrorResponse(err: GatewayError, extraHeaders: Record<string,string> = {}) {
  return NextResponse.json({ error: err }, {
    status: GATEWAY_ERROR_STATUS[err.code],
    headers: { ...PRIVATE_HEADERS, 'x-request-id': err.request_id, ...extraHeaders },
  });
}
```

同时导出一个**纯**上游映射函数（这是被测对象——镜像路由 L72 的 `bearerFrom` 导出供测试模式）：

```ts
export type UpstreamOutcome =
  | { kind: 'allowance_exhausted' }
  | { kind: 'rate_limited'; retryAfterSec: number
```