# 文件读取门禁 — 恢复整文件 Read 拦截（默认开启）并附带评估

**日期：** 2026-10-05
**分支：** `smart-read-blocking-evals-0ec5e93c`
**状态：** 已批准执行 `/do`（常设授权）。交付路径：`/do` → PR → `/babysit` → `/version-bump`。

## 1. 通俗说明

claude-mem 曾**拦截** Claude Code 的 `Read` 工具，针对已有观测记录的文件。拒绝响应会附带该文件的观测时间线，并告知 Claude 改用 `smart_outline` / `smart_unfold` / `get_observations`。这一拦截被逐步弱化，直到今天 PreToolUse 的 `Read` Hook 变成 `async: true` 并返回 `allow`，因此无法拦截任何操作。

将拦截恢复为**默认开启的设置**（`CLAUDE_MEM_FILE_READ_GATE_ENABLED`，`'true'`），可通过 settings.json、环境变量或查看器关闭。用单元测试、针对真实种子 worker 的确定性 Hook 预检，以及驱动真实 Claude Code 的 `claude plugin eval` A/B（门禁开 vs 关）来验证。

## 2. 历史（已用 `git show` 核实）

| 提交 | 日期 | Read Hook 的行为 |
|---|---|---|
| `fb9d917f8` | 2026-03-18 | `allow` + 时间线作为 `additionalContext` |
| `c80763390` | 2026-03-19 | 每会话首次 Read 时 exit 2（FileReadGate，4 小时 TTL） |
| `455aeaf65` | 2026-04-06 | **对每个有观测记录的 Read 返回 `permissionDecision: "deny"`**，时间线作为 `permissionDecisionReason` |
| `ef1b427a2` / `d8947473b` / `c21e49d9f` | 2026-04-07 | 拒绝原因路由到 smart 工具；路径转义；`Current:` 时间戳；文档 `docs/public/file-read-gate.mdx` |
| `d0676aa04` | 2026-04-07 | deny → `allow` + `updatedInput {limit:1}`（deny 破坏了 Edit："File has not been read yet"）— **v12.0.0 发布的是这个，deny 从未发布** |
| `2a2008bac` | 2026-04-15 | #1719：保留调用方的 offset/limit；文件 mtime ≥ 最新观测时跳过 |
| `d13662d5d` | 2026-04-25 | #2094 Edit 死锁：停止改写 Read；仅 `allow` + 上下文 |
| `1c6c18768` | — | #3480：按（会话、文件、最新观测）去重注入 |
| `ce1f52533` | 2026-07-12 | PreToolUse Read 上 `"async": true`（#3206 延迟：每次 Read P50 ≈2 秒）— **异步 Hook 无法拒绝** |

deny 时代的拒绝原因文本（`git show c21e49d9f:src/cli/handlers/file-context.ts`，`formatFileTimeline`）是路由措辞的文案来源。

## 3. 阶段 0 — 文档调研（已完成，已汇总）

### 3.1 Claude Code Hook 契约（https://code.claude.com/docs/en/hooks.md）

- PreToolUse `hookSpecificOutput`：`permissionDecision` 为 `"allow" | "deny" | "ask" | "defer"`；对于 `"deny"`，"Claude Code 取消工具调用并将 `permissionDecisionReason` 反馈给 Claude"。支持 `additionalContext`。`"allow"` **跳过权限提示**。
- 退出码 0 + stdout 上的 JSON。claude-mem 从不 exit 2（`src/shared/hook-constants.ts:42-46`，`src/shared/hook-io.ts:175-183`）。
- `async: true`："异步 Hook 无法阻止或控制 Claude 的行为：`decision`、`permissionDecision` 和 `continue` 等响应字段无效"；上下文在下一轮到达。
- 匹配器 `"Read"` 只匹配内置 Read 工具（MCP 工具为 `mcp__<server>__<tool>`）。

