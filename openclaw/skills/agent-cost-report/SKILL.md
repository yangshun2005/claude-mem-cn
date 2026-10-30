---
name: Agent Cost Report
description: >-
  为任意时间段生成可信的 Agent 成本报告，默认最近 7 个完整 PT 日，不含今天。
  以 OpenRouter 公开标价对 Claude Code 转录记录中的实测 token 计价（ESTIMATED），
  在有合规来源时使用实测供应商支出，记录者成本单独列出，输出 Timing 风格 HTML/PDF 以及
  report.json、line-items.csv、evidence.json。
allowed-tools:
  - Bash
  - Read
  - Write
  - AskUserQuestion
  - mcp__plugin_claude-mem_mcp-search__search
  - mcp__plugin_claude-mem_mcp-search__timeline
  - mcp__plugin_claude-mem_mcp-search__get_observations
---

# Agent Cost Report

**Claude-Mem / Claude Code 技能。** 运行时是 `scripts/` 流水线（转录记录 → token → 美元 → Timing 风格报告），外加一轮渐进式 Mem Search 复核，用于确认草拟的标签。Notion 草稿仅为 SPEC 历史记录——不是产品，不是运行时，也不是交付载体。

解析包含此 `SKILL.md` 的绝对目录；所有辅助路径均相对于该目录。`${CLAUDE_SKILL_DIR}` 是快捷方式：`python3 "${CLAUDE_SKILL_DIR}/scripts/acr.py" …`。仅使用 Python 3.9+ 标准库，并需要 `America/Los_Angeles` 的 IANA 时区数据；不安装 pip 包。`tzdata` 是"不安装 pip 包"规则的唯一例外，且仅在 `acr.py` 退出并提示时区数据不可用时才安装（Windows 自带的 Python 不包含时区数据）：先向用户展示它打印的行，并在安装前询问（AskUserQuestion）。若用户同意，运行适用于你 shell 的那一行（Bash 或 cmd 中使用带引号的普通行，PowerShell 中使用 `PowerShell:` 行），然后重新运行失败的命令。改为设置 `PYTHONTZPATH` 则无需安装。外观样式位于 `scripts/acr/render.py`，绝不在此处。

## 目的

将 Claude-Mem 活动转化为**管理者可读的成本与失败报告**。

产品构想：一个可复用技能，通过 Mem Search 搜索 Claude-Mem，重建真实工作单元，分配成本与失败类别，并渲染可打印报告。

## 洞察北极星

头条是美元金额，保留两位小数，并带标签。美元金额来自 Claude Code 转录记录（每次回复的精确 token 用量），按 OpenRouter 公开标价计价，因此是 **ESTIMATED**。实测供应商支出仅在存在合规来源时出现。美元金额正下方：错误造成的成本、已交付成果，以及同一时间轴上的两者。

## 报告必须回答的问题

1. 完成了哪些工作？
2. 每个成果花了多少钱？
3. 因循环、含糊其辞、走错方向、返工或路由不当浪费了多少？
4. 是否尝试过任何未授权操作？
5. 管理者下一步应改变什么？

**主要单位 = 每个已完成成果的成本**（不是每个 observation 的成本）。

## 何时使用

- "Agent 成本报告" / "每个成果的成本" / "失败经济学" / "这次会话值不值" / "本周 agent 花了多少钱"
- 在真实的 Mem 会话挖掘之后，领导层需要成果经济学时
- 需要自包含 HTML + JSON + CSV + 证据的示例 / 交付包

记忆挖掘机制：claude-mem 的 `mem-search` 技能（渐进式召回）。

## 默认范围（始终）

- 除非用户指定**特定会话 / 范围 / 项目**，窗口为 **PT 时区最近 7 个完整日，不含今天**：`end` = 今天开始的 PT 午夜（不含），`start` = `end − 7 天`。默认窗口绝不包含不完整的一天（G3，Alex 2026-09-25）。
- 显式窗口：`--start YYYY-MM-DD --end YYYY-MM-DD`（PT 日历日，end 不含）。显式 `--end` 晚于今天时，最后一天标记为"部分，生成于 HH:MM PT"。
- 单个会话：`--session <content_session_id>`。单个项目加时间段：`--project <name> --start … --end …`（包含该项目的 worktree）。
- 报告 **Scope** 条显示 PT 范围，Details 列出范围内每个会话 id。

