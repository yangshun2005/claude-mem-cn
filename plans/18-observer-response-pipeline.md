<!-- GitHub Issue #3606 的镜像。Issue 是公开追踪器；本文档是设计文档。
     若两者不一致，设计以文档为准，状态以 Issue 为准（oh-my-issues 健康检查）。 -->

> **追踪器：** https://github.com/thedotmack/claude-mem/issues/3606

# [plan-18] Observer 响应流水线 — 分类每个输出，失败时绝不确认批次，为每段历史设上限

> **追踪器：** #3606 · **设计文档：** `plans/18-observer-response-pipeline.md` · **系列：** plan-14 #3602 · plan-15 #3603 · plan-16 #3604 · plan-17 #3605 · plan-18 #3606 · plan-19 #3607 · plan-20 #3608 · plan-21 #3609 · plan-22 #3610 · plan-23 #3611

## 缺陷

`ResponseProcessor` 实际上只有一个桶来处理"模型没有返回格式良好的 `<observation>`/`<summary>`/`<skip_summary/>` XML"，而这个桶会**确认已认领的批次**（`confirmClaimedMessages()`）——因此每个非 XML 回复都是静默数据丢失，流水线的各项对策要么失效，要么自相矛盾：

- 跳过契约不匹配：`modes/*.json` 的 `skip_guidance` 说"返回空响应"；解析器只接受 `<skip_summary reason=…/>`；空响应 → `idle`，"no observations to record" → `prose`。两者都被计入 `consecutiveInvalidOutputs >= 3` → "SDK 会话已中毒——杀死并重启"（每天数百次重启，`conversationHistory` 被清空，批次从未重新投递）。13.11.0 移除了升级逻辑，但在五处把计数器赋值为 `0` 且从不递增，静默丢弃也依然存在。
- 认证失败（`Failed to authenticate 403/401`、`Not logged in`）、模型错误（`There's an issue with the selected model`）、传输错误（`Connection closed mid-response`）、上下文溢出（`Prompt is too long` 以助手*文本*形式到达，而非抛出异常）、XML Schema 漂移（用 `<kind>/<detail>` 而非 `<type>/<title>`）、以及带散文但无字段的闭合 `<observation>` 块，全都被分类为 `prose`/`xml` 并丢弃；只有配额和（最近的）认证有保留工作的分支；格式错误的轮次在验证*之前*就被推入 `conversationHistory`，因此漂移会在会话剩余时间里自我强化。
- 空请求：生成器在 `queueDepth=0` 时自动启动，发送空的 INIT 轮次，收到 `""` 返回，然后丢弃已认领的消息；`SessionManager` 在会话不在内存映射中时丢弃已存储的 `memory_session_id`（"Issue #817"），即使 worker 从未重启，因此 180 秒的空闲驱逐会让下一批重新走 INIT，模型的回复就此被消耗。
- 任何地方都没有 token 预算：`session.conversationHistory` 只增不减（`ClaudeProvider.ts:490/528/552`、`OpenAICompatibleProvider.ts:105/215/265`、`ResponseProcessor.ts:294`），且 #3096 移除了 OpenAI 兼容路径的 `truncateHistory()`；保留的 OpenRouter 版本在单条消息超过 `MAX_TOKENS` 时返回 `[]`（API 返回 400，observation 被标记为失败）；OpenAI 兼容 provider 把每个助手回复**追加两次**；`max_tokens: 4096` 是硬编码的，`finish_reason=length` 从不记录（76% 的响应被截断，尾部块丢失，退化重复被 `content_hash` 掩盖）；base64/图片载荷和无界的逐字段工具输出直接进入提示词；因认证暂停的缓冲区（`buffers=new Map`，仅由 `confirm()` 移除）增长到 GB 级；约 10 倍的 token 放大以 DEBUG 日志级别计入用户订阅账单。
- 输出质量无人把关：`type` 枚举被记录为无效后仍原样存储（或默认为 `bugfix`）；写入时去重仅基于字节完全相同的 `content_hash`，因此改述的重复内容会淹没召回窗口；推理模型合法的空 `content` 加 `reasoning_content` 被当作初始化失败；provider 设置无论当前激活的是哪个都会被全部校验，因此一个已退役的 Gemini id 会阻止所有设置保存。