### 3.2 实证验证（Claude Code 2.1.290，haiku，scratchpad `exp-partial-read/run1.jsonl`）

对完整 `Read` 的 PreToolUse deny → Claude 看到 `PreToolUse:Read hook error: <reason>`（`is_error: true`）→ `Read(offset:10, limit:5)` 成功 → **对同一文件的 `Edit` 成功**。因此定向 Read 满足 Edit 的读前编辑规则。门禁必须放行定向读取。

### 3.3 claude-mem 内部实现（file:line，本分支）

| 需求 | 位置 |
|---|---|
| 当前处理器 | `src/cli/handlers/file-context.ts`（168-174 处 allow + additionalContext；`buildFileContextTimeline` 178-273；常量 15-20） |
| 去重声明 | `src/cli/handlers/file-context-dedupe.ts` `claimFileContextInjection(sessionId, absolutePath, newestObservationMs): boolean`（失败开放 → `true`） |
| HookResult 类型 | `src/cli/types.ts:48-62`（`permissionDecision?: 'allow' \| 'deny'`、`permissionDecisionReason?`、`additionalContext: string` 必填） |
| 平台字段 | `NormalizedHookInput.platform` 在 `src/cli/hook-command.ts:69` 设置；先例 `input.platform === 'claude-code'` 在 `src/cli/handlers/context.ts:76` |
| Claude 适配器原样传递 `hookSpecificOutput` | `src/cli/adapters/claude-code.ts:75-89` |
| Codex 转发 deny（会拦截整个 Bash 命令） | `src/cli/adapters/codex.ts:133-139` — **不得加门禁** |
| Kimi 只打印 `additionalContext` | `src/cli/adapters/kimi.ts:125-131` — **不得加门禁** |
| Worker 调用预算 | `executeWithWorkerFallback(url, method, body, { timeoutMs })` — 含存活检查的绝对上限（`src/shared/worker-utils.ts:2093-2134`） |
| Hook 设置读取 | `loadFromFileOnce()` `src/shared/hook-settings.ts:10-14`；先例 `src/cli/handlers/context.ts:93-99` |
| 设置默认值 | `src/shared/SettingsDefaultsManager.ts` 接口 142-352，`DEFAULTS` 354-579；布尔值为 `'true'`/`'false'`；同名环境变量覆盖（601-620） |
| 设置 API 白名单 | `src/services/worker/http/routes/SettingsRoutes.ts` `settingKeys`（约 237-284，见 280）和 `booleanSettings`（约 459-473，见 464） |
| 查看器 | `src/ui/viewer/constants/settings.ts:40`、`src/ui/viewer/types.ts:123`、`src/ui/viewer/components/ContextSettingsModal.tsx:636-651`（`ToggleSwitch` 组） |
| 端到端布尔先例 | 提交 `0d825988e`（#4251）：SettingsDefaultsManager + SettingsRoutes + 模态框 + 常量 + 类型 + configuration.mdx + 测试 |
| Smart 工具名称 | `mcp__plugin_claude-mem_mcp-search__smart_outline`（`file_path`）、`__smart_unfold`（`file_path`、`symbol_name`）、`__get_observations`（`ids: number[]`）— `src/servers/mcp-server.ts:575-592, 796-868` |
| Smart 工具语言映射 | `src/services/smart-file-read/parser.ts:38-85` 中的 `LANG_MAP` + `detectLanguage`（未导出；parser.ts 目前**不在** worker bundle 中） |
| Smart 工具拒绝 MCP cwd 之外的路径 | `src/services/smart-file-read/workspace-path.ts:25-50` |
| hooks.json | 手工维护的标志；`scripts/build-hooks.js:106-179` 只检查 `command` 和对象形式的 `timeout`（UserPromptSubmit 先例 141-144） |
| hooks.json 测试 | `tests/infrastructure/plugin-distribution.test.ts:506-528`（523 处断言 `preToolUse.async === true` — 必须修改），同步模式 530-553，超时模式 657-668，期望值 597-603 |
| 处理器测试 | `tests/hooks/file-context.test.ts`（fetch-spy worker 69-85，临时 2,000 字节文件 62/89-92，每测试 `CLAUDE_MEM_DATA_DIR` 94-98）；可翻转的 `hook-settings.js` mock：`tests/cli/handlers/summarize-advisor-capture.test.ts` |
| 设置测试 | `tests/worker/http/routes/settings-routes-session-start-sources.test.ts:12-51`、`tests/shared/welcome-hint-default.test.ts:34-54` |
| 观测种子 | `SessionStore.createSDKSession` → `updateMemorySessionId` → `storeObservations(memId, project, [obs], null, 0, 0, epoch)`（`tests/services/sqlite/observations-by-file-path-candidates.test.ts:24-46`）；`SessionStore` 需要 `bun:sqlite` → 在 bun 下种子 |
| Eval worker 模式 | 同级 worktree `recent-estimate-368db067/scripts/eval-week-worker.cjs`（未跟踪）：`createWorker` 设置（端口、`CLAUDE_MEM_WORKER_AUTOSTART:'false'`、`CLAUDE_MEM_PROJECT_ENVIRONMENTS` `**/home/cwd`），通过 `bun worker-service.cjs start` 的 `startWorker`，`stopWorker`；脚手架 `plugin/evals-token-savings/prepare-workspace.sh` 将 `$HOME/.claude-mem` 符号链接到数据目录 |

