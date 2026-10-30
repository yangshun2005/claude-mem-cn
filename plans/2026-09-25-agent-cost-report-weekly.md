# Agent 成本报告 — 每周重建（按 transcript 实测、Timing 风格、美元计价）

**日期：** 2026-09-25（PT）
**致：** Alex Newman
**状态：** 已由 Alex 于 2026-09-25 4:42pm PT 批准（GREEN）；合并/发布/版本号升级仍受门控。决策 G1–G16 记录在批准清单中。尚未实现任何内容；目前唯一的提交都只针对本计划文件。
**分支 / worktree：** `work/cost-report-weekly`，位于 `/workspace/claude-mem/.claude/worktrees/cost-report-weekly`。绝不切换分支。
**执行方式：** 对本文件运行 `/do`，每个全新会话执行一个阶段，每个已验证阶段结束时提交。
**2026-09-25（PT）修订：** 新增 Phase 2B，即取自 Alex 本人抱怨的 agent 行为指标，以及配套的渲染、验证和批准清单条目。状态不变。
**2026-09-25 ~4:35 PM（PT）再次修订：** Alex 划定了一个"Wins vs mistakes"（成果 vs 失误）区块，直接放在美元主数字下方（4:28 PM PT）：一行失误成本、带成本的成果，以及同一时间轴上的两条逐日时间线。已加入 2.8（成果与归因）、2B.9（失误行与时间线数据）、3.8（渲染）、8.6（对账）。Phase 2B 的指标列表现为暂定，待 Frustration Arc 席位确定。新增"范围之外"章节。状态不变。
**2026-09-25 ~4:40 PM（PT）修订：** Phase 2B 已与 Frustration Arc 席位的最终列表（`/workspace/frustration-arc/metrics-ideas.md`）对齐。移除 PROVISIONAL；M1–M6 合并为 12 个模式（P1–P12）加 2 个辅助指标、4 个摘要卡片。失误行现在以低值（同会话）数字为主，高值作为上限放在 Details 中。在会话与 PR 关联之前，成果成本显示"未测量"。新增规则有效性（2B.10）、前置条件列表（0.5）、Phase 8 中的 Frustration Arc 对比（8.7），以及重新编号的批准清单（G1–G16）。状态不变。
**2026-09-25 4:42 PM（PT）修订，GREEN：** Alex 在提交 `7f5e9e7d` 上批准了本计划（由 Ori 转达）。G1–G16 的答案记录在批准清单中。计划文本已与所选答案对齐：G3（默认窗口 = 最近 7 个完整 PT 日，不含今天）、G8（分类器默认关闭，启用时每次运行上限 $2.00）、G11（在会话与 PR 关联之前，成果成本"未测量"；放弃血缘回退）、G13（会话 id 仅作为提交 trailer 打标，只写 id，不写 PR 正文行），以及 G2（镜像携带该 skill）、G4（Grok Bot "unavailable"，不显示席位数量）、G10（Alex 对 hedging 的定义）、G12（成果 = 已合并 PR、已发布版本或 Alex 的称赞；完成的工作不算成果）。推送工作分支和开 PR 是例行操作。**合并到 `main`、npm publish 和 `/version-bump` 各自仍需 Alex 单独批准。**
**取代：** `/workspace/plans/2026-09-25-agent-cost-report-timing-style.md` 中的纯渲染计划（其数据基础是 `discovery_tokens`，单位是美分；两者均已废弃）。其映射表在 Phase 3 中复用。
**所有时间均为 PT。** 所有密钥仅以环境变量名引用。绝不读取 `.env` 或设置文件来获取密钥。
**每个验证命令的临时目录：** 每个会话定义一次，`ACR_TMP=${ACR_TMP:-/tmp/acr-weekly}`，然后按阶段使用 `$ACR_TMP/p1`、`$ACR_TMP/p2`、……。Phase 3 和 Phase 8 从 `$ACR_TMP/p2` 读取 Phase 2 输出。`$ACR_TMP` 下的任何内容都不提交。

