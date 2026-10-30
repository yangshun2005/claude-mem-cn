<!-- 镜像自 GitHub Issue #2785。Issue 是公开跟踪器；本文档是设计。
     若两者出现偏差，设计以本文档为准，状态以 Issue 为准（oh-my-issues 健康检查）。 -->

> **跟踪器：** https://github.com/thedotmack/claude-mem/issues/2785

# [plan-12] Provider 与可扩展性路线图 — 新增能力，而非缺陷

## 缺陷

这是路线图总纲，不是缺陷集合：它汇总了跨 provider、摄取/过滤、可观测性、认证和 UX 的新增能力，使它们以连贯的切片交付，而不是零散的 PR。

## 子项

### Provider 与认证
- #2522 — Gemini provider 支持 Vertex AI（GCP 服务账号 / ADC + Vertex 端点）
- #2704 — 用于可刷新网关令牌的 auth-helper 命令（等价于 apiKeyHelper）

### 摄取与过滤
- #2690 — 回填 / 摄取已有的 Claude Code 会话 JSONL 文件
- #2498 — 增量扫描技能，用于 Claude 会话之外所做的变更
- #2463 — `tool_response` 级过滤（扩展名 / 大小 / 内容启发式），防止二进制/Playwright 内容膨胀
- #2423 — 按目录禁用支持
- #2711 — 将 worktree 观察写入父项目的选项
- #2736 — 跳过 / 节流子代理观察（Dynamic Workflow `workflow-subagent`）— 重新提出 #2303

### 可观测性与日志
- #2566 — MCP 语法内省路由 + worker provider 重试遥测 + 审计日志
- #2513 — 明确 logger 审计策略并固化 CI 行为
- #2702 — 日志清理：类型安全的 Component 联合类型、删除无用日志、降低噪音

### UX / 语义 / 可扩展性
- #2645 — 启动 UI 标签的 i18n 支持
- #2467 — 在连续工具序列中，将 PreToolUse:Read 注入视为轮次边界（语义决策）
- #2418 — 启用 OpenHarness 集成 / 允许来自 Fork 的 PR

## 修复顺序

设计文档：`plans/12-provider-and-extensibility-roadmap.md`。按子领域作为独立切片交付；每个切片自带测试。没有单个 PR 能关闭此总纲 — 各子领域落地后它才关闭。

## 范围之外

所有缺陷集合（plan-01..11、plan-13）。


---

## 轮次：2026-09-12 oh-my-issues

三个新增能力请求被归入此路线图并作为子项关闭：

- **#4060** — 对 `tool_uses.tool_input` + `tool_uses.tool_response` 建立 FTS5，通过 `search`/`smart_search` 暴露为可选更深层级，仅在观察级搜索无结果时查询。必须复用 plan-25（#3982）的分词器方案，而不是另起一套。
- **#3833** — 在 observer/compression 调用的稳定系统前缀上启用提示缓存（`cache_control: {type: "ephemeral"}`）。与 provider 无关，带安全的无缓存回退；绝不在逐观察轮次上使用。验证依据是缓存与未缓存的输入 token 对比，以及对不支持缓存的 provider 请求体逐字节一致。
- **#3845** — Brainbeats：将匹配的观察作为简报 POST 到可配置的 Grok Bot webhook，复用 `TelegramNotifier` 及现有的触发类型/概念匹配。不引入第二套规则引擎；POST 失败不得中断摄取。仅开源引擎（不含 `claude-mem-pro`）。PR #3846 正在处理此项。