## 渐进式 Mem Search（始终）

遵循 claude-mem `mem-search` 的三层。保持低开销。

1. **Search** — 获取 ID 索引（标题、类型、token 提示）。
2. **Timeline** — 仅围绕你关心的锚点。
3. **Observations** — 对将作为证据引用的筛选后 ID 使用 `get_observations`。

流程：

1. 解析范围（默认：最近 7 个完整 PT 日）。
2. Search → 收集 ID。
3. 仅在上下文不足时使用 Timeline。
4. 对 intended / actual / outcome / waste / rework / blocked / unauthorized / status 使用 Observations。
5. 分组为命名工作项 + 失败事件。
6. 计算行项目成本。
7. 渲染 HTML + 可选 PDF（+ json/csv/evidence）。
8. 将证据 ID 保留在附录中——不要把完整时间线倾倒进主报告。

ADHD 流程要点：

- 先 Search → 选 ID → 仅在上下文不足时用 timeline → 只获取需要的 obs。
- 工作项标题为管理者而虚构（"Chroma 崩溃循环后恢复搜索"）；observation 标题仅作证据。
- 证据附录列出 obs ID + 短标题；主章节保持成果优先。

在此技能中，搜索环节是**复核步骤**（见流程第 4 步）：流水线根据关键词草拟类别和失败类型（`label_source: keyword`）；编排器根据引用的证据 ID 确认或修改每个行项目的 `category` 和 `failure_type`，并用 `review --apply` 应用结果。未复核的项目保留"草稿标签"标记，页脚会统计它们。

## 工作类别（始终）

`Feature` · `Bug fix` · `Incident` · `Maintenance` · `Investigation` · `Experiment`

## 失败 / 浪费类型（始终）

`Looping` · `Hedging` · `Wrong turn` · `Rework` · `Regression` · `Premature completion` · `Unauthorized action` · `Suboptimal path` · `Duplicate work` · `Blocked work` · `Missed requirement` · `Unnecessary escalation` · `Context re-read` · `Model thrash` · `Fan-out waste` · `Recovery after miss`

**返工锁定（始终）：** Rework **仅**存在于 `failure_type` 下——绝不作为工作类别。类别保持为预期工作类型；发生返工时设置 `failure_type: Rework`。

一个行项目可以同时有工作类别**和** failure_type（例如 Maintenance + Looping）。

## 成本模型

实测 token 来自 Claude Code 转录记录（`~/.claude/projects/**/*.jsonl`，assistant 回复按 `(message.id, requestId)` 去重）；Codex 转录记录以相同方式读取。价格为 OpenRouter 公开标价（每百万 token），运行时获取并随报告保存。

```
agent_cost_i (per reply, micro-dollars) =
      input × price.input + output × price.output
    + cache_write_5m × price.cache_write + cache_write_1h × price.cache_write_1h
    + cache_read × price.cache_read                    # cache_write_1h = listed rate, else 2 × input

agent_estimated_usd        = Σ agent_cost_i over every reply in the window (matched or not)   # ESTIMATED, the headline
extrapolated_unmeasured    = observer tokens of sessions with no transcript here × (measured $ per observer token)
                           # EXTRAPOLATED (low confidence); "all sessions measured" when nothing remains
observer_note_taker_est    = note-taker (observer) tokens, deduped per reply, × its input list price
                           # priced separately, never agent cost, never in the headline
mistakes_estimated_usd     = Σ agent_cost_i over the same-session union of wasted turns, each turn once   # low figure
cost_per_completed_outcome = Σ attributed $ for status ∈ {shipped, completed} / count(those work items)
waste_rate                 = Σ wasted_cost / Σ attributed $     recovery_share = Σ recovery_cost / Σ attributed $
```

行项目是会话：`attributed_usd` = 估算（本机有转录记录）或外推（无转录记录）。`wasted_cost` 和 `recovery_cost` 来自行为分析环节（见下文），为 ribbon、行项目和错误行使用同一个并集。上限（重做窗口加项目级回退）保留在 Details 中。

