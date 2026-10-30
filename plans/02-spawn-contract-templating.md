# Spawn-Contract 模板化歧义 — 分阶段修复计划

**根因：** `${CLAUDE_PLUGIN_ROOT}` 及类似占位符在 spawn 边界上的处理方式不一致。部分宿主在 hook/MCP spawn 时替换它们，部分由 shell 展开，部分两者都不做（原始 `${CLAUDE_PLUGIN_ROOT}` 直接到达二进制）。结果：MCP 服务器启动失败；hook 命令解析到错误路径；跨 IDE 行为在 12-IDE 矩阵中不一致。

**本次修复的净效果：** 每类集成有单一、文档化的规范解析规则；集中式模板生成器产出 shell 防御性前导和绝对路径固化；构建时护栏防止漂移；文档与规范规则对齐；验证矩阵覆盖每个（IDE × hook 事件 × 平台）组合。

**不在范围内：**
- Codex 市场缓存版本不匹配（由 `plans/2026-05-06-codex-plugin-version-mismatch.md` 覆盖）。
- `bun-runner.js` 的 stdin 处理任何重构（issue #2188 领域 — 独立关注点）。
- Pro 功能端点或 worker 端口解析（使用 `CLAUDE_MEM_WORKER_PORT`，而非 `CLAUDE_PLUGIN_ROOT`；正交）。

---

## Phase 0 — 文档发现

以下事实来自 2026-05-07 对工作树的直接文件读取（grep + Read）。下面每个实现阶段按行号引用它们；不要重新推导。**置信度：** 代码为高；上游 IDE 宿主文档为中（Phase 0 必须在新上下文中通过 web fetch 验证这些）。

### 0.1 占位符调用点 — 已确认目录

