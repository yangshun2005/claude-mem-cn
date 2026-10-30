# 移除 Gemini CLI，新增 Antigravity CLI（完全对齐）

## 原因

Google 已正式确认（2026 年 5 月 19 日，https://developers.googleblog.com/an-important-update-transitioning-gemini-cli-to-antigravity-cli/）：免费/个人层级的 Gemini CLI 访问已于 2026 年 6 月 18 日终止，转而推荐 **Antigravity CLI**——一个独立的、支持无头模式的二进制文件（`https://github.com/google-antigravity/antigravity-cli`），它复用 Gemini CLI 的 `~/.gemini/` 配置树，并"保留 Gemini CLI 最关键的功能：Agent Skills、Hooks、Subagents 和 Extensions"。本计划移除 claude-mem 的 Gemini CLI *宿主*集成，替换为功能对齐的 Antigravity CLI 集成。

**范围锁定（已与用户确认）：**
- 仅移除 Gemini CLI 宿主集成（适配器、安装器、Hook、IDE 检测入口、文档）。**不要触碰**独立的、无关的 Gemini LLM/观察 *provider*（`CLAUDE_MEM_GEMINI_API_KEY`、`GeminiProvider.ts`、`GeminiObservationProvider.ts`、`SettingsDefaultsManager.ts` 中的 Gemini 键、`ContextSettingsModal.tsx` 中的 Gemini 表单、`docs/public/usage/gemini-provider.mdx`、`docs/public/cursor/gemini-setup.mdx`、`openclaw/install.sh --provider gemini`）。那是另一个 Google 产品（Gemini API），并未被弃用。
- Antigravity CLI 的 Hook 接线将基于一个工作假设构建：其 Schema 与 Gemini CLI 的 Hook 系统兼容（同一厂商声明："相同的底层 agent harness"），直接以 `GeminiCliHooksInstaller.ts` 为模板。**针对真实 Antigravity CLI 安装的手动验证是硬性合并门槛**（Phase C）——Google 未独立记录该 Schema（其文档页面是不可抓取的 Angular SPA），绝不能仅凭假设就发布。
- **更新：** Phase B0 此后已在开发机上针对一个真实、已安装的 Antigravity CLI 完成（不再是假设——见下方"B0 发现"）。Phase C 的手动门槛从"从零安装并配置"降级为"确认发布的安装器输出与磁盘上已验证可用的内容一致"，因为本机已存在来自手动/先前使用的实时 Hook/MCP/上下文文件。

## Phase 0 — 文档调研（规划期间已完成）

**允许作为构建依据的事实**（有引用，非杜撰）：

| 事实 | 来源 | 置信度 |
|---|---|---|
| Antigravity CLI 是真实的独立二进制文件，通过 `curl -fsSL https://antigravity.google/cli/install.sh \| bash` 安装 | https://github.com/google-antigravity/antigravity-cli | 高——一手来源 |
| 存在无头标志：`-p`/`--print`、`--sandbox`、`-c`/`--conversation`、`--model` | CLI 自带的 CHANGELOG.md（直接获取）+ https://codelabs.developers.google.com/antigravity-cli-hands-on | 高 |
| 全局指令文件为 `~/.gemini/GEMINI.md`；工作区规则位于 `.agent/rules/`，工作流位于 `.agent/workflows/` | https://atamel.dev/posts/2025/11-25_customize_antigravity_rules_workflows/（直接获取） | 高 |
| AGENTS.md/CLAUDE.md 默认不会被自动加载 | 同上 | 中（2025 年 11 月日期，目前未确认） |
| 原生支持 MCP；格式为 `mcpServers` 对象，远程用 `serverUrl`（不是 `url`/`httpUrl`），无顶层 `timeout`，无 JSON 注释 | https://cloud.google.com/blog/products/data-analytics/connect-google-antigravity-ide-to-googles-data-cloud-services + https://medium.com/google-cloud/configuring-mcp-servers-and-skills-for-antigravity-cli-and-ide-a938c7eebb78 | 中——第三方佐证，非 Google 原文 |
| 据第三方文档，MCP 配置路径为 `~/.gemini/config/mcp_config.json` | 同上 Medium 文章 | **与现有代码库假设冲突**（`McpIntegrations.ts:127` 中为 `~/.gemini/antigravity/mcp_config.json`）——必须在 Phase B0 验证哪个为真 |
| Google 确认"Hooks"功能名从 Gemini CLI 沿用 | https://developers.googleblog.com/an-important-update-transitioning-gemini-cli-to-antigravity-cli/（直接获取，已引用） | 该声明为高，**确切 Schema 未验证** |
| Antigravity 的确切 Hook 事件名/JSON 结构 | 任何可抓取来源均未找到 | **未知——不要杜撰。** 仅以 Gemini CLI 的 Schema 作为起始假设。 |

