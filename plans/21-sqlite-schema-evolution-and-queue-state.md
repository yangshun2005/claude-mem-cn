<!-- GitHub Issue #3609 的镜像。Issue 是公开跟踪器；本文档是设计文档。
     若两者出现偏差，设计以本文档为准，状态以 Issue 为准（oh-my-issues 健康检查）。 -->

> **跟踪器：** https://github.com/thedotmack/claude-mem/issues/3609

# [plan-21] SQLite Schema 演进与队列状态完整性 —— 单一 DDL 路径，由内省守护、不可变会话键，以及真正可转换的队列/会话状态

> **跟踪器：** #3609 · **设计文档：** `plans/21-sqlite-schema-evolution-and-queue-state.md` · **系列：** plan-14 #3602 · plan-15 #3603 · plan-16 #3604 · plan-17 #3605 · plan-18 #3606 · plan-19 #3607 · plan-20 #3608 · plan-21 #3609 · plan-22 #3610 · plan-23 #3611

## 缺陷

存储层通过两条相互竞争的 DDL 路径演进，由版本标志而非真实 Schema 守护，且其运行时状态机存在死转换：

- `initializeSchema()` 在迁移链之前运行，其中包含 `CREATE INDEX … ON sdk_sessions(platform_source)`；在 v23 数据库上 `CREATE TABLE IF NOT EXISTS` 是空操作，该索引抛出 `no such column: platform_source`，迁移 24–35 永不执行，worker 永远停留在"健康但未就绪"。`removeSessionSummariesUniqueConstraint()`（v7）由每次构造时重跑的实时索引探测守护；旧版本包的宽松探测会误判 v41 的 `ux_session_summaries_origin` 索引，并**用冻结的 14 列列表重建 `session_summaries`**，丢弃 `discovery_tokens` 和 v21 的 `ON UPDATE CASCADE` 外键；`ensureDiscoveryTokensColumn()` 由 v11 账本标记守护，因此永不自我修复；两条重建路径在同一 Schema 版本下产出不同 DDL。没有任何东西记录哪个二进制版本最后写入数据库，因此旧 worker（plan-16 的陈旧目录启动）可以破坏更新的 Schema。
- 子表（`observations`、`session_summaries`）的外键指向**可变**的 `sdk_sessions(memory_session_id)`；`startSession` 在活跃子行集下重写它（先 `updateMemorySessionId(id, null)` 再写入新 id）→ 10k–27k 条 `foreign_key_check` 违规，缺少级联处每 5–10 秒出现 `FOREIGN KEY constraint failed` 生成器循环。
- 队列/会话状态无法自愈：`PendingMessageStore.retryMessage()` 从未被调用；生成器失败将整个会话标记为 `failed`，绕过 `maxRetries`；`completeByDbId()` 从不写入 `status='completed'`（46 活跃 / 1546 失败 / 0 完成）；`markAllSessionMessagesAbandoned` 会放弃 `status='processing'`，导致会话完成时正在处理的摘要被杀（#1519 的回归）；`session_summaries.files_read/files_edited` 从未填充；`relevance_count` 从未递增。
- 增长无界且不可回收：`user_prompts.prompt_text` 逐字存储完整的 UserPromptSubmit 载荷（平均约 990 KB），并在 `sdk_sessions.user_prompt` 中再存一份，FTS5 触发器又使其翻倍；175k 条 `pending_messages` 行永久 `failed`；无保留策略；`auto_vacuum=0` 且无 VACUUM，批量删除后留下 131 GB 文件、90% 空闲列表；`busy_timeout=0` 使任何争用都变成 500 和一条丢失的 observation；v12.4.3 清理门误读 Bun 的 int32 截断 `statfs`，永不运行。`settings.json` 写入非原子，解析失败时重置为 `{}`（13.10.2 已修复——验证并关闭）。文档/技能引用了不存在的列。