---

## 主要目标

Claude-Mem 插件 skill `agent-cost-report` 为任意周期生成可信的 agent 成本报告，默认最近 7 个完整 PT 日，不含今天（G3）。主数字是美元。美元来自 Claude Code transcript（每次回复的精确 token 用量），按 OpenRouter 公开标价计价，并标注为 ESTIMATED。只有在有受认可来源时才显示实测的提供商支出。记录者（note-taker）自身的 token 单独计价，绝不计入 agent 成本。报告外观仿照 Timing 应用 mockup：主数字、成本条带、五个可视化，以及折叠的 Details 区块。

---

## 已定决策（不再重开）

1. **主数字单位是美元**（Alex 经 Ori 确定）。主数字和总计显示两位小数的美元（`$109.25`）。任何地方都不再以美分为先显示。每个美元数字都带有标签（实测 vs 估算）和依据，例如"estimated at OpenRouter list prices from measured tokens"。这取代了早先计划（`/workspace/plans/2026-09-25-agent-cost-report-timing-style.md:390-408`）和简报（`/workspace/timing-report-brief/brief.md:44`）中"低于 $1 用美分"的规则。
2. **默认范围 = 最近 7 个完整 PT 日，不含今天**（G3，Alex 2026-09-25）：`end` = 今天开始时的 PT 午夜（不含），`start` = `end − 7 days`，按 PT 日边界。默认窗口从不包含不完整的一天。支持任意周期：显式 start/end、单个会话，或单个项目加周期。

## 约束性要求（来自 Alex 的原始规格，原样保留）

当前 SKILL.md 已声明这些。重写时逐字保留（`/workspace/claude-mem/plugin/skills/agent-cost-report/SKILL.md:186-195`）：

- 始终标注估算 vs 实测（`cost_basis` = `estimated_usage` 或 `measured_provider`）。
- 不匹配时始终说明 **measured spend unavailable**。未知绝不写"$0 spent"。
- 始终以**已完成成果**作为工作单位（每个已完成成果的成本）。
- 始终将 Rework 仅放在 `failure_type` 下，绝不作为工作类别（另见 `SKILL.md:82`）。
- 始终使用渐进式 Mem Search（search → timeline → get_observations，`SKILL.md:51-55`）；证据 ID 放在附录（现为 Details 区块）。
- 交付物保持不变：自包含 HTML、可选 PDF、`report.json`、`line-items.csv`、`evidence.json`（`SKILL.md:163-170`）。

本计划新增，精神相同：

- 记录者（observer）成本与 agent 成本分开计价，绝不加入 agent 主数字。
- 关键词派生的类别标签在复核通过前只是草稿。记录标签来源。
- Grok Bot 席位用量和 Mac transcript 在存在受认可来源之前显示为"unavailable"或"extrapolated (low confidence)"。绝不写 $0，绝不猜测。
- 绝不打开实时数据库进行写入。每次运行都在只读快照上工作。

## 本计划范围之外

- **会话开始时自动注入经验教训**（把过去失误的教训注入新会话）不在范围内。这是 claude-mem 的另一个记忆功能，有自己的计划。本报告只测量和展示成果与失误，不向 agent 会话反馈任何内容。
- **0.5 中列出的上游数据修复**（claude-mem 写入提示词时打标签、为 Grok 和 Codex 记录模型 id、将会话 id 作为提交 trailer 打标、保留 Grok Bot 聊天）是对其他产品或内部工作流的更改。本计划将它们列为前置条件，只构建报告侧。每项都需要自己的计划和 Alex 的批准。

---

## Phase 0 — 文档发现、允许的事实，以及将 skill 目录纳入跟踪

**会话目标：** 用列出的命令重新验证以下事实，然后将未跟踪的 skill 目录原样带入本分支，使后续每个阶段都是可审查的 diff。一次提交。

### 0.1 允许的事实和 API（2026-09-25 由规划者子代理验证）

#### A. Transcript 工作流（实测 token 来源）