**反模式防护：** 不要编写断言上表之外、或 Phase B0 未确认的 Hook 事件名、CLI 标志或配置路径的安装器代码。不确定处，复制 Gemini CLI 的确切 Schema，并在行内标注 `// HYPOTHESIS: verify against real Antigravity CLI install`——不要默默假设正确。

---

## Phase A — 移除 Gemini CLI 宿主集成

### A1. 删除专用文件
- `src/cli/adapters/gemini-cli.ts`（79 行，整个文件）
- `src/services/integrations/GeminiCliHooksInstaller.ts`（393 行，整个文件）
- `tests/gemini-cli-compat.test.ts`（204 行，整个文件）
- `docs/public/gemini-cli/setup.mdx`（192 行，整个文件/目录）

### A2. 从枢纽文件注销
- `src/cli/adapters/index.ts` — 移除 `import { geminiCliAdapter } from './gemini-cli.js';`（第 5 行），移除 `case 'gemini': case 'gemini-cli': return geminiCliAdapter;`（第 14-15 行），从桶导出中移除 `geminiCliAdapter`（第 22 行）。
- `src/npx-cli/commands/ide-detection.ts` — 移除 `gemini-cli` 条目（第 49-53 行）。
- `src/npx-cli/commands/install.ts` — 移除动态导入 `installGeminiCliHooks()` 的 `case 'gemini-cli':` 分支（约第 323-335 行）。
- `src/npx-cli/commands/uninstall.ts` — 移除 `{ label: 'Gemini CLI hooks', ... }` 条目（约第 358-360 行）。
- `src/services/worker-service.ts` — 移除 `import { handleGeminiCliCommand } from './integrations/GeminiCliHooksInstaller.js';`（第 73-75 行），移除 `case 'gemini-cli': { ... }` 分发（第 1233-1237 行），更新 `console.error('Platforms: ...')` 帮助文本（第 1250 行）去掉 `gemini-cli`。
- `src/cli/handlers/context.ts` — 移除 `platform === 'gemini-cli' || platform === 'gemini'` 显示分支（第 137 行）。**先验证**该分支是关于 Hook 调用的平台显示，而非 `CLAUDE_MEM_PROVIDER`/LLM provider 选择——不要触碰任何读取 `CLAUDE_MEM_PROVIDER` 的内容。
- `src/npx-cli/index.ts` — 从帮助文本中移除 `gemini-cli`（第 56 行）。
- `src/services/integrations/install-paths.ts` — 更新第 5 行列出 Gemini 为安装器之一的注释。

### A3. 测试
- `tests/install-error-matrix.test.ts` — 从 IDE 矩阵中移除 `'gemini-cli'`（第 24 行）。
- `tests/infrastructure/plugin-distribution.test.ts` — 移除期望 `'src/services/integrations/GeminiCliHooksInstaller.ts'` 出现在发布文件清单中的断言（第 436 行）。

### A4. 文档
- `docs/public/docs.json` — 移除 `"group": "Gemini CLI Integration"` 导航条目（第 62-65 行）。
- `docs/public/installation.mdx` — 从支持的 IDE 列表中移除"Gemini CLI"（第 20、22、47 行）。
- `docs/public/introduction.mdx` — 从支持的 IDE 列表中移除"Gemini CLI"（第 83 行）。
- `README.md` — 移除 `npx claude-mem install --ide gemini-cli` 行（139-142），"Restart Claude Code or Gemini CLI"提及（第 158 行），以及设置文档链接（第 194 行）。