| # | 文件 | 行号 | 替换层 | 备注 |
|---|---|---|---|---|
| 1 | `plugin/hooks/hooks.json` | 11, 24, 30, 42, 55, 68, 80（每个 hook 命令） | Claude Code 注入环境变量 → bash 展开 `${CLAUDE_PLUGIN_ROOT:-${PLUGIN_ROOT:-}}` | 6 个 hook 事件。显式设置 `shell: bash`。 |
| 2 | `plugin/hooks/codex-hooks.json` | 10, 15, 20, 32, 44, 56, 67（每个 hook 命令） | Codex *应*注入环境变量 → sh 展开。额外增加 PATH 解析前导。 | 5 个 hook 事件（无 `shell` 字段；假定为 sh）。 |
| 3 | `.mcp.json` | 8（单个 mcp-search 命令） | `sh -c "..."` 参数展开 `${VAR:-default}`。构建断言与 #4 字节一致。 | 包含 `$PWD/plugin`、`$PWD` 和 `~/.codex/plugins/cache/...` 回退。 |
| 4 | `plugin/.mcp.json` | 8 | 同 #3。 | 打包在插件内；#3 的副本。 |
| 5 | `plugin/scripts/version-check.js` | 7–17 | 读取 `process.env.CLAUDE_PLUGIN_ROOT`，然后回退到 `dirname(fileURLToPath(import.meta.url))/..`。 | 运行时解析层。 |
| 6 | `plugin/scripts/bun-runner.js` | 11（`RESOLVED_PLUGIN_ROOT`）、13–21（`fixBrokenScriptPath`）、168（诊断输出） | 读取 `process.env.CLAUDE_PLUGIN_ROOT`，回退到脚本 dirname。`fixBrokenScriptPath` 是权宜之计：当参数以 `/scripts/` 开头（即原始未替换的 `${CLAUDE_PLUGIN_ROOT}/scripts/X.cjs` 以 `/scripts/X.cjs` 形式到达）时，它前置 `RESOLVED_PLUGIN_ROOT`。 | 运行时解析层。 |
| 7 | `src/services/integrations/CodexCliInstaller.ts` | 60–78（`resolvePluginMarketplaceRoot`）、66–67（查询的环境变量） | 读取 `process.env.CLAUDE_PLUGIN_ROOT`，然后 `process.env.PLUGIN_ROOT`，然后 `process.cwd()`，然后脚本 dirname。 | 仅安装时。 |
| 8 | `src/services/integrations/CursorHooksInstaller.ts` | 84–110（`findMcpServerPath`、`findWorkerServicePath`）、230–232（`makeHookCommand`） | 无 — 从 `MARKETPLACE_ROOT` 或 `process.cwd()` 固化绝对路径。 | 纯绝对路径固化。 |
| 9 | `src/services/integrations/GeminiCliHooksInstaller.ts` | 46–60（`buildHookCommand`） | 无 — 固化绝对 `bunPath` 和 `workerServicePath`。 | 纯绝对路径固化。 |
| 10 | `src/services/integrations/WindsurfHooksInstaller.ts` | （使用 CursorHooksInstaller 的 `findBunPath`、`findWorkerServicePath`） | 无 — 固化绝对路径。 | 纯绝对路径固化。 |
| 11 | `src/services/integrations/McpIntegrations.ts` | 16–21（`buildMcpServerEntry`）、175–192（Goose YAML 构建器） | 无 — 固化 `process.execPath`（Node）+ 绝对 `mcpServerPath`。 | 纯绝对路径固化。目标：copilot-cli、antigravity、goose、roo-code、warp。 |
| 12 | `src/services/integrations/OpenCodeInstaller.ts` | 29–46（`findBuiltPluginPath`） | 无 — 将 `dist/opencode-plugin/index.js` 复制到 `~/.config/opencode/plugins/claude-mem.js`。 | OpenCode 在自己的沙箱中运行 JS；无 shell。 |
| 13 | `src/integrations/opencode-plugin/index.ts` | 74–80（`resolveWorkerPort`） | 使用 `CLAUDE_MEM_WORKER_PORT` 环境变量（与 plugin-root 范围正交）。 | 无 plugin-root 模板化。 |
| 14 | `openclaw/install.sh`（1653 行） | grep 对 `CLAUDE_PLUGIN_ROOT` 或 `PLUGIN_ROOT` 返回 0 命中。使用 `${HOME}`、`${COLOR_*}` 等。 | 不适用 — OpenClaw 通过 `configSchema`（`workerPort`、`workerHost`）配置；无 plugin-root 模板化。 | 不在范围内，但为完整性记录。 |
| 15 | `.claude-plugin/marketplace.json`、`.claude-plugin/plugin.json`、`.codex-plugin/plugin.json`、`plugin/.claude-plugin/plugin.json`、`plugin/.codex-plugin/plugin.json`、`.agents/plugins/marketplace.json` | manifest 字段 | 无 — 仅相对路径（`./plugin`、`./.mcp.json`、`./hooks/codex-hooks.json`）。 | 由宿主市场机制解析。 |
| 16 | `docs/public/hooks-architecture.mdx` 第 100、176、223、283、337、604、754 行 | 代码示例 | 文档 — 当前教用户原始 `${CLAUDE_PLUGIN_ROOT}/scripts/...` 语法。 | 这些示例驱动第三方复制粘贴；必须与 Phase 1 选定的规范规则对齐。 |
| 17 | `docs/public/configuration.mdx:142`、`docs/public/development.mdx:257`、`docs/public/architecture/hooks.mdx:196,204,208,215,223,230,237` | 代码示例 | 文档 — 与 #16 相同模式。 | 同上。 |

### 0.2 Spawn-contract 矩阵 — 我们拥有的站点已确认

| 站点 | 由谁 spawn | `${CLAUDE_PLUGIN_ROOT}` 由谁替换 | Shell 语义 |
|---|---|---|---|
| `plugin/hooks/hooks.json` | Claude Code hook 运行器 | Claude Code 注入环境变量；bash 展开 `${VAR:-default}` | bash（`shell: bash`） |
| `plugin/hooks/codex-hooks.json` | Codex CLI hook 运行器 | Codex *应*注入环境变量；sh 展开 | sh（无 `shell` 字段） |
| `.mcp.json` / `plugin/.mcp.json` | Claude Code / Codex MCP 加载器 | `sh -c "..."` 展开 `${VAR:-default}` | `sh -c` 带 args[] |
| Cursor `hooks.json` / `mcp.json` | Cursor | 无 — 安装器固化绝对路径 | 原生 exec |
| Gemini `settings.json` hooks | Gemini CLI | 无 — 安装器固化绝对路径 | 原生 exec |
| Windsurf `hooks.json` | Windsurf | 无 — 安装器固化绝对路径 | 原生 exec |
| Copilot/Antigravity/Goose/Roo/Warp `mcp.json` | 各 IDE 的 MCP 加载器 | 无 — 安装器固化绝对路径 | 原生 exec |
| OpenCode 插件 | OpenCode 运行时 | 不适用 — JS 插件，无 shell | JS |
| OpenClaw 插件 | OpenClaw 网关 | 不适用 — 通过 `configSchema` 设置 | JS |