修复方案是单一 Schema 权威：迁移在任何查询*之前*运行，每个迁移由 `PRAGMA table_info`/`index_list` 内省守护，每张表有单一规范 `CREATE TABLE` 并据此生成重建，写入者版本戳并拒绝降级，不可变外键键，以及带重试/完成转换及保留和 vacuum 策略的显式状态机。

## 子项

- #2793 — 数据库无界增长：prompt_text 逐字存储（×2），无保留策略，无 VACUUM
- #2821 — Worker 在锁争用下丢弃 observation（busy_timeout=0 → 500 "database is locked"）
- #2969 — 跟踪：SQLite 数据完整性——无界 prompt 膨胀与 busy_timeout=0 丢弃写入
- #3080 — 并发 Hook 下 ~/.claude-mem/settings.json 被截断为一个键（13.10.2 已验证修复——验证后关闭）
- #3094 — 从旧数据库升级到 v13.9.1 后迁移 v24-v35 未应用 → "no such column: platform_source"，worker "健康但未就绪"
- #3118 — 直接 Schema v23 → v35 失败：worker 后台初始化抛出 `no such column: platform_source`
- #3164 — sdk_sessions.status 永不达到 'completed'——completeByDbId() 只拆除内存状态
- #3332 — timeline-report 技能：示例 SQL 引用了不存在的 observations.source_tool 列
- #3344 — 失败的 pending_messages 在结构上永不自愈：retryMessage() 是完整实现、完全死掉的代码
- #3419 — 会话完成丢弃进行中的 summarize：markAllSessionMessagesAbandoned 放弃 status=processing（#1519 的回归）
- #3446 — 从旧插件版本目录启动的 worker 静默丢弃 session_summaries.discovery_tokens——所有会话摘要永久失败
- #3502 — PRAGMA foreign_key_check 报告 10k-27k 条孤儿 observation——memory_session_id 父键在活跃子行集下被重写
- #3503 — 生成器失败循环：session_summaries 缺少 ON UPDATE CASCADE 使每次父键重写都抛错（Schema 49）
- #3517 — session_summaries.files_read 和 files_edited 从未写入——5,059 行中 0 行
- #3551 — v12.4.3 清理在 linux-x64 上永久失败并报 "Insufficient disk"（Bun statfsSync int32 截断）

需评估/变基的相关 PR：#2904（维护/VACUUM）、#2849（busy_timeout）、#2862/#2860（prompt 上限，已合入）、#3378（`repairOrphanedSessionParents`）、#1519/#1624（先前修复）、#2770。

## 修复顺序

1. **迁移优先，基于内省。** `SessionStore` 构造函数：打开 → `PRAGMA busy_timeout=5000; synchronous=NORMAL; foreign_keys=ON` → 运行迁移链 → *然后* `initializeSchema()`（索引仅在其列可证明存在后创建）。每个迁移守护都基于 `PRAGMA table_info/index_list/foreign_key_list`；账本仅供参考。启动时协调器将实时 DDL 与规范 `CREATE TABLE` 字符串（单一来源，重建由其生成）比对并修复漂移（缺失列、缺失 `ON UPDATE CASCADE`），记录所做操作。
2. **写入者版本戳。** `PRAGMA user_version`/`meta.writer_version` = 插件 semver；比该戳更旧的二进制拒绝运行迁移或重建，并以类型化 `schema_newer_than_binary` 原因退出（plan-16 防止此情况发生；本方案使其在发生时非破坏性）。
3. **不可变会话键。** 子表外键指向 `sdk_sessions(id)`；`memory_session_id` 变为普通可变列；移除 `startSession` 中的置空写入；一次性修复按 `session_db_id` 重新挂接孤儿行。
4. **带真实转换的状态机。** `pending_messages`：`pending → processing → (done | pending[retry_count+1] | failed[retry_count ≥ maxRetries])`，`retryMessage()` 接入每条失败路径，定时退避遍历重新处理不在活跃映射中的会话的 `pending` 行，`markPendingMessagesAbandoned` 只触碰 `pending`，会话终结有界等待进行中的处理；`sdk_sessions`：`completeByDbId()` 写入 `completed`，陈旧扫描写入两个 `completed_at` 字段；`session_summaries` 从会话的 observation 文件列表写入 `files_read/files_edited`；`relevance_count` 在召回时递增。
5. **保留与回收。** `prompt_text` 设上限且只存一次（已剥离注入上下文）；`pending_messages` 失败行 N 天后清除；新数据库设置 `auto_vacuum=INCREMENTAL`，一次性后台 `VACUUM` 由正确的空闲空间检查守护（`statfsSync(path,{bigint:true})`，负值防护）；observer 转录保留（位置由 plan-19 负责）；按项目统计端点使增长可见。
6. **文档由 Schema 生成。** 枚举列的技能/文档由 `PRAGMA table_info` 生成（或经 CI 检查）。

