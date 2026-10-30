# Grok Bot 实时时间线 INDEX

在附加过程中，向每个 Grok Bot 座位的 Memory 文件中实时写入一份持续增长的观察 INDEX，其思路与 CLAUDE.md 文件夹写入器和 SessionStart `/api/context/inject` 相同。

CCS（`ccs/seats/*/TIMELINE.md`）不在产品范围内。不要凭空发明 Grok Bot 引擎 Hook。

## 阶段 0 — 允许使用的 API（已验证）

| API | 来源 | 用途 |
| --- | --- | --- |
| `GET /api/context/inject?projects=&platformSource=` | `SearchRoutes.handleContextInject` | 供 Claude Code / Cursor 使用的规范 INDEX 文本。**不要添加查询参数。** 严格限定在项目范围内。 |
| `generateContextWithStats` / `queryObservationsMulti` | `ContextBuilder.ts`、`ObservationCompiler.ts` | 为显式项目列表获取最新行。项目为空 → 空结果 / 欢迎提示。**不使用全局回退。** |
| SessionStart | `src/cli/handlers/context.ts` | `projects=getProjectContext(cwd).allProjects`（仅 cwd / worktree）。 |
| CLAUDE.md 实时写入 | 存储后由 `ResponseProcessor` 调用 `updateFolderClaudeMdFiles` | 项目 + 文件夹文件路径。同样不使用全局回退。 |
| 紧凑行 | `AgentFormatter.renderAgentTableRow` → `` `${id} ${time} ${icon} ${title}` `` | 保留 ID 供 `get_observations` 使用。 |
| 宿主 Memory 事实语法 | `tests/grok-bot-session-inject.test.ts` | `- (YYYY-MM-DD) …`、`[episode]` 层级、500 字符截断、仅 `zz-claude-mem-inject.md`。 |
| 座位映射 | `transcript-watch.json` + `discoverGrokBotAgentDataRoot` | `cmem_work_<slug>` 日记通常内容单薄。 |
| 感知推送器 | `GrokBotAwarenessPusher` | 将线索写入 `YYYY-MM.md`。**不是** INDEX。保持原样。 |

### 产品规则（内容单薄的日记）

`/api/context/inject?projects=<seat>` 和 CLAUDE.md 重新生成仍限定在项目范围内。这对 Claude Code 是正确的。

Grok Bot 座位是命名日记（`cmem_work_prioritizer`），运行在能看到整个宿主的 worker 上。如果座位项目内容单薄，**将最新的座位行与最新的宿主行（不加项目过滤）合并，按最新优先，在 `WINDOW` 处滑出**。仅在 Grok Bot 写入器中执行此操作。不要改变 inject/hook 的语义。

### 反模式

- 不要将 `ccs/seats/*/TIMELINE.md` 作为事实来源来要求或写入。
- 不要添加 Grok Bot `hooks` / 引擎 SessionStart。
- 不要为 `/api/context/inject` 添加新的查询参数。
- 不要写入 `profile.md` 或宿主拥有的 `YYYY-MM.md`。
- 不要将阶段 1 的 CCS 守护进程作为产品启动。

## 阶段 1 — 写入器 + 设置

Worker 原生的 `GrokBotIndexWriter`（类似 CLAUDE.md / 感知推送器）：

- 观察数据存储后，防抖刷新**所有**已映射的座位（宿主信息流会改变每个座位）。
- 在 transcript watcher 之后，worker 启动时触发一次。
- 直接写入 `agents/<uuid>/memory/log/zz-claude-mem-inject.md`。
- 设置：`CLAUDE_MEM_GROK_BOT_INJECT_ENABLED`（默认 `true`，无座位时为空操作）、`AGENT_IDS` 默认 `*`、`TIER=episode`、`WINDOW=80`、`FALLBACK=house`、`PLATFORM_SOURCE` 为空。

## 阶段 2 — 测试

纯合并/格式化/路径测试 + 写入增长测试。不要破坏 Claude/Cursor hook 测试。

## 阶段 3 — 文档

`docs/public/grok-bot/index.mdx`、`configuration.mdx`、Grok Bot 安装/mem-search 技能。说明如何启用/禁用，以及如何验证座位文件在增长。

## 阶段 4 — 验证 / 看护 / 版本号升级

CI 通过后，进行 MINOR 版本升级（`13.24.23` → `13.25.0`）。npm 发布留给 Prioritizer。