### 0.3 覆盖此范围的现有测试

`tests/infrastructure/plugin-distribution.test.ts`：
- 第 110–114 行：每个 hook 命令必须包含 `CLAUDE_PLUGIN_ROOT`。
- 第 116–122 行：每个 hook 命令必须包含 `$_C/plugins/marketplaces/thedotmack/plugin` 回退（issue #1215）。
- 第 124–132 行：缓存路径必须在市场回退之前尝试（issue #1533）。
- 第 84–99 行：MCP 启动器包含 `.codex/plugins/cache/claude-mem-local/claude-mem` 和 `plugins/cache/thedotmack/claude-mem` 回退；根和打包启动器保持同步。
- 第 135–177 行：`.mcp.json`、codex hooks 和 claude hooks 的完整 shell 前导断言（`${CLAUDE_CONFIG_DIR:-$HOME/.claude}`、`_E="${CLAUDE_PLUGIN_ROOT:-${PLUGIN_ROOT:-}}"`、`while IFS= read -r _R`、`[ -f "$_Q/scripts/..." ]`、`command -v cygpath` 等）。

`tests/plugin-version-check.test.ts:10`：在 version-check 时演练 `CLAUDE_PLUGIN_ROOT: root` 环境变量注入。

### 0.4 现有构建时强制

`scripts/build-hooks.js`：
- 第 392–396 行：`.mcp.json` 与 `plugin/.mcp.json` 之间字节一致同步。
- 第 397–403 行：MCP 启动器必须包含 codex 缓存和 claude 缓存回退。
- 第 361–404 行：必需分发文件检查。
- 第 381–386 行：codex hook 事件名对照允许列表验证。
- 第 387–391 行：`.agents/plugins/marketplace.json` 的 source.path 必须为 `./plugin`。

### 0.5 计划将复用的现有工具

| 项目 | 位置 | 用途 |
|---|---|---|
| `CLAUDE_CONFIG_DIR` 常量 | `src/shared/paths.ts:41` | 在 shell 模板回退中用作 `${CLAUDE_CONFIG_DIR:-$HOME/.claude}` |
| `MARKETPLACE_ROOT` 常量 | `src/shared/paths.ts:43` | 由 `findMcpServerPath`、`findWorkerServicePath` 使用 |
| `shell-quote` 包 | 已在 `plugin/package.json` 依赖中（`scripts/build-hooks.js:101`） | 构建模板时用 `quote()` 转义字面 shell token |
| `findBunPath()`、`findMcpServerPath()`、`findWorkerServicePath()` | `src/services/integrations/CursorHooksInstaller.ts:84–130` | 被 Windsurf、Gemini、仅 MCP 安装器复用 — 已是事实上的集中化点 |

### 0.6 仍需的文档发现（Phase 0 子代理任务）

在 Phase 1 最终确定规范规则之前，部署文档发现子代理确认：

1. **Claude Code hook 规范。** Claude Code 文档是否说明 `CLAUDE_PLUGIN_ROOT` 在 hook spawn 时*保证*被设置？还是仅在 hook 从插件加载时（而非用户级 hook）？来源：https://docs.claude.com/claude-code/ — 找到 hook 契约页面。
2. **Codex CLI hook 规范。** Codex CLI 0.128+ 的同一问题。本仓库的 codex-hooks 模板防御该变量缺失；确认这是必要的还是偏执。来源：codex CLI 文档 / `codex --help plugin`。
3. **Cursor hook 契约。** 确认 Cursor 通过直接 exec（无 shell 展开）调用 hook 命令。当前安装器假设如此。来源：https://docs.cursor.com/。
4. **Gemini CLI hook 契约。** Gemini 的同一问题。
5. **Windsurf hook 契约。** Windsurf 的同一问题。
6. **OpenCode 插件契约。** 确认 OpenCode 通过 `OpenCodePluginContext.directory` 字段而非环境变量传递 plugin-root 信息。来源：`src/integrations/opencode-plugin/index.ts:11`。
7. **MCP 服务器协议。** 确认 IDE 拥有的 `mcp.json` 文件（Cursor、Copilot、Antigravity、Goose、Roo、Warp）中的 MCP 服务器注册不提供任何 `${VAR}` 替换 — 即这些宿主必须使用绝对路径。来源：Anthropic MCP 文档。