## 测试矩阵

| 起始数据库 | 升级到 | 必需行为 |
|---|---|---|
| 全新 | HEAD | 所有迁移执行，`foreign_key_check` 为空，`integrity_check` 通过 |
| v7、v11、v21、v23、v32、v35、v41、v49 的真实夹具（从用户捕获） | HEAD | worker 在 ≤ 30 秒内到达 `/api/readiness`；无 `no such column`；`session_summaries` 具有 `discovery_tokens` + `ON UPDATE CASCADE`；账本连续或仅供参考 |
| HEAD | 旧二进制（13.11.0） | 拒绝迁移/重建；以 `schema_newer_than_binary` 退出；数据库不变 |
| v49 含 10k 外键孤儿 | HEAD | 孤儿重新挂接；`foreign_key_check` 为空 |
| 任意 | 生成器失败 ×5 | 行保持 `pending`，`retry_count` 1..3 后 `failed`；下一轮重试 |
| 任意 | 会话完成时 summarize 进行中 | 摘要写入；无 `Drained N orphaned pending messages` |
| 任意 | 20 个并发写入者 + `integrity_check` | 零 `database is locked` 500 |
| 100 GB 夹具（合成） | 维护 | 回收运行，大小下降，无锁风暴 |
| linux NFS 类 `bavail > 2^31` | 清理门 | 空闲空间计算为正；清理运行 |

矩阵位于 CI 中（`tests/sqlite/migrations-upgrade-matrix.test.ts`，基于检入的夹具数据库）。回归必须在用户提交前使 CI 失败。

## 范围之外

哪个二进制可以打开数据库（plan-16）。解析失败的 observer 输出（plan-18）。项目键派生（plan-20；本方案只存储别名）。Chroma/向量存储（plan-22）。


---

## 轮次：2026-09-12 oh-my-issues

- **#4062** — SDK OAuth 中断约 10.5 小时期间，每次生成器重启都出现 `NOT NULL constraint failed: session_summaries.memory_session_id`，并伴随匹配的写入缺口。这是本方案可变外键缺陷的现场确认，已在讨论串中诊断：INSERT 收到的并非 null。子列是 `NOT NULL`，其外键带 `ON DELETE CASCADE ON UPDATE CASCADE`（`SessionStore.ts:1015`），因此**新 SDK 生成前的 id 重置**向 `sdk_sessions.memory_session_id` 写入 NULL，级联将其传播到子表——这就是为什么存储调用前的 `!e.memorySessionId` 守卫从未起作用，以及为什么它只在子行存在时触发（"带积压恢复"）。

  **已修复**，由 `cd8258f` / PR #3629，发布于 **13.24.16**：重置现在只清除内存中的 id（推理见 `src/services/worker/ClaudeProvider.ts:196-205`），唯一剩余的写入者是 `OpenAICompatibleProvider.ts:134`，使用合成的非空 id，`tests/fk-constraint-fix.test.ts` 保持 NULL 写入抛错作为绊线。记录于此，因为该绊线*就是*"不可变外键键"的矩阵单元。13.24.16 之后的约束错误是另一条路径——请附上 traceback、`sessionDbId` 和 `queueDepth` 重新提交。