**未授权且被阻止：** `direct_cost` $0，`risk_exposure` high，`action_status` blocked。始终让 risk_exposure 保持非美元，除非涉及真实现金/补救——绝不虚构风险美元金额。

`confidence` — token + 模型 + 成果清晰时为 `high`；跨 obs 分配需主观判断时为 `medium`；证据薄弱时为 `low`。

未定价模型（不在价格列表中，或价格为负的"variable"）按名称列出其 token，不增加金额；绝不静默按零计价。

## 行为指标（复核前为启发式）

对同一批转录记录进行第二轮分析，将每个用户轮次标记为 human / bot / unknown（转发的 agent 提示绝不是 Alex 的话），发现挫败事件，并运行 Frustration Arc 研究中的模式检测器：虚构人工门禁、破坏可用功能、错误或昂贵模型、过度工程、做了未要求的事、虚假输出、虚假"完成"、错误工具或联系人、不良外发（事件 × 收件人，绝不用美元）、记忆或规则丢失、术语堆砌、原因不明，外加工具错误和含糊其辞（Alex 的定义：答案已可得时仍给出保留意见）。四个摘要卡片；其余在 Details 中。每个计数在复核或分类前都标记为 `heuristic`。可选分类器（`--classify`）默认关闭，每次运行上限 $2.00，仅使用常规推理 `OPENROUTER_API_KEY`，其支出单独显示。

## 金额标签（始终）

| 标签 | 含义 |
|-------|---------|
| **Measured** | 来自合规来源的供应商报告支出（目前：OpenRouter 按 key 快照，显示其 UTC 桶标签） |
| **Estimated** | 实测转录 token × OpenRouter 每 MTok 标价（上述公式） |
| **Extrapolated** | 本机无转录记录的会话，按 observer token 比例推算；始终"低置信度" |
| **Unavailable** | 无实测值——写 `measured spend unavailable`，未知绝不写 `$0 spent` |
| **risk_exposure** | 严重程度 / 定性，除非涉及真实现金（席位、退款、SLA） |

每个行项目的 `cost_basis`：`estimated_usage`、`measured_provider` 或 `extrapolated`。所有美元金额均保留两位小数（`$109.25`）；每个数字都带标签和依据。

## 行项目 Schema

`line-items.csv` / `report.json.line_items` 中的每一行：

| 字段 | 说明 |
|-------|------|
| `work_item_id` | 稳定 id（`WI-1` …） |
| `title` | 人类可读工作项标题（复核时始终虚构；草稿使用会话首个提示） |
| `scope` / `project` / `worktree` | 时间段、会话或项目；主项目；相关时的工作树 |
| `session_ids` / `content_session_id` | Mem 会话 id 和转录记录关联键 |
| `status` | `shipped` / `completed` / `in_progress` / `abandoned` / `blocked` |
| `category` / `work_category` | 六个工作类别之一 |
| `failure_type` / `failure_signals` | 空或失败类型（Rework 在此） |
| `cost_measured` | 数字或 `unavailable` |
| `cost_estimated` / `cost_extrapolated` / `attributed_usd` | 标价估算；无转录记录会话的外推；实际使用值 |
| `wasted_cost` / `recovery_cost` / `productive_cost` | 来自行为并集的美元（无则 0） |
| `risk_exposure` | `none` / `low` / `medium` / `high`（非美元，除非真实现金） |
| `evidence_ids` / `summary_ids` | Observation 和 summary ID |
| `agent_tokens` | `{input, output, cache_write_5m, cache_write_1h, cache_read}` |
| `observer_tokens` | 记录者 token，仅 Details |
| `model` / `model_prices_usd_per_mtok` | 主导模型及所用价格 |
| `cost_basis` | `estimated_usage` / `measured_provider` / `extrapolated` |
| `label_source` | `keyword`（草稿）/ `llm` / `human`，附 `reviewed_by` 和 `reviewed_at_pt` |
| `device` | `local`、`remote-N`，或导出标签如 `mac` |
| `behavior_counts` | 此会话中的模式命中 |
| `recommended_action` / `confidence` / `date_pt` / `notes` | 一个杠杆；high/medium/low；PT 日；一句平实的话 |

