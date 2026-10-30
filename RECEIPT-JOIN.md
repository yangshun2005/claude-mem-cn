# RECEIPT-JOIN — tool_uses ↔ 费用统计

**时间戳：** 2026-09-06 (PT)，实现于 2026-09-07 落地 · Tool SQLite ↔ Receipt
**状态：** 已发布 — schema v51 与本契约一致；`tool_uses` 已存在于 `~/.claude-mem/claude-mem.db`
**职责划分：** Claude-Mem `tool_uses` = 身份 / DISTINCT COUNT。OpenRouter 时间戳 + 消费日志 = **$**。切勿将 `cost_usd` / 微单位作为权威数据写入 `tool_uses`。

对于按 token 数 × 公开标价（无生成 id）计算的观测 token 费用报告，请使用 `docs/expense-pricing/`。

---

## 冻结的连接键

| Claude-Mem `tool_uses` | Receipt / 消费日志 | 用途 |
|---|---|---|
| `content_session_id` | ↔ `or_session_id`，按下文约定 | 会话/任务分组 |
| `tool_name` | ↔ 统计 `groupBy: 'tool'` 标签 | 功能/工具汇总 |
| `tool_use_id` | 优先用 DISTINCT COUNT，而非 `or_trace.tool_calls` 整数 | 使用次数 |
| `project` | 过滤 | 范围 |
| `agent_id` | 过滤 | 席位/agent |
| `created_at_epoch` | 周期 | 时间窗口 |
| `observation_id`（可空） | 后续可选 | Claude-Mem 反向引用。**批次指针，非 1:1** — 见下方注意事项 |
| `or_generation_id`（可空） | ↔ 消费日志 / Ledger `or_generation_id`（`gen-…`） | 当工具处于带时间戳的 OR/推理调用内时，作为**现金行连接** |
| `or_session_id`（可空，精确时间戳回显） | ↔ 消费日志 `or_session_id` | 存在时优先使用，而非重建的 `job:`+id |

**不要添加：** `tool_uses` 上的 `cost_usd` / 微单位。

---

## `content_session_id` ↔ `or_session_id` 字符串格式

Receipt 通过 `buildOrStamp`（`/workspace/mem-invoice/src/or/stamp.ts`）为 OpenRouter 打时间戳：

```text
or_session_id  =  "job:" + runId     # ≤256 chars; body field session_id
```

Claude-Mem Hook 将平台会话 id 存为 `content_session_id`（Claude Code / Cursor / grok-bot 会话 UUID — **无** `job:` 前缀）。

### 必需的时间戳约定（以便二者匹配）

当付费 OR 调用关联到 Claude-Mem 会话时，打时间戳方**必须**设置：

```text
runId = <claude-mem content_session_id>
⇒ or_session_id = "job:" + content_session_id
```

**连接方法（COUNT / 会话过滤）：**

1. 若 `tool_uses.or_session_id` 非空 → 与消费日志 `or_session_id` 等值连接（精确）。
2. 否则 → `spend.or_session_id = 'job:' || tool_uses.content_session_id`

**反模式**
- 用裸 UUID 给 `session_id` 打时间戳，却期望与 `content_session_id` 直接等值（前缀不匹配）。
- 打上不同的 `runId`（费用报告 id、随机任务 id），却未将同一字符串写入 `tool_uses.or_session_id`。
- 截断任一侧 — `buildOrStamp` 对超长 id 会抛错；保持该行为。

### 示例

| content_session_id | 正确的 or_session_id |
|---|---|
| `a1b2c3d4-…` | `job:a1b2c3d4-…` |
| `cmem-sess-9f` | `job:cmem-sess-9f` |

MemBench 离线聚合可能使用 `job:membench:<runHash>` — 除非同一字符串存入 `tool_uses.or_session_id`，否则这些**不会**自动连接 `tool_uses`。

---

## 现金归属

- **统计工具使用次数：** `COUNT(DISTINCT tool_use_id)`，按 session/project/tool_name/period 过滤。
- **将 $ 归属到工具行：** 仅当 `or_generation_id` 已设置且匹配实测的消费日志 / Ledger 行。没有它，按 session+tool 的 COUNT 仍然有效；微单位保留在 OR 生成行上。
- 过渡期 Receipt 路径（`or_trace.tool` + `or_trace.tool_calls`）保留，直到 `tool_uses` 成为持久的 COUNT 来源。

---

## Schema 列（已发布）

迁移 **v51**，`src/services/sqlite/tool-uses.ts`（DDL）+ `SessionStore.ensureToolUsesTable()`。
恰为冻结集合 — 可空 `or_generation_id`、可空 `or_session_id`、无费用列：