修复方案是一个显式分类器加一张策略表：每个输出类别映射到 {保留批次 | 确认批次 | 带原因中止会话}，且每段历史/缓冲区/响应都有预算。

## 子项

- #2817 — v13.4.0：尽管有 plan-11 的恢复机制，observer 仍会在散文式空响应上中毒循环
- #2866 — Observer 在"Prompt is too long"时静默丢弃 observation——剥离图片/base64 载荷
- #2906 — 洪泛队列：单条消息中出现超过 100 个嵌套 observer 包
- #2915 — 通过 cc switch 将 Claude Code 切换到智谱 GLM 后，只记录提示词，从不记录回答（模型错误被分类为 prose）
- #2935 — 生成器中毒循环：提示词说"返回空响应"，但解析器只接受 <skip_summary/>
- #2955 — SDK 会话中毒循环：生成器输出散文式"empty observation"而非 <skip_summary/>
- #2956 — Claude provider（SDKAgent）没有上下文边界——observer 溢出，长会话中保存零记忆
- #2958 — 追踪：Claude provider observer 溢出上下文窗口——历史大小无上限
- #2960 — 追踪：observer 中毒循环——跳过散文被计为无效
- #2995 — 支持为 OpenRouter / OpenAI 兼容 provider 禁用推理（推理会吃掉 `max_tokens`）
- #3038 — 近似重复的 observation 累积：content_hash 只去重字节完全相同的行
- #3083 — Windows 可靠性：13.8.1 中仍存在中毒循环（生命周期部分 → plan-15）
- #3163 — 捕获侧去重：标题级模板化 observation 绕过 content_hash 并淹没召回窗口
- #3188 — 在 CLI 安装器和设置 UI 中暴露 BYOK / 自定义 OpenAI 兼容端点
- #3193 — Observer 在正确的空批次上中毒：礼貌的"ready/waiting"散文被计为三振无效
- #3351 — 数据丢失：带非结构化散文的闭合 <observation> 块被永久丢弃（无抢救，无重试）
- #3443 — 自 #3096 起 observer 历史再次无界：OpenAI 兼容 provider 路径上出现 100 万 token 的单次调用
- #3454 — Observer 丢弃 63% 的轮次——每天 170 次空（idle）响应，加上失效的 consecutiveInvalidOutputs 计数器
- #3457 — 当 settings.json 中残留已退役的 Gemini 模型时，查看器设置永远无法保存——且 UI 隐藏了原因
- #3461 — Observer XML Schema 漂移导致会话剩余时间内永久、静默的 observation 丢失
- #3490 — `max_tokens` 硬编码为 4096，静默截断 76% 的 observation 响应
- #3491 — OpenRouter truncateHistory 在单条消息超过 MAX_TOKENS 时返回空数组 → 400 并丢失 observation
- #3492 — Observer 永远无法离开 INIT 提示词：空闲驱逐在从未重启的 worker 上丢弃 memory_session_id
- #3495 — Claude-mem 记录死胡同和错误的记忆
- #3497 — Observer 的 conversationHistory 从不裁剪——约 10 倍 token 放大被静默计费
- #3499 — OpenAI 兼容 provider 把同一助手响应追加到 conversationHistory 两次
- #3560 — 按命令模式过滤 Bash 捕获（SKIP_TOOLS 太粗）——捕获门控作为 token 预算
- #3569 — OpenRouter 初始化在推理模型的空 content 上失败（DeepSeek V4 Flash）
- #3588 — 认证失败 → 缓冲的 observation 永不排空：单 worker 堆内存达到 5.6 GB
- #3589 — Observer 上下文溢出被分类为 prose 并静默丢弃批次
- #3592 — Observation `type` 枚举未强制执行：解析器原样存储无效类型；缺失 `<type>` 默认为 `bugfix`