## 交付物（始终）

`acr.py` 写入一个目录，包含：

1. **`report.html`** — 自包含（内联 CSS，无脚本，无外部资源），Timing 风格
2. **`report.pdf`** — 由 `report.print.html` 通过无头 Chrome 生成（PATH 中的 `google-chrome`，否则 macOS 和 Windows 上 Chrome 的标准安装位置）；Chrome 缺失时运行提示"PDF skipped, HTML is canonical"
3. **`report.json`** — window、scope、spend、totals、by_day、by_model、by_device、line_items、wins、behavior、timeline
4. **`line-items.csv`** — 每个工作项一行
5. **`evidence.json`** — 引用的 observation ID、短标题、observer token、模型、会话 id
6. **`labels.review.json`** — 供复核环节使用的草拟标签

### 面向管理者的 HTML 章节（必需顺序）

1. **Hero** — 带标签的美元金额（ESTIMATE / MEASURED）、依据句、"完成 N 项，每项约 $X"、外推行、实测供应商支出不可用时的"Measured provider spend: unavailable"，以及故事句（计算得出，绝不手写）
2. **成果 vs 错误** — 错误成本行（低值）、已交付成果（已合并 PR、已发布版本、好评；在会话与 PR 关联前成本为"unmeasured"），以及同一 PT 日轴上的两条时间线
3. **成本 Ribbon** — 每项工作为一个按成本大小的块，浪费部分斜线填充，进行中条纹填充
4. **钱花在哪**（按工作类型环形图）· **逐日**（堆叠柱，空日显示"no agent work"）· **多少有用**（环形图，行为条最多四个卡片）
5. **完成了什么** — 每个工作项一行可折叠，最贵在前，草稿标签标记
6. **值得关注** — 最多三条，计算得出
7. **Details & evidence**（折叠；PDF 中展开）— 成果、按日错误、带上限的行为模式表、规则有效性、支出与定价、模型、失败核算、行项目、设备、未匹配转录记录、未定价模型、标签计数、诚实规则、CSV 和 JSON 链接

## 真实性始终规则

仅正向真实规则：

1. 始终标注估算 vs 实测（`cost_basis` = `estimated_usage` 或 `measured_provider`）。
2. 未匹配时始终写 **measured spend unavailable**——未知绝不写"$0 spent"。
3. 始终以**已完成成果**为单位（每个已完成成果的成本）。
4. 始终让 `risk_exposure` 保持非美元，除非涉及真实现金/补救。
5. 始终虚构人类工作项标题；observation 标题仅作证据。
6. 始终渐进式 Mem Search；证据 ID 放在附录。
7. 始终将 Rework 仅放在 `failure_type` 下。
8. 除非用户指定其他范围，始终默认最近会话。

重建新增，同一精神：记录者成本单独列出，绝不进头条；关键词标签在复核前是草稿；Grok Bot 和 Mac 数字为"unavailable"或"extrapolated (low confidence)"，绝不为 $0，绝不猜测；实时数据库始终只读打开（每次运行都在快照上工作）。

## 流程（编排器）

所有命令从技能目录运行；`OUT` 是一个运行目录（例如 `/tmp/acr-weekly/run`）。每一步都是幂等的。

1. **价格** — `python3 scripts/acr.py prices --out $OUT`（OpenRouter 标价；`--prices <saved prices.json>` 离线运行；无需 key）。
2. **收集** — `python3 scripts/acr.py collect --out $OUT`（默认窗口）或 `… --start YYYY-MM-DD --end YYYY-MM-DD`、`… --session <id>`、`… --project <name> --start … --end …`。
3. **汇总** — `python3 scripts/acr.py rollup --out $OUT`，使用相同时间段参数。写入 report.json、line-items.csv、evidence.json、labels.review.json。选项：`--no-gh`（跳过只读 `gh pr view` 对已合并 PR 的确认）、`--classify`（见上文）、`--rules-dir <house rules>`（规则有效性）。
4. **复核** — 对 `labels.re