```sql
CREATE TABLE tool_uses (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  tool_use_id TEXT NOT NULL,
  content_session_id TEXT NOT NULL,
  memory_session_id TEXT,
  session_db_id INTEGER,
  project TEXT NOT NULL,
  platform_source TEXT NOT NULL DEFAULT 'claude',
  tool_name TEXT NOT NULL,
  tool_input TEXT,
  tool_response TEXT,
  cwd TEXT,
  prompt_number INTEGER,
  agent_type TEXT,
  agent_id TEXT,
  observation_id INTEGER,
  or_generation_id TEXT,
  or_session_id TEXT,
  content_hash TEXT,
  created_at TEXT NOT NULL,
  created_at_epoch INTEGER NOT NULL,
  UNIQUE(content_session_id, tool_use_id)
);
```

已建索引：`project`、`memory_session_id`、`content_session_id`、`session_db_id`、
`tool_name`、`created_at_epoch`、`observation_id`、`or_generation_id`。

---

## Receipt 的读取 API（优先用 HTTP，而非直接打开 SQLite 文件）

worker 持有 DB 句柄；直接读文件会与写入方竞争。

### `GET /api/tool-uses` — 索引 / 统计列表

查询参数：`project`、`session`（= `content_session_id`）、`memorySessionId`、
`tool_name`（逗号分隔）、`agentId`、`dateStart`、`dateEnd`（epoch 毫秒）、
`limit`（默认 50，最大 500）、`offset`、`orderBy`（`date_desc` | `date_asc`）。

返回**轻量**结构 — 仅身份、连接键和大小提示。绝不返回 `tool_input` / `tool_response`：

```json
{
  "count": 1,
  "toolUses": [{
    "id": 7,
    "tool_use_id": "toolu_01ABC",
    "tool_name": "Read",
    "project": "claude-mem",
    "content_session_id": "…",
    "memory_session_id": "…",
    "platform_source": "claude",
    "agent_id": null,
    "agent_type": null,
    "observation_id": 99,
    "or_generation_id": "gen-abc",
    "or_session_id": "job:…",
    "prompt_number": 2,
    "created_at": "2026-09-07T00:00:00.000Z",
    "created_at_epoch": 1757203200000,
    "tool_input_bytes": 25,
    "tool_response_bytes": 11
  }]
}
```

这是用于统计的端点：用 `limit`/`offset` 分页并计数。

### `POST /api/tool-uses/batch` — 原始正文，仅按 id

请求体：`{ "ids": [...], "limit"?, "project"?, "contentSessionId"? }`。`ids` 接受
数字 `tool_uses.id` 或不透明 `tool_use_id` 字符串。**必填且非空** — 刻意不提供
分页获取整表原始载荷的方式。Receipt 通常根本不需要此端点。

同时暴露为 `get_tool_uses` MCP 工具（渐进披露第 4 层）。

---

## Receipt 应知的注意事项

**`observation_id` 是批次指针，不是逐调用映射。** 观察器将*一批*工具调用
汇总为一个或多个观测；该批次中的每个工具使用都链接到批次的第一个观测 id，
且链接一旦设置便不再覆盖。用它导航回摘要 — 不要将其视为"此观测花费了此工具调用"。
可靠的逐调用身份仍是 `(content_session_id, tool_use_id)`。

**行可能缺失，这是设计使然。** 以下情况工具调用不会出现在 `tool_uses` 中：
平台未提供 `tool_use_id`；项目被排除；提示被标记为私有；工具在
`CLAUDE_MEM_SKIP_TOOLS` 中；或工具是 claude-mem 自身的披露工具（`memory_*`，以及
claude-mem MCP 的 `search` / `timeline` / `get_observations` / `get_tool_uses`），
跳过它们是为了防止表在每次读取时增长。统计应理解为"claude-mem 观测到的工具调用"，
而非"所有工具调用"。

**载荷上限为 64 KB**，带 `…[truncated: N bytes]` 标记；截断后的值不再是有效 JSON。
`content_hash` 基于截断前的原始载荷计算，因此仍能区分共享前缀的两个超大正文。

**重放是更新，不是重复。** `UNIQUE(content_session_id, tool_use_id)` + upsert；
后续重放可补全首次写入缺失的 `tool_response`。`COUNT(DISTINCT tool_use_id)` 仍是
推荐的统计方式，因为它无论如何都正确。

---

## 联系方式

Tool SQLite 负责 Claude-Mem 表 + HTTP/`get_tool_uses`。Receipt 负责消费日志 + 统计。冻结后如需更改连接键，请联系 Tool SQLite。