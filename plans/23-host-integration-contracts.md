<!-- 镜像自 GitHub Issue #3611。Issue 是公开跟踪器，本文档是设计文档。
     若两者不一致，设计以文档为准，状态以 Issue 为准（oh-my-issues 健康检查）。 -->

> **跟踪器：** https://github.com/thedotmack/claude-mem/issues/3611

# [plan-23] 宿主集成契约 — 每个非 Claude Code 适配器及 MCP/HTTP 工具接口，均通过针对宿主真实 Schema 的契约测试验证

> **跟踪器：** #3611 · **设计文档：** `plans/23-host-integration-contracts.md` · **系列：** plan-14 #3602 · plan-15 #3603 · plan-16 #3604 · plan-17 #3605 · plan-18 #3606 · plan-19 #3607 · plan-20 #3608 · plan-21 #3609 · plan-22 #3610 · plan-23 #3611

## 缺陷

每个宿主集成（OpenCode 插件、Codex Hook、Antigravity CLI/IDE、Hook-CLI 平台、transcript 监视器，以及 `mcp-search` MCP 服务器 / worker HTTP 接口）都是手写适配器，唯一的测试是"源文件能否导入"。没有任何一个针对宿主的真实加载器/Hook/工具 Schema 做过校验，而它们依赖的共享管道（端口解析、生成器启动、查看器来源过滤）各适配器各自分叉：

- **OpenCode：** 插件绑定 `chat.message`（上游已移除；现为 `message.updated`/`message.part.updated`），发送捕获 POST 时不等待，从插件入口导出两个 `as const` 数组，被 opencode 加载器拒绝（`Plugin export is not a function`）——13.4.0 → 13.4.1 回归，13.14.0 仍存在——从环境变量/默认值解析 worker 端口而非 `settings.json`，安装器刷新的是过期的 12.4.7 npx 缓存插件文件。OpenCode 不是查看器来源。
- **Codex：** `codex-hooks.json` 输出 `suppressOutput`（Codex 拒绝），且 POSIX shell 命令字符串没有 `commandWindows`，导致 Windows Codex 应用通过 PowerShell 执行（`ParserError`）。
- **Antigravity / transcript 监视器：** `--ide antigravity` 只接入了 `agy` CLI Hook，GUI IDE 从不触发它们；transcript 监视器的 `Bun.Glob` 缺少 `dot: true`，IDE 的 `.system_generated/logs/transcript_full.jsonl` 永远无法被发现；Windows 上 agy 路径不记录任何内容；安装器依赖一个它并不安装的市场文件；自定义监视 Schema 无文档。
- **Hook-CLI 平台：** `worker-service.cjs hook <platform> observation` 返回 `{"status":"queued"}`，但对于通过 `/api/sessions/init` 在 Claude Code 之外注册的会话，生成器从不启动。
- **MCP / HTTP 接口：** 工具定义没有 `annotations.readOnlyHint`，导致 plan 模式每个会话都重新提示；server-beta 的 `observation_add` 在协议层与处理器之间丢失 `content`；`claude mcp list` 渲染条目乱码；`skillOverrides: off` 不隐藏内置技能描述，且没有插件级退出 smart-* 工具的选项；CORS 允许列表回归为仅 localhost 前缀，导致无法从局域网查看器保存设置（曾是 #1029）；worker 启动失败时 stdio MCP 服务器约 59 秒后关闭（Windows）。

修复方案是为每个宿主建立契约——签入宿主加载器/Hook/工具 Schema 的 fixture，适配器在 CI 中针对它生成或校验，并用共享解析器（端口、项目、生成器启动）替代分叉。

## 子项

