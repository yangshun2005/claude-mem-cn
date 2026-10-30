# PostHog 自驱动配置报告

_生成于 2026-07-14，项目 CMEM（ID 463218）_

## 摘要

已为 claude-mem 项目配置 PostHog 自驱动。已启用错误追踪（三种触发类型）、会话回放、GitHub Issues 和支持工单信号源；侦察兵队伍已调整为三名侦察兵，匹配本项目的主要使用面（AI/LLM 使用和 MCP 工具调用）。发现结果将在约 30 分钟内开始出现在[自驱动收件箱](https://us.posthog.com/project/463218/inbox)。

---

## AI 数据处理

**已批准。** 组织级 AI 数据处理同意已在本次运行前授予。

---

## GitHub

**已连接** — 配置时 GitHub App 集成（`thedotmack`，集成 ID 175967）已存在，连接于 2026-06-10。已同步一个仓库：`thedotmack/claude-mem`。

---

## 已启用产品

| 产品 | 状态 | 备注 |
|---|---|---|
| 会话回放 | **已激活** | 录制数据来自 cmem.ai；此部署中 `products-enable` API 不可用 — 服务器状态已由实时录制确认 |
| 错误追踪 | **已激活** | 通过 `posthog-node` 配合 `enableExceptionAutocapture` 上报 110 万+ 事件；`products-enable` API 不可用 |
| 支持（对话） | **需要后续处理** | 无法通过 API 启用该产品（工具不可用）；手动步骤见后续事项 |

本项目使用 `posthog-node`（后端），而非 `posthog-js`。无需检查 `posthog.init` 覆盖。Web 前端（cmem.ai）的会话回放似乎由另一个仓库或代码片段接入。

---

## 信号源

| source_product | source_type | 操作 |
|---|---|---|
| `error_tracking` | `issue_created` | **已启用**（新增，ID `019f6318-32fa-73a9-914f-55617f6800de`） |
| `error_tracking` | `issue_reopened` | **已启用**（新增，ID `019f6318-3775-7090-a2e5-d98f893734d0`） |
| `error_tracking` | `issue_spiking` | **已启用**（新增，ID `019f6318-39b6-7161-8bce-8e03a206d92b`） |
| `session_replay` | `session_analysis_cluster` | **已启用**（新增，ID `019f6318-3dd1-7dae-b4a0-36eba57cb746`；默认 10% 采样率） |
| `conversations` | `ticket` | **已启用**（新增，ID `019f6318-4047-764c-94fc-ee62a511a9b2`；在连接渠道前处于休眠状态） |
| `signals_scout` | `cross_source_issue` | **默认开启** — 无需配置行；侦察兵发现结果自动进入收件箱 |
| `llm_analytics` | `evaluation_report` | **已跳过** — 仅内部响应器，不面向用户 |
| `logs` | — | **已跳过** — 不是 v1 响应器 |

---

## 已连接工具

| 工具 | 状态 |
|---|---|
| GitHub Issues | **已连接** — `Github` 数据仓库源（ID `019eb017-8f11-0000-8ce2-d7bed20cb960`）已存在；截至 2026-07-14 已同步 1,766 个 Issue。响应器（`github` / `issue`，ID `019f6324-8a46-71ab-ad0b-8cea5b4fb795`）已启用。 |
| Linear | **未使用** — 未选择 |
| Zendesk | **未使用** — 未选择 |
| pganalyze | **未使用** — 未选择 |

注意：GitHub 数据仓库源的 `stargazers` 表存在令牌错误（`Access forbidden`）。响应器读取的 `issues` 表同步正常。见后续事项。

---

## 侦察兵队伍

**已启用 3 个，已禁用 23 个。**

### 已启用

| 侦察兵 | 原因 |
|---|---|
| `signals-scout-general` | 始终开启 — 扫描跨产品关联，覆盖无专职侦察兵负责的领域 |
| `signals-scout-ai-observability` | 本产品封装 Claude/Anthropic API — LLM 成本、延迟和错误回归直接关系产品 |
| `signals-scout-mcp-tool-calls` | 本产品本身就是 MCP 服务器 — `$mcp_tool_call` 遥测覆盖工具失败率和令人困惑的 Schema |

### 已禁用

| 侦察兵 | 原因 |
|---|---|
| `signals-scout-error-tracking` | 已由原生 `error_tracking` 源覆盖（上述三种触发类型均已启用） |
| `signals-scout-session-replay` | 已由上述启用的原生 `session_replay` 源覆盖 |
| `signals-scout-product-analytics` | 未确认有活跃的漏斗/留存洞察 — 若构建产品分析仪表板可启用 |
| `signals-scout-web-analytics` | 本仓库未确认有 UTM/来源追踪 — 若为 cmem.ai 添加 Web 分析可启用 |
| `signals-scout-feature-flags` | 未确认使用功能标志 — 若采用标志可启用 |
| `signals-scout-surveys` | 使用中的调查为 0 |
| `signals-scout-revenue-analytics` | 未发现支付 SDK |
| `signals-scout-logs` | 未确认使用 PostHog 日志产品 |
| `signals-scout-csp-violations` | 未配置 CSP 报告 |
| `signals-scout-experiments` | 无活跃 A/B 实验 |
| `signals-scout-customer-analytics` | 未确认有群组/账户分析 |
| `signals-scout-data-pipelines` | 未发现 CDP 目标或 hog 流程 |
| `signals-scout-replay-vision` | 未配置 Replay Vision 扫描器 |
| `signals-scout-apm` | 未发现 OpenTelemetry/APM 追踪 |
| `signals-scout-anomaly-detection` | 对本项目规模而言，`general` + 专职侦察兵已足够覆盖 |
| `signals-scout-observability-gaps` | 若事件覆盖缺口成为问题，可稍后启用 |
| `signals-scout-health-checks` | 若配置健康成为问题，可启用 |
| `signals-scout-inbox-validation` | 全新配置下无用 — 尚无已交付的修复可验证 |
| `signals-scout-ingestion-warnings` | 若出现摄取错误，可启用 |
| `signals-scout-insight-alerts` | 未发现已配置的洞察告警 |
| `signals-scout-skills-store` | PostHog 内部技能卫生侦察兵 |
| `signals-scout-data-warehouse` | GitHub 源 `stargazers` 令牌问题 — 见后续事项 |
| `signals-scout-web-vitals` | 本仓库未确认有 `$web_vitals` 事件 |

---

## 自定义侦察兵

提议了两个使用面，均被拒绝：

| 提议的侦察兵 | 使用面 | 拒绝原因 |
|---|---|---|
| 记忆流水线健康 | `observer_turn_rollup` → `context_injected_rollup` 量/比率下降 | 用户驳回了提议 |
| 安装到首次记忆漏斗 | `install_completed` → `worker_started` → 首次观察 | 用户驳回了提议 |

**缺口分析已排除：**
- Chroma 依赖健康 — 已由 `error_tracking` 原生源覆盖（`ChromaUnavailableError` 是主要 Issue）

**噪音逃生通道：** 要让任何侦察兵进入试运行（运行并记录日志，但不向收件箱发送任何内容），在 PostHog > 自驱动设置中将其配置设为 `emit: false`。

---

## 后续事项

- [ ] **启用支持/对话产品** — 此 PostHog 部署中 `products-enable` API 不可用。若尚未开启，请在[项目设置 > 产品](https://us.posthog.com/project/463218/settings)中手动启用会话回放、错误追踪和对话。
- [ ] **连接支持入站渠道** — `conversations/ticket` 信号源已启用并等待中。在 PostHog 中连接电子邮件、收件箱或 Slack 渠道，以便支持工单开始进入收件箱。（[集成设置](https://us.posthog.com/project/463218/settings/environment-integrations)）
- [ ] **修复 GitHub 令牌权限** — `stargazers` 表报错"Access forbidden / rate limits"。在[集成设置](https://us.posthog.com/project/463218/settings/environment-integrations)中检查 GitHub App 令牌范围 — `issues` 表同步正常，因此不阻塞自驱动响应器，但可能影响其他数据仓库查询。
- [ ] **考虑启用 `signals-scout-memory-pipeline`** — 一个监视 `observer_turn_rollup` / `context_injected_rollup` 量和比率下降的自定义侦察兵，可在用户察觉记忆质量下降前捕获静默的流水线故障。如需此覆盖，请通过[自驱动侦察兵](https://us.posthog.com/project/463218/inbox)创建。
- [ ] **考虑启用 `signals-scout-install-funnel`** — 一个监视 `install_completed` → `worker_started` → 首次观察的自定义侦察兵，可捕获损坏的首次运行流程和平台特定回归。若安装转化率成为优先指标，请创建。

---

## 后续流程

侦察兵协调器约 30 分钟内加载新配置。侦察兵每日运行（间隔 1,440 分钟），并将发现结果作为报告发送到[自驱动收件箱](https://us.posthog.com/project/463218/inbox)。错误追踪和会话回放信号到达更快 — 它们随单个事件落地即触发。可立即操作的报告可直接从收件箱转为编码任务。