**子代理报告契约**（按 make-plan 技能）：每个发现必须引用（URL 或 file:line），包含引用的确切契约声明，并标记任何"这是隐含而非明示"的假设。

### 0.7 不存在的反模式 / API 方法（避免发明）

- 不存在现有的集中式 shell 模板生成器。Phase 2 必须创建它。
- 不存在跨安装器共享的现有 `getMcpServerAbsolutePath()` / `getBunAbsolutePath()` 辅助模块；每个都重复逻辑。Phase 3 必须创建它。
- `bun-runner.js` 的 `fixBrokenScriptPath()` 辅助函数就是权宜之计 — 在 Phase 5 验证确认没有剩余调用点会泄漏原始 `/scripts/...` 参数之前，本计划中不得删除它。
- `${CLAUDE_PLUGIN_ROOT}` 在 JSON 解析时**从不展开**。任何直接读取 `.mcp.json` 或 `hooks.json` 的代码都会看到字面字符串 `${CLAUDE_PLUGIN_ROOT}`，除非它 shell out 到 bash/sh。不要写假设相反的测试。
- 根据 Claude/Codex 市场规范，manifest 文件（`plugin.json`、`marketplace.json`）**不**支持 `${VAR}` 替换。不要提议添加。

---

## Phase 1 — 编纂规范解析规则

**要实现的内容：** 决策文档 + 对 `CLAUDE.md` 的修订。代码在 Phase 2–4 跟进。

### 1.1 三个选项（回顾）

(a) **安装时始终预解析为绝对路径。** 每个 hook/MCP 条目包含硬编码的 `/Users/<user>/.claude/plugins/cache/.../scripts/X.cjs`。优点：零 spawn-contract 表面。缺点：每次 claude-mem 版本升级都会使宿主不拥有的 IDE 配置（Cursor、Gemini、Windsurf、仅 MCP IDE、OpenClaw）中固化的路径失效。

(b) **始终依赖 POSIX-shell 防御性展开。** Hook/MCP 条目包含 `_E="${CLAUDE_PLUGIN_ROOT:-${PLUGIN_ROOT:-}}"; _P=$(... 回退链 ...)`。优点：跨升级无需重新安装。缺点：需要宿主可用的 POSIX shell（Windows 原生 cmd.exe 不符合；cygpath 变通已处理 Git-Bash/MSYS）。

(c) **通过包装脚本双重解析。** Hook/MCP 条目为 `node /known/path/wrapper.js <event>`；包装器在 JS 中解析真实 plugin root。优点：单一解析规则，易于测试。缺点：包装器本身需要已知绝对路径 → 对包装器自身安装位置回退到 (a)。

### 1.2 决策（编排器建议 — 在 Phase 0 子代理中确认）

采用**双规则拆分**，按谁拥有配置文件索引：

- **规则 A（宿主管理的 shell 模板）：** 宿主（Claude Code、Codex CLI）拥有配置文件（`hooks.json`、`codex-hooks.json`、`.mcp.json`、`plugin/.mcp.json`）且可能在插件升级时轮换缓存目录的站点。使用 POSIX-shell 防御性展开（选项 b）。
- **规则 B（安装器管理的固化）：** claude-mem 安装器拥有配置文件（Cursor、Gemini、Windsurf、仅 MCP IDE）的站点。使用绝对路径固化（选项 a）。在 `claude-mem` 版本升级时，安装器幂等地重新固化路径。
- **规则 C（运行时解析）：** `plugin/scripts/version-check.js` 和 `plugin/scripts/bun-runner.js` 按顺序接受 `CLAUDE_PLUGIN_ROOT` 环境变量和脚本自身的 `dirname(import.meta.url)/..`。这已是现状（version-check.js 第 7–17 行，bun-runner