### A5. 验证 Gemini transcript-parser 对此关注点确实已无用
- `src/shared/transcript-parser.ts` — `isGeminiTranscriptFormat()`/`extractLastMessageFromGeminiTranscript()` 解析 Gemini CLI 的 transcript 文件格式。删除前，grep 调用方；若摘要流水线因其他原因仍需它，则保留——否则删除。**不要盲目删除。**

### A6. Phase A 验证清单
- `grep -ril "gemini-cli\|GeminiCliHooksInstaller\|geminiCliAdapter" src/ tests/ docs/public/ README.md` 返回零命中（CHANGELOG.md 历史除外，保留）。
- `grep -ril "CLAUDE_MEM_GEMINI\|GeminiProvider\|GeminiObservationProvider" src/` 仍返回所有原始 Part-B 命中——确认 LLM provider 未被触碰。
- `npm run build-and-sync` 成功。
- 完整测试套件在删除测试文件后通过（非跳过）。

---

## Phase B — 新增 Antigravity CLI（完全对齐）

### B0. 验证探查——已完成（发现如下，2026-07-03 针对实时安装确认）

开发机已安装并积极使用 Antigravity IDE 和独立 Antigravity CLI，包括一个**预先存在、实时、可用的 claude-mem Gemini CLI Hook 安装**共享同一配置树——这使 B0 从假设练习变为直接的 filesystem/二进制检查。未执行网络安装；未触发实时 `agy` 会话/模型调用（避免消耗用户 API 配额/未经请求变更其插件状态）。

**已确认事实：**

1. **二进制名称**：`agy`（`~/.local/bin/agy`，v1.0.16）是真正的独立无头 Antigravity CLI。`antigravity`（`~/.antigravity/antigravity/bin/antigravity`，v1.107.0）是**不同的二进制文件**——桌面 IDE 的内部可执行文件，不是 CLI。检测必须检查 `isCommandInPath('agy')`，而非 `antigravity`。
2. **Hook 配置位置——已确认**：`~/.gemini/settings.json`，与 Gemini CLI 使用的完全同一文件。证据：claude-mem 预先存在的 `gemini-cli` Hook（由当前即将移除的 `GeminiCliHooksInstaller.ts` 安装）实时存在于该文件中，而该机器已安装 `agy` 并日常使用——同一文件，同一 JSON Schema（`hooks.<Event>[].hooks[].{name,type,command,timeout}`）。这证明 Antigravity CLI 读取共享的 Gemini 设置文件，而非单独文件。不存在单独的 `~/.gemini/antigravity-cli/settings.json` Hook 块——该文件仅保存无关的 CLI 偏好（`colorScheme`、`model`、`trustedWorkspaces`、`enableTelemetry`），不含 Hook。
3. **Hook 事件名——已确认超集**：实时安装的配置有 **8** 个事件，而非当前 `GeminiCliHooksInstaller.ts` 源码中的 7 个：`SessionStart`、`BeforeAgent`、`AfterAgent`、`BeforeTool`、`AfterTool`、`Notification`、`PreCompress`，**以及 `SessionEnd` → `session-complete`**（磁盘上存在，但当前源码的 `GEMINI_EVENT_TO_INTERNAL_EVENT` 映射中缺失——来自旧版安装器的预先存在的文档/代码漂移）。新 Antigravity 映射使用全部 8 个，因为磁盘配置证明全部 8 个都是有效、实时的事件名。
   - `agy --help` 没有专门的 `hooks` 子命令——Hook 完全由配置文件驱动（在 settings.json 中声明，无 CLI 注册步骤），与 Gemini CLI 自身模型完全一致。
4. **MCP 配置路径——存在两个真实文件，确实模糊，需要双写**：
   - `~/.gemini/antigravity/mcp_config.json`（旧路径，已由现有仅 MCP 安装器注册了 claude-mem，真实内容，使用中）。
   - `~/.gemini/config/mcp_config.json`（存在但**为空**，目录最近与 CLI 自身 app-data 目录一同创建）——匹配第三方文档声称的较新/统一路径，但本机尚无任何内容写入真实内容。
   - **决定：写入两个路径。** 两者都是廉价的幂等 JSON 合并（对每个路径逐字复用 `McpIntegrations.ts` 中的 `writeMcpJsonConfig`）——在通过实时工具调用测试确认 `agy` 实际读取哪个之前，这是安全的保险。不要凭猜测只选一个。