| 事实 | 引用 |
|---|---|
| 周期为任意 PT 窗口，end 不含；调用 `python3 fetch_prices.py` 然后 `python3 weekly_report.py 2026-09-18 2026-09-26` | `/workspace/weekly-cost-workflow/WORKFLOW.md:3,7-8` |
| 流水线步骤 1–10：只读快照 DB → 会话/席位 → 成果 → 关键词标签 → 活跃分钟 → transcript 去重 → 按标价计价 → observer 单独计价 → 外推未测量 → 可选云端交叉核对 | `WORKFLOW.md:15-24` |
| Transcript glob `~/.claude/projects/**/*.jsonl`，按 mtime 预过滤；项目目录 = `/.claude/projects/` 之后的路径段 | `parse_transcripts.py:16-19` |
| 仅 `type == "assistant"` 且带 `timestamp` 的行；窗口测试 `s <= t < e` | `parse_transcripts.py:26-28` |
| 读取的 usage 字段：`message.usage.input_tokens`、`output_tokens`、`cache_creation_input_tokens`、`cache_creation.ephemeral_1h_input_tokens`、`cache_read_input_tokens`；可选 `costUSD` | `parse_transcripts.py:34-38` |
| 5m 缓存写入推导为 `cache_write - cache_write_1h` | `weekly_report.py:94` |
| 去重键 `(message.id, requestId)` | `parse_transcripts.py:15,29-31` |
| 会话键 = transcript `sessionId`（子代理文件共享父级 id）；保留 `isSidechain` 和 `message.model` | `parse_transcripts.py:32-33` |
| 真实 transcript 行证明字段名：`type:"assistant"`、`timestamp`、`sessionId`、`requestId`、`isSidechain`、`cwd`、`message.id`、`message.model:"claude-fable-5-1"`、`message.usage{input_tokens, cache_creation_input_tokens, cache_read_input_tokens, cache_creation{ephemeral_5m_input_tokens, ephemeral_1h_input_tokens}, output_tokens}`。抽样行中无 `costUSD`。 | 持久事实：字段列表本身，已与 `parse_transcripts.py:24-38` 交叉核对。2026-09-25 抽样于 `/home/box/.claude/projects/-workspace-claude-mem--claude-worktrees-cost-report-weekly/0e3f6cc9-3feb-4e4a-b20e-cc3704ef868d/subagents/agent-aa1975f524bd06ab3.jsonl:13`；该文件可能被清理（30 天保留），因此用 `grep -m1 -l cache_creation ~/.claude/projects/**/*.jsonl` 在当前任意 transcript 上重新验证 |
| 1h 缓存写入在数据中真实存在：802 行中有 259 行 `cache_write_1h > 0` | `out/transcript_usage.json`（已计数） |
| 价格：`GET https://openrouter.ai/api/v1/models`，无需密钥；存储 `prompt`、`completion`、`input_cache_read`、`input_cache_write` × 1e6 为 USD/MTok，以 OpenRouter id 为键 | `fetch_prices.py:3-7`；`out/openrouter_prices.json` 样本 `anthropic/claude-fable-5.1 = {input 10.0, output 50.0, cache_read 0.25, cache_write 12.5}` |
| 模型键规范化 `norm()` 和价格回退 `rate()`（缺失时 cache_read = input×0.1，cache_write = input×1.25；以 `<` 开头的模型不计价） | `weekly_report.py:23-42` |
| 成本公式：`(input×in + output×out + cw5×cache_write + cw1h×in×2 + cache_read×cache_read_rate)/1e6`，整数微美元 | `weekly_report.py:91-105` |
| 实时 OpenRouter 数据现在在某些模型上暴露额外的 `input_cache_write_1h` 键（在 `anthropic/claude-opus-5.5` 上见到），`fetch_prices.py` 未存储 | 子代理对 `/api/v1/models` 的实时探测；不在 https://openrouter.ai/docs/guides/overview/models 的文档字段列表中 |
| 只读快照模式：用 `file:...?mode=ro` URI 打开实时 DB，`src.backup(dst)` 到 `snap/snap.db`，只在快照上工作 | `weekly_report.py:15-18` |
| 会话来自 `sdk_sessions` 按 `started_at_epoch`；设备 = 在 observations、session_summaries、user_prompts 上 `coalesce(origin_device_id,'local-box')` | `weekly_report.py:45-53` |
| Ship 正则和已完成摘要规则（`len(completed) > 20`） | `weekly_report.py:65,74-79` |
| Observer token 按 `(memory_session_id, created_at_epoch, discovery_tokens)` 去重，仅按 input 费率计价，单独保留为 `observer_note_taker_est_usd` | `weekly_report.py:70-73,229` |
| `CAT_RULES`（Incident、Experiment、Investigation、Maintenance、Bug fix、Feature；默认 Investigation）、`FAIL_RULES`（Rework、Recovery after miss、Wrong turn）、Looping = 相同 80 字符提示前缀 ≥3 | `weekly_report.py:109-127` |
| `active_minutes(stamps, gap_min=15)` | `weekly_report.py:128-133` |
| Mac 外推：`ratio = measured agent USD per 1M observer tokens` 在 transcript 会话上，应用于无 transcript 的会话，标签以 `EXTRAPOLATED (low confidence)` 开头 | `weekly_report.py:239-246` |
| 摘要 schema 和写入器（`summary.json`、`projects.csv`、`sessions.csv`、`models.csv`、`tables.json`） | `weekly_report.py:207-251` |
| 需删除的硬编码：`/workspace/reports/mem-invoice/PRICE-TABLE.json`（`:21`）、设备 UUID（`:54-56`）、`/tmp/gateway-server.log`（`:158-168`）、`cloud_page_stats.py:12` 中 DST 脆弱的 `-7h` 偏移 | 如引用 |
| 研究数字在 `out/summary.json` 中确认：`sessions` 80、`substantive_sessions` 47、`projects` 23、`seats` 3（box 23、remote A 52、remote B 5）、`outcomes.sessions_with_completed_summary` 26、`outcomes.ship_events_distinct` 5、`hours.agent_session_hours` 13.8、`tokens.agent_tokens_measured_box_transcripts` 81,783,140、`spend.agent_api_equivalent_est_usd` 109.25、`spend.extrapolated_unmeasured_sessions_usd` 54.69、`spend.weekly_total_api_equiv_est_usd` 163.94、`spend.observer_note_taker_est_usd` 5.84、`window.end_exclusive` "2026-09-26 00:00 PT" | `out/summary.json` |
| 1.9 美分的错误：旧报告计价了 `discovery_tokens`，即记录者自身的 token | `NOTION-PAGE.md:5,8,23,34` |
| 差异：`WORKFLOW.md:20` 说按文件名连接，代码按 transcript `sessionId` = `content_session_id` 连接 | `parse_transcripts.py:32`、`weekly_report.py:137` |
| 差异：研究窗口是 8 天（Sep 18–26 不含），但 Notion 页面写"Sep 18–25" | `WORKFLOW.md:8`、`NOTION-PAGE.md:5,53` |

#### B. Timing 风格设计

| 事实 | 引用 |
|---|---|
| 区块顺序：主数字 + 副行 + 故事；成本条带；"Where the money went" 环形图 + 排名列表；"Day by day" 堆叠柱，空日说明；"How much was useful" 环 + 检查项 + 一句失败说明；"What got done" 行按最贵优先；"Worth your attention" ≤3 个行动；折叠 Details（证据 ID、会话 ID、token、模型和每 MTok 价格、置信度、风险、失败表、CSV） | `/workspace/timing-report-brief/brief.md:33-42` |
| 诚实规则：每个美元数字带 ESTIMATE 标签；"Measured spend unavailable" 绝不显示为 $0；页脚说明方法；未完成工作不计为完成；失败被点名；证据 ID 保留在 Details | `brief.md:53-59` |
| 自包含 HTML，无外部依赖 | `brief