待评估/变基的相关 PR：#3460（传输错误分类器）、#2957（SDKAgent 滑动窗口）、#2884、#2943、#2927、#2857、#2905、#1775（散文抢救）、#3352（提示词：检查先前输出）、#3207（认证散文保留批次）、#3283（已退役模型 id）。

## 修复顺序

1. **一个分类器，一张策略表。** `sdk/output-classifier.ts` 返回一个封闭枚举——`valid`、`skip`、`empty_request`、`idle`、`auth`、`quota`、`model_error`、`transport`、`overflow`、`xml_drift`、`prose_salvageable`、`prose`——`ResponseProcessor` 通过一张表分派：对 `empty_request|idle|auth|quota|transport|overflow|xml_drift` 保留批次（`resetProcessingToPending`），对 `valid|skip` 确认，对 `prose_salvageable` 先抢救再确认（标题 = 前 100 字符，叙述 = 文本，WARN），对真正的 `prose` 带 ERROR 确认。任何内容在通过验证之前都不会进入 `conversationHistory`；`xml_drift` 弹出坏轮次并重新发送 Schema。`consecutiveInvalidOutputs` 要么计数并在 3 次后带类型化 `abortReason`（`drift:*`、`overflow:*`）中止，要么删除——而不是赋值为 `0` 并记录日志。
2. **跳过是哨兵，不是沉默。** 每个模式的 `skip_guidance` 都输出 `<skip_summary reason="…"/>`（observation 轮次则输出 `<skip_observation/>`）；解析器两者都接受；生成器绝不在 `queueDepth=0` 时发送轮次；INIT 提示词仅在 `bootId` 变化时发送（plan-15 所有者记录）——已存储的 `memory_session_id` 在同一 worker 实例上恢复，而不是在空闲驱逐时被丢弃。
3. **Token 预算作为一等对象。** `ObserverBudget {maxHistoryTokens（模型上下文的比例）、maxFieldChars、maxToolOutputChars、maxOutputTokens}` 由所有 provider 共享；历史被裁剪（压缩较旧的一半 → 摘要）但绝不清空（`truncateHistory` 保留 ≥ 1 条消息并截断内容）；移除重复追加；base64/图片 blob 在摄入时替换为 `[image omitted: type, N KB]`；`max_tokens` 可配置并与 `CLAUDE_MEM_API_TIMEOUT_MS` 耦合；`finish_reason=length` 和重复的 `obsIds` 记录 WARN；累计输入/输出 token 记录 INFO 并在 `/api/health` + 状态栏上暴露；认证暂停的缓冲区设上限并溢出到磁盘，超过上限后停止接收。
4. **Provider 适配器对齐。** 统一的 `OpenAICompatibleProvider` 配置面：base URL、`max_tokens`、`reasoning: {effort}`（仅在支持时）、带 `reasoning_content` 的空 content 视为无操作、`SettingsRoutes` 只校验当前激活 provider 的字段并在加载时规范化已退役 id；查看器保存处理器暴露 JSON 错误体；安装器/设置 UI 暴露 base URL。
5. **输出质量门控。** 未知 `type` → 强制转换为中性类型，原始值存入 `metadata`（枚举在逐工具使用模板中重申）；缺失标题被拒绝；写入时按（标题、项目）规范化去重并带 `occurrence_count`（可选的模糊层仅用于审查）；`CLAUDE_MEM_SKIP_BASH_PATTERNS` 与 `SKIP_TOOLS` 在同一门控；observation 提示词中加入一行"检查你之前的输出"。

## 测试矩阵