5. **规则/上下文文件路径——已确认，现有代码本就正确**：`~/.agents/rules/`（**复数** "agents"）真实且已填充（`~/.agents/rules/claude-mem-context.md` 已由当前 `ANTIGRAVITY_CONFIG` MCP 安装器存在）。早先研究引用单数 `.agent/rules/` 对此安装是错误的——任何地方都不存在 `.agent/`（单数）目录。保持 `McpIntegrations.ts:129` 中现有代码库的路径假设不变。
6. **GEMINI.md——已确认实时**：`~/.gemini/GEMINI.md` 真实、积极使用中（有原生 Antigravity "Gemini Added Memories"自动条目，加上 claude-mem 自己的 `<claude-mem-context>` 标签块已注入并正确渲染）。上下文注入目标无需更改。
7. **新架构发现（对齐非必需，后续候选）：** `agy plugin {list,import,install,uninstall,enable,disable,validate,link}` 是一流的插件市场子命令系统，结构类似 Codex CLI 的 `codex plugin marketplace`——特别是 `agy plugin import gemini|claude` 暗示原生跨工具插件迁移。`agy plugin list` 当前在本机报告"No imported plugins"（未使用）。这可能比手动编辑 `settings.json` 更干净、更地道的集成路径（像 Codex 的 `.codex-plugin/plugin.json` 那样打包 Hook+MCP+技能注册），但其 manifest Schema 无法在不实际针对真实 manifest 运行 `agy plugin import`/`install` 的情况下发现，而这会变更用户的实时本地插件状态——**本计划范围外**；现在发布已验证的 settings.json + 双 mcp_config.json 方案，在新文档页中将插件市场路径记为未来增强。

无需进一步 B0 工作——以确认值而非假设进入 B1。

### B1. IDE 检测——升级现有 `antigravity` 条目，不要分叉

`src/npx-cli/commands/ide-detection.ts` — 现有条目（第 90-95 行）仅 MCP。就地升级，而非添加重复的 `antigravity-cli` id（据 Google，Antigravity IDE 和 CLI 共享同一 `~/.gemini/antigravity` 配置命名空间——第二个几乎相同的条目会混淆安装选择器）：
```ts
{
  id: 'antigravity',
  label: 'Antigravity',
  detected: existsSync(join(home, '.gemini', 'antigravity')) || isCommandInPath('agy'),
  hint: 'hooks + MCP integration',
}
```

### B2. 新安装器：`src/services/integrations/AntigravityCliHooksInstaller.ts`

整体复制 `GeminiCliHooksInstaller.ts` 作为起点（遵循现有代码库自身模式——Cursor/Windsurf/Gemini 安装器已互相复制；尚无共享工厂，据 `.plan` 观察"安装器代码重复审计显示 44% 可减少 LoC"，所以不要在此发明一个——本计划范围外）。具体：
- 幂等 Hook 合并机制（Gemini 文件第 97-131 行的 `mergeHooksIntoSettings`）是通用 JSON 组合并——逐字复制，它不依赖事件名。
- 保持 `GEMINI_SETTINGS_PATH`/`GEMINI_MD_PATH` 指向相同的已确认路径（`~/.gemini/settings.json`、`~/.gemini/GEMINI.md`）——这些与（前）Gemini CLI 共享，B0 已确认，非单独文件。
- 使用已确认的 8 事件映射：`SessionStart→context, BeforeAgent→session-init, AfterAgent→observation, BeforeTool→observation, AfterTool→observation, Notification→observation, PreCompress→summarize, SessionEnd→session-complete`（第 8 个 `SessionEnd` 相对被移除的 Gemini 安装器 7 事件源码映射是新增的，但据 B0 确认为实时/有效——在此沿用）。
- 将 Hook 命令字符串 `hook gemini-cli ${internalEvent}` 替换为 `hook antigravity-cli ${internalEvent}`。
- 将 MCP 服务器注册并入同一安装器，写入**两个**已确认路径（`~/.gemini/antigravity/mcp_config.json` 和 `~/.gemini/config/mcp_config.json`），