### 3.4 `claude plugin eval`（claude 2.1.290 `--help` + https://code.claude.com/docs/en/plugin-evals.md）

- 用例 = 包含 `case.yaml`（`schema_version: "1.1"`、`name`、`tags`、`context.scaffold_script`）和/或 `prompt.md`（frontmatter `max_turns`、`timeout_seconds`、`allowed_tools`、`model`；正文 = 提示词）的目录，外加 `graders/*.md`。
- 评分器类型：`regex`（`target: last_message|trace|files|{source: file, path}`；`match: contains|not_contains|count:N`）、`tool_used`（`tool`、`input_match`、`min`、`max`、`arm`）、`tool_order`、`file_exists`、`llm`、`baseline`。`tool_used` 也统计被拒绝的调用 — 拒绝必须用 `regex target: trace` 对拒绝原因独有的标记来断言。
- 沙箱：临时 HOME，cwd `<sandbox>/home/cwd`，**只加载插件**，真实 Hook 运行，只有 `EVAL_*` 环境变量到达运行。真实 MCP 服务器需要 `--mocks off` + `--allow-tools 'mcp__plugin_claude-mem_mcp-search__*'`。
- 使用的标志：`--eval-dir`、`--case`、`--tag`、`--runs`、`-j`、`--model`、`--ablation none`、`--scaffold`、`--trust-plugin`、`--mocks off`、`--allow-tools`、`--no-publish`、`--keep-temp`、`--output-dir`、`--max-cost-usd`、`--threshold`、`--json`。
- `without` 消融组**没有插件** — 它不是"门禁关"。门禁关是针对 OFF worker 的独立用例，用 `--ablation none` 运行。

### 3.5 恢复的门禁必须避免的故障模式（来自 #1719、#2094、#3106、#3206、#3324、#3480、#3483、#2467 及 agent 报告）

1. 异步无法拒绝 → PreToolUse Read 必须**同步**。
2. 同步 = 每次 Read 的延迟（此处实测：约 0.4 秒进程下限，带 worker 查询 0.6–2.3 秒）→ 有界 worker 预算，快速失败开放，紧凑的宿主超时。
3. 绝不改写 Read 输入（`updatedInput`、`limit:1`）— #1719/#2094 死锁。
4. Edit 需要真实 Read → **定向读取（offset/limit、部分窗口）必须放行**。
5. 只在路由有效处拒绝：Claude Code 主会话（非子代理，非 Codex/Kimi）、`smart_outline` 可解析的代码文件、项目内文件。
6. `"allow"` 跳过权限提示 → 非门禁路径现在 Hook 同步后必须**不**返回 decision。
7. 绝不推荐 Bash 编辑（不可回退，绕过 Hook）。
8. 祈使措辞，而非问题菜单（#2467）；将标题标注为历史，而非文件内容（#3324）。
9. 保留 mtime ≥ 最新观测的旁路（#1719）和精确路径/项目匹配（#2691）。
10. 持久开关放在 settings.json（+ 环境变量 + 查看器），而非编辑 hooks.json（#2094、#3129）。