- #2832 — OpenCode 插件在 opencode 1.16.2 上加载但不捕获任何内容（失效的 chat.message Hook + 未等待的 POST）
- #2854 — opencode 插件加载错误："Plugin export is not a function"
- #2871 — Codex PostToolUse Hook 输出当前 Codex 不支持的字段（suppressOutput）
- #2916 — observation_add：调用方传入了 content 参数，但 server-beta 处为 undefined
- #2968 — 跟踪：OpenCode 插件不捕获任何内容——失效的 Hook 名、未等待的 POST、无效的导出结构
- #2975 — 跟踪：Codex PostToolUse Hook 输出不支持的 suppressOutput 字段——每次工具调用 Hook 均被拒绝
- #2986 — OpenCode 插件加载但自动捕获未初始化（过时的 OpenCode Hook：chat.message）
- #3075 — Windows 上的 Codex 插件 Hook 需要 commandWindows 覆盖
- #3101 — Windows Codex 应用用 PowerShell 执行 codex-hooks.json 的 POSIX 命令，导致 ParserError
- #3239 — Antigravity GUI IDE：无自动 transcript 捕获（仅接入了 agy CLI Hook）
- #3328 — OpenCode 插件在 v13.11.0 中加载失败："Plugin export is not a function"（非函数 const 导出）
- #3330 — OpenCode 插件在 v13.11.0 中加载失败："Plugin export is not a function" + 包装 fetch 的 .text() 返回对象（#3328 的重复）
- #3334 — Windows 上的 Antigravity CLI 集成不记录任何会话/观察；重装失败（安装部分 → plan-16）
- #3365 — OpenCode 插件忽略 settings.json 中的 CLAUDE_MEM_WORKER_PORT，捕获中断
- #3471 — 检查 Claude mcp list 时一团糟
- #3483 — plan 模式每个会话都重新提示 mcp-search 工具（无 readOnlyHint 注解）
- #3504 — OpenCode 未在 Web 查看器来源过滤中显示为独立来源
- #3512 — transcript-watcher：Bun.Glob 需要 dot: true——点目录下的 transcript 永远无法被发现
- #3526 — 回归：从非 localhost 来源保存设置时 500 "CORS not allowed"（13.14.0，曾是 #1029）
- #3578 — skillOverrides "off" 不隐藏会话中的内置技能描述
- #3585 — 来自非 Claude Code 平台的 Hook CLI 观察已入队但从不生成

待评估/变基的相关 PR：#2855、#2985（OpenCode 事件 + await）、#2953（Codex suppressOutput）、#3208（OpenCode 端口）、#3527（readOnlyHint）、#3070（远程 MCP）、#2699/#2679（Codex Windows）、#1214（原始 CORS 修复）。净新增宿主（Hermes #2825、Goose #3329、omp #3355、HTTP/SSE MCP 传输 #3086）在路线图主项 #2785 上跟踪，本契约建立后即可低成本接入。

## 修复顺序

1. **宿主 Schema fixture + 契约测试。** 为我们声称支持的每个宿主版本签入加载器/Hook/工具 Schema（`fixtures/hosts/opencode-<ver>.json`、`codex-hooks-<ver>.json`、`antigravity-<ver>.json`、`claude-code-mcp-annotations.json`）；CI 导入*构建后*的 `dist/opencode-plugin/index.js`，断言每个导出都是函数或 `{server: fn}`，注册的 Hook 名 ⊆ 宿主事件列表，且 Codex Hook JSON 只含宿主接受的键，每个条目都有 `commandWindows`。
2. **适配器共享解析器。** OpenCode 插件、Codex 适配器、Antigravity/transcript 监视器和 Hook-CLI 都调用同一个 `resolveWorkerEndpoint()`（settings.json → 环境变量 → 默认值，感知 `CLAUDE_MEM_DATA_DIR`）、同一个 `resolveProjectIdentity()`（plan-20）和同一个 `ensureGeneratorRunning(sessionDbId)`，使任何宿主初始化的会话都能生成；每个捕获 POST 都带有限超时等待。
3. **Antigravity IDE 与自定义 transcript。** `Bun.Glob(..., {dot: true})`；内置 Antigravity IDE Schema；自定义 `transcript-watch.json` Schema 附带文档和校验命令；`--ide antigravity` 如实报告捕获的是哪个界面（CLI 还是 IDE）；安装器在升级时刷新宿主插件文件并打印已安装版本。
4. **MCP/HTTP 接口。** 所有只读工具加 `readOnlyHint: true`，注解在 `tools/list` 中转发；server-beta 写工具透传 `content`（在 MCP 协议路径上做集成测试，而非处理器）；插件级 `CLAUDE_MEM_MCP_TOOLS` 允许列表用于裁剪 `smart_*`；技能描述遵循 `skillOverrides`/`CLAUDE_MEM_SKILLS_DISABLED`；CORS 允许同宿主来源和 `CLAUDE_MEM_ALLOWED_ORIGINS` 列表，拒绝时返回 JSON 错误体；`mcp-server.cjs` 报告干净的 name/title/description，worker 无法连接时带原因及时退出（worker 启动由 plan-15 负责）。
5. **查看器。** `platform_source` 端到端传递，来源过滤器由去重值生成（OpenCode、Codex、Antigravity、Cursor、hook-CLI）。