| Provider | 输出 | 要求行为 |
|---|---|---|
| claude (SDK)、openrouter、gemini、openai-compatible | 空请求时返回 `""` | 类别 `empty_request`；批次保留；无 INIT 反复；无日志刷屏 |
| 全部 | `<skip_summary reason=…/>` / `<skip_observation/>` | 确认；零 ERROR 行 |
| 全部 | "No observations to record" 散文 | 类别 `idle`；批次保留；计数器不变 |
| 全部 | 401/403/`Not logged in`/`OAuth session expired` 文本 | 类别 `auth`；批次保留；会话暂停并带 ERROR + 健康标志 |
| claude | `Prompt is too long` 作为文本 | 类别 `overflow`；批次保留；历史重置；预算减半后重试一次 |
| 全部 | `<observation><kind>…` 漂移 | 类别 `xml_drift`；轮次弹出；重新发送 Schema；3 次后以 `drift:observer_schema` 中止 |
| 全部 | 闭合 `<observation>` 带 1,000 字符散文 | 抢救为叙述；WARN |
| 全部 | `type=verification`（不在枚举中） | 存储为中性类型 + metadata；绝不默认 `bugfix` |
| openrouter | 单条消息 > MAX_TOKENS | 请求仍有 ≥ 1 条消息；内容截断；无 400 |
| openai-compatible | 200 轮 | 历史 token ≤ 预算；每个助手回复恰好出现一次 |
| openrouter/gemini | `finish_reason=length` | 记录 WARN；部分块抢救；不盲目重试 |
| 全部 | 因认证暂停 50 小时 | 缓冲区 ≤ 上限；堆内存平稳；`/login` 后 observation 排空 |
| 全部 | 模型返回先前标题的改述 | 去重递增 `occurrence_count`，不新增行 |
| claude、gemini | provider=claude 时保存含已退役 gemini id 的设置 | 200；id 被规范化 |

该矩阵在 CI 中运行（`tests/sdk/output-classifier.test.ts` + `tests/services/worker/ResponseProcessor.test.ts`，配合录制的 provider 转录）。回归必须在用户提交之前就让 CI 失败。

## 范围之外

observer 子进程如何启动、认证、隔离，以及其转录存放位置（plan-19）。observation 归档到哪个项目下（plan-20）。队列/会话状态持久化与重试（plan-21）。


---

## 轮次：2026-09-12 oh-my-issues

- **#4066** — 预算回收与模型竞速。`MessageBuffer.drain()` 让出时不限速也不等待响应，预算统计的是*已发送但未应答*的字符，触发时会中止正在进行的回答；`resetProcessingToPending()` 重新投递完全相同的批次，因此循环零进展，直到 2 次回收后的 10 分钟暂停。证据：三个循环均为 `400265 chars / discardedMessages=100`、`historyLength=0`、`queueDepth=105 → 105`。降低预算只会更糟。修复：背压（未应答尾部已达预算时不再让出），且预算回收时不中止进行中的轮次；仅当回收未产生已确认回答*且*队首消息单独就超过预算时，才将其计入 #2468 暂停。
- **#3897** — observer 把未经检验的假设写成已完成的、带 `gotcha` 标签的发现，且当会话在 21 秒后推翻它们时无法修正；`build_corpus` 随后以同等权威提供错误内容。原因在输出契约：没有类型或概念表达暂定性，`recording_focus` 强制平铺直叙，`field_guidance` 剥离模糊措辞，`observer_role` 排除了修正。第 0 步是 A/B——**(A)** 通过搜索和语料渲染传递 provisional/`unverified` 标记（提示词 + 验证器，落在本计划）或 **(B)** 同会话 `supersedes` 链接（列 + 迁移在 plan-21，渲染在 plan-25）。建议：先发布 (A)。
- **#4062，残余部分**（崩溃本身属于 plan-21）：从 `startSession` 抛出的非 `auth:`/`quota:` 异常只被外层 `.catch` 捕获，并**终结**会话而非暂停，损失整个待处理窗口；且对已确认无法刷新的凭据没有退避（同一个过期的 OAuth token 每 15–90 秒重试一次，持续约 10.5 小时）。