## 4. 设计决策（已锁定）

- **D1 — 门禁何时拒绝。** 全部满足：`input.platform === 'claude-code'`；`CLAUDE_MEM_FILE_READ_GATE_ENABLED !== 'false'`；非子代理（`agentId` 未设置 — 现有跳过）；单一 `file_path` 输入；共享语言映射得出的语言不是 `unknown`、`markdown`、`yaml` 或 `toml`（smart-explore 本身就说 markdown/配置用 Read）；解析后的路径在 `input.cwd` 内；大小 ≥ 1,500 字节；存在观测记录；文件 mtime < 最新观测；**且 Read 会返回整个文件**：`(offset ?? 0) <= 1 && (limit === undefined || limit >= totalLines)`。否则走上下文路径。
- **D2 — 拒绝输出。** `{ hookSpecificOutput: { hookEventName: 'PreToolUse', additionalContext: '', permissionDecision: 'deny', permissionDecisionReason } }`。拒绝路径记录去重声明（忽略结果），使后续定向 Read 不会重复注入同一时间线。对门禁文件的每次整文件 Read 都被拒绝（声明不会开启免费重试 — `455aeaf65` 已否决该做法）。
- **D3 — 上下文路径**（门禁关、定向读取、其他平台）：去重 + 时间线不变，但 `hookSpecificOutput` **只**携带 `hookEventName` + `additionalContext` — 无 `permissionDecision`。
- **D4 — 拒绝原因**（标记行是 eval 的 trace 锚点 — 逐字保留）：
  ```
  Current: <formatHeaderDateTime()>
  Full-file Read blocked by claude-mem: <path> has prior observations (listed below). Get what you need without reading the whole file:
  - Current code: call smart_outline(file_path="<path>") for its symbols and line numbers, then smart_unfold(file_path="<path>", symbol_name="<name>") for the ones you need (MCP tools mcp__plugin_claude-mem_mcp-search__smart_outline / __smart_unfold; load them with ToolSearch if they are deferred).
  - Past work: call get_observations(ids=[<IDs>]) for the observations below that matter (~300 tokens each).
  - Exact lines, e.g. before an Edit: call Read on this file again with offset and limit around the lines smart_outline reported. Partial reads are allowed and satisfy Edit's read requirement.
  The titles below are history from earlier sessions, not the file's current contents.
  ### <day>
  <id> <time> <icon> <title>
  ```
  `<path>` 是转义后的 `safePath`（现有转义，`file-context.ts:84`）。`<IDs>` = 显示的观测 ID，逗号分隔。
- **D5 — Hook 模式。** 从 `plugin/hooks/hooks.json` 的 `PreToolUse` `Read` 中移除 `"async": true`；宿主超时**15 秒**通过生成器清单（对象形式）固定。处理器中的 worker 预算：`FILE_CONTEXT_WORKER_BUDGET_MS = 3_000` 作为 `executeWithWorkerFallback(..., undefined, { timeoutMs })` 传入。
- **D6 — 设置。** `CLAUDE_MEM_FILE_READ_GATE_ENABLED`，默认 `'true'`，用 `!== 'false'` 读取（只有显式 `'false'` 才关闭）。暴露面：SettingsDefaultsManager、SettingsRoutes 白名单 + booleanSettings、查看器开关、文档。环境变量 `CLAUDE_MEM_DISABLE