## 测试矩阵

| 宿主 | 版本 | 场景 | 要求行为 |
|---|---|---|---|
| OpenCode | 1.16, 1.17, 1.18 | 加载 + 一次工具调用 + 一条消息 | 插件加载；`/api/sessions/init|observations|summarize` 在 settings.json 端口上收到；观察以 `platform_source=opencode` 存储 |
| Codex CLI / Codex App | 当前版本，macOS + Windows | PostToolUse | Hook 被接受（无不支持的键）；Windows 执行 `commandWindows`；观察已存储 |
| Antigravity CLI + IDE | 当前版本 | 会话 | CLI：Hook 捕获；IDE：transcript 监视器发现点目录 transcript 并摄取 |
| hook-CLI（`hook <platform>`） | 任意 | init + observation | 生成器启动；观察行在 ≤ 60 秒内存在 |
| Claude Code plan 模式 | 当前版本 | `search`、`timeline`、`get_observations`、`smart_*` | 无权限提示（readOnlyHint） |
| server-beta MCP | 当前版本 | 通过 stdio 的 `observation_add` | 行已写入 |
| 从 `http://<lan-host>:<port>` 访问查看器 | 当前版本 | 保存设置 | 200；CORS 允许同宿主；显式拒绝返回 JSON 403 |
| Codex | 当前版本 | `enabled_tools`/`CLAUDE_MEM_MCP_TOOLS` | 仅通告列出的工具 |

矩阵在 CI 中运行（`tests/integrations/*-contract.test.ts` 针对签入的宿主 fixture，外加一个安装真实 OpenCode CLI 的 Docker 任务）。回归必须在用户提交前让 CI 失败。

## 范围之外

Claude Code 自身的 Hook 包装器（plan-17）。安装器具体落地哪个插件副本（plan-16）。MCP 服务器背后的 worker 启动/存活（plan-15）。新宿主集成（路线图 #2785）。


---

## 轮次：2026-09-12 oh-my-issues

- **#4057** — Antigravity CLI（`agy` 1.2.1）在 Windows 上记录零观察，源于反汇编 `agy.exe` 与适配器对比发现的四个独立未校验的不匹配：(1) 安装器将 Hook 写入 `~/.gemini/settings.json`，但 `agy` 读取 `hooks.json`（`.agents/hooks.json` 或插件的 `hooks.json`）；(2) 它注册的是旧版 Gemini 事件名——二进制中 `BeforeTool`、`AfterTool`、`BeforeAgent` 出现 **0 次**；Antigravity 派发的是 `PreToolUse`、`PostToolUse`、`PreInvocation`、`PostInvocation`、`Stop`；(3) `agy` 使用 camelCase protojson（`conversationId`、`workspacePaths`、`transcriptPath`、`toolCall`、`stepIdx`），期望返回 `{"decision":"allow"}` / `{"injectSteps":[...]}` / `{}`，而 claude-mem 返回 `{"continue":true,"systemMessage":"..."}`；(4) transcript 使用 `type: PLANNER_RESPONSE` / `USER_INPUT` 和顶层 `content`，而非 `type: assistant` 和 `message.content`。

  四个不匹配无一被任何测试捕获，这正是本计划存在的理由——适配器目前唯一的测试是"源文件能否导入"。交付路径：在 `AntigravityCliHooksInstaller.ts`、`src/cli/adapters/antigravity-cli.ts`、Antigravity transcript 分支和 `tests/antigravity-cli-compat.test.ts` 中，将窄范围重新托管到当前 `main`——明确不复用 #3742（混入 OpenCode）或 #3397（陈旧、大型捆绑重建）。RCA 后续部分留在此处：签入实时 `agy` Schema fixture 并在 CI 中校验，否则会在 `agy` 1.3 上复发。见 PR #4058。