# 计划：Hooks 直接调用 `claude-mem`，不再使用 bash 字符串

**日期：** 2026-10-02 · **跟踪项：** plan-17 #3605（Hook 包装器契约） · **取代：** #3403、#3564、#4182；重塑 #4282
**基线：** `origin/main` @ `039c6160f`。每个阶段都在从 `origin/main` 切出的全新 worktree 中工作，绝不从 `strategic-pr-merge` 切出，后者落后 230 个提交。

## 主要目标

`plugin/hooks/hooks.json` 中的每个 Claude Code hook 都变成一条简短的、不经过 shell 的命令：

```json
{ "type": "command", "command": "claude-mem", "args": ["hook", "claude-code", "observation"] }
```

这取代了今天约 1,700 字符的 bash 字符串。结果是：Windows 上零控制台窗口，任何平台都不需要 bash，也没有每次调用的 shell 开销。安装器把 `claude-mem` 放到 PATH 上，方式与 `rtk` 相同。

## 原因（面向用户的问题）

在 Windows 上，每个 claude-mem hook 都运行 `bash → node.exe → bun.exe`。Claude Code 隐藏了 bash 的控制台，但 bash 不会把 `SW_HIDE` 传给 `node.exe`，因此会闪过一个控制台窗口，并可能抢走键盘焦点：

- 在**每次**工具调用后一次（PostToolUse，匹配器 `*`），
- 每次 Read 之前再一次（PreToolUse `Read`），
- 以及每个提示、每轮结束、会话开始和结束时各一次。

一个 20 次工具调用的回合会产生约 27 次闪烁。相关报告：#3559、#3521、#3396（继 #676/#681/#748 之后的第三次复发）、#3248、#4121（每次工具调用约 3.5 秒的 bash 前置开销）。这些都已并入 plan-17，而 main 仍然在所有 8 个条目上发布 `"shell": "bash"`。

---

## 阶段 0：文档调研（已完成，已汇总）

### 允许使用的 API：Claude Code hooks（官方文档）
- **存在 exec 形式。** `"type": "command"` 的 hook 接受 `command` + `args`（数组）。当存在 `args` 时，`command` 会作为 PATH 上的可执行文件解析并直接启动，不经过 shell，且 `shell` 被忽略。来源：https://code.claude.com/docs/en/hooks#core-fields
- **字段：** `command`、`args`、`shell`（`"bash"` | `"powershell"`）、`async`、`asyncRewake`、`timeout`。仅此而已。
- **变量：** `${CLAUDE_PLUGIN_ROOT}`、`${CLAUDE_PLUGIN_DATA}` 和 `${CLAUDE_PROJECT_DIR}` 会在 `command` 和 `args` 中被字面替换。它们**也会作为环境变量导出**给 hook 进程。来源：https://code.claude.com/docs/en/plugins/manifest-reference#environment-variables
- **插件 `bin/`：** 插件启用期间，其中的文件会加入 **Bash 工具**的 PATH。**没有文档**说明 hook 进程能看到 `bin/`，因此不要依赖它（在阶段 1 验证）。
- **退出码：** 0 表示成功（解析 stdout JSON）。2 表示阻塞。其他任何码都是非阻塞错误。stdin 是 JSON 载荷。
- **未记录：** hook 子进程的 Windows 控制台可见性；GUI 启动的 Claude Code 如何构建 PATH。

### 当前流水线（origin/main）：从这里复制，不要重新发明
| 内容 | 位置 |
|---|---|
| bash 字符串生成器（根目录顺序：环境变量 → 按版本排序的缓存，跳过 `.orphaned_at` → marketplace） | `src/build/hook-shell-template.ts` L316–363（`buildShellCommand`） |
| **纯 Node 根目录解析器，已无 shell**（启动器解析器的模板） | `src/build/hook-shell-template.ts` L223–261（`buildMcpNodeLauncher`）、L276–309（`buildCodexWindowsCommand`） |
| 每个 hook 命令的清单 | `scripts/build-hooks.js` L106–176（`shellTemplateManifest`） |
| 规则 A 校验器（手工编辑时抛错） | `scripts/build-hooks.js` L183–292（`verifyShellTemplateCanonical`），在 L847 调用 |
| 运行时运行器：插件禁用检查、`findBun`、5 秒 stdin 缓冲、`windowsHide`、空 stdin 诊断 | `plugin/scripts/bun-runner.js` L23–285 |
| Worker `hook` 分发 | `src/services/worker-service.ts` L1566–1584 |
| 失败开放核心（任何失败时退出 0 + 一行 stderr） | `src/cli/hook-command.ts` L78–165 |
| 根目录判定 | `src/shared/worker-utils.ts` L563 `resolvePluginRoot()`、L471–543 |
| 安装器流程 | `src/npx-cli/commands/install.ts` `runInstallCommandInner` L2489；运行时设置 L2619–2652 |
| Shell rc PATH 追加器（POSIX，现有） | `src/npx-cli/commands/install.ts` L587–654（`applyClaudeCodePathSetupIfNeeded`） |
| Bun 安装和路径列表 | `src/npx-cli/install/setup-runtime.ts` L64–80、L184–219、L331–380 |
| 现有编译命令（先例） | `package.json` `build:cli-binary`；`scripts/build-worker-binary.js` L14–17 |
| 固定 bash 形态的测试（必须重写） | `tests/infrastructure/plugin-distribution.test.ts` L261–372、L457–498、L505–832 |
| 其他以命令文本为键的测试 | `tests/shared/host-hook-limits.test.ts` L34–36（`'hook claude-code context'`） |

### 影响设计的事实
- 失败开放逻辑已经存在于 TypeScript 中（`hook-command.ts`）。bash 字符串只负责找到 Node 和插件根目录。**这就是启动器需要做的全部。**
- 目前没有任何东西把 `claude-mem` 安装到 PATH 上。`package.json` 的 `bin` 指向一个 Node ESM CLI，其 `runHookCommand`（`src/npx-cli/commands/runtime.ts` L198–220）运行的是 **npm 包**的副本，会大声失败，且不适合 hooks。不要复用它。
- Marketplace 自动更新（`autoUpdate: true`，`install.ts:207`）不运行任何安装器代码。因此 PATH 上的二进制会变旧，除非它**版本稳定**。启动器不得包含任何在版本之间变化的 claude-mem 逻辑。
- 安装器已保证 Bun（`ensureBun`）。针对宿主目标的 `bun build --compile` 使用本地 Bun，不需要网络，也不需要 CI 矩阵。
- 没有 macOS CI 任务。Windows CI 是 `windows.yml`（`windows-2022`）。

### 反模式（不要做）
- 不要发明 hook 字段；只存在上述六个字段。
- 不要在发布的 `hooks.json` 中放入每用户绝对路径。
- 不要把编译后的二进制提交到 git（`plugin/scripts/claude-mem` 在 `.gitignore:42` 被忽略是有原因的）。
- 不要复用 `runHookCommand` 或 npx ESM CLI 作为 hook 入口。
- 不要在启动器中重现 claude-mem 逻辑（没有处理器，没有 HTTP）。它只解析根目录、运行 `worker-service.cjs`，并失败开放。
- 不要手工编辑 `plugin/hooks/hooks.json`。它必须经过生成器和规则 A。

---

## 阶段 1：Spike（验证每个假设；不写产品代码）

**目标：** 在构建之前把未知变成事实。将结果写入 `plans/2026-10-02-claude-mem-on-path-hooks.SPIKE.md`，包含命令、输出和结论。使用一次性的本地插件或临时 worktree，绝不要用用户的真实安装。

验证：
1. **真实 Claude Code（macOS）中的 exec 形式。** 安装一个临时插件，带一个 PostToolUse hook `{"command":"<name>","args":["x"]}`，其中 `<name>` 是 PATH 上的脚本，把其 argv 和 stdin 追加到文件。确认它在无 shell 下运行并接收 stdin。
2. **`CLAUDE_PLUGIN_ROOT` 在 exec 形式下位于 hook 的环境中**（`process.env.CLAUDE_PLUGIN_ROOT`）。
3. **命令不在 PATH 上：** Claude Code 显示什么？退出码、UI 消息、是否阻塞？这决定"缺少启动器"的提示有多响亮。
4. **插件 `bin/` 中的可执行文件对 hooks 可见吗？** 用 `bin/` 文件和 exec 形式测试。如果可见，它就成为零安装回退方案；记录下来。
5. **macOS 上 GUI 启动的 PATH：** 从桌面应用的 Code 标签页或非终端方式启动 Claude Code，并从 hook 记录 `PATH`。`~/.local/bin` 是否存在？
6. **`bun build --compile`** 一个 30 行的启动器：确认可用，记录其大小和时间，并确认它能运行。
7. **`BUN_BE_BUN=1`：** 用 `BUN_BE_BUN=1` 启动编译后启动器自身的 `process.execPath`，是否会作为 `bun` CLI 并运行 `worker-service.cjs`？引用 Bun 文档。如果不行，启动器使用 `bun-runner.js` 中的 `findBun`。
8. **`Setup` hook 何时触发**（安装、更新、`--init`）？引用文档加一次观察到的运行。这决定 Setup 能否为仅用 marketplace 的用户自安装启动器。
9. **Windows（需要真实 Windows 机器；如果没有本地机器，请找社区 Windows 测试者）：** 指向 PATH 上 `.exe` 的 exec 形式 hook 在 50 次工具调用中给出**零**可见窗口。记录谁验证的以及如何验证。

**验证：** SPIKE.md 对全部 9 项都有带证据的结论。第 1–8 项在这台 Mac 上运行。第 9 项要么已验证，要么明确标记为 PENDING-WINDOWS-TESTER，它只阻塞阶段 6，不阻塞阶段 2–5。

**阶段 1 后停止。** 如果第 1 或第 2 项失败，停止并报告；设计需要改变。

---

## 阶段 2：启动器（版本稳定，极小）

**目标：** `src/launcher/claude-mem-launcher.ts`。它唯一的职责是：`claude-mem hook <platform> <event>` → 找到插件根目录 → 用 hook 的 stdin 运行 `worker-service.cjs hook <platform> <event>` → 任何失败时退出 0，并输出一行 stderr。

**复制，不要发明：**
- **根目录解析：** 移植 `buildMcpNodeLauncher`（`hook-shell-template.ts` L223–261）和 `buildCodexWindowsCommand`（L276–309）。顺序是 `CLAUDE_PLUGIN_ROOT` / `PLUGIN_ROOT` 环境变量，然后是按版本排序的缓存（跳过 `.orphaned_at`），然后是 marketplace。`scripts/bun-runner.js` 和 `scripts/worker-service.cjs` 都必须存在。使用与 bash 模板 L170–175 相同的 `_E/plugin` 规范化。
- **插件禁用检查、stdin 缓冲、带 `windowsHide: true` 的 spawn、空 stdin 诊断和退出码映射：** 从 `plugin/scripts/bun-runner.js` L101–285 移植。
- **Bun：** 如果阶段 1 第 7 项通过，在 `process.execPath` 上使用 `BUN_BE_BUN=1`。否则使用 `bun-runner.js` L23–99 中的 `findBun`。
- **失败开放：** 匹配 `hook-command.ts` L138–161。根目录未找到、Bun 未找到或 spawn 错误各打印一行 `claude-mem: <reason>, continuing without memory`，并退出 0。**绝不退出 2。**
- **子命令：** `hook <platform> <event>` 和 `--version`。版本打印 `LAUNCHER_PROTOCOL`（一个整数常量，从 1 开始）。其他任何内容打印用法并退出 0。

**构建：** 在 `scripts/build-hooks.js` 中其他目标旁边添加一个 esbuild 目标（L380–430 模式），输出 `plugin/scripts/claude-mem-launcher.cjs`，像其他 bundle 一样提交。不要在构建中编译；编译发生在用户机器上（阶段 3）。

**测试：** `tests/launcher/claude-mem-launcher.test.ts` 用 Bun 针对临时 HOME 运行 bundle。用例：
- 环境变量根目录
- 缓存回退，最高版本胜出，孤立版本被跳过
- marketplace 回退
- 未找到根目录：退出 0 加 stderr 行
- 插件禁用：退出 0
- stdin 完整传递给 stub `worker-service.cjs`
- 子进程以 7 退出：启动器退出 0（失败开放）

从 `plugin-distribution.test.ts` L635–832 复制用例形态。

**验证：** `bun test tests/launcher/`、`npm run typecheck`、`npm run build`。另外 `grep -n "fetch\|http" src/launcher/` 无返回（启动器中无 claude-mem 逻辑）。

**反模式防护：** 不从 `src/services`、`src/cli/handlers` 或 `src/shared/worker-utils` 导入（启动器必须保持版本稳定）。除 `.cmd`/`.bat` 外不使用 `shell: true`，与 `bun-runner.js` 完全一致。

**阶段 2 后停止。**

---

## 阶段 3：把启动器安装到 PATH（并保持在那里）

**目标：** `npx claude-mem install` 和 `update` 编译启动器并把 `claude-mem` 放到 PATH 上。仅用 marketplace 的用户自动获得它，缺少启动器时提示响亮，绝不静默。

1. **安装时编译：** 在运行时设置步骤（`install.ts` L2619–2652）中的 `ensureBun` 之后，运行 `bun build --compile <cache>/plugin/scripts/claude-mem-launcher.cjs --outfile <binDir>/claude-mem[.exe]`。
2. **Bin 目录：**
   - POSIX：`~/.local/bin`。复用 `install.ts` L587–654 的 rc 文件追加器，将其泛化，使其在 `~/.local/bin` 不在 PATH 上时运行，而不仅在安装 Claude Code 之后。
   - Windows：`%LOCALAPPDATA%\claude-mem\bin`，添加到**用户** PATH。通过 PowerShell 使用 `[Environment]::SetEnvironmentVariable('Path', …, 'User')`，仅在不存在时追加，绝不截断；禁止使用 `setx`，因为它会在 1024 字符处截断。
   - 如果阶段 1 第 5 项显示 macOS 上 GUI 启动缺少 `~/.local/bin`，还要符号链接到 `/opt/homebrew/bin` 和 `/usr/local/bin` 中第一个可写目录，并记录使用了哪个。
3. **当前时跳过：** 如果 `claude-mem --version` 已打印当前 `LAUNCHER_PROTOCOL`，什么都不做。
4. **仅 marketplace 安装的自愈：** 如果阶段 1 第 8 项确认 Setup 在安装和更新时触发，那么 `plugin/scripts/version-check.js`（Setup 运行的）在启动器缺失或其协议过旧时执行相同的编译并放置步骤。如果 Setup 不在插件安装时触发，使用 SPIKE.md 中选择的回退方案（例如第 4 项的插件 `bin/`）并记录决定。
5. **卸载：** `commands/uninstall.ts` L57–68 移除它放置的二进制。它不动 rc 文件，并打印要移除的行。
6. **Doctor：** 添加一行 `doctor`：PATH 上的启动器（位置）、其协议，以及一次计时的 `claude-mem --version` 往返。

**测试：** 扩展现有安装器测试（用 `grep -rl "runInstallCommand\|applyClaudeCodePathSetupIfNeeded" tests/` 找到它们）。覆盖：按平台用正确 outfile 调用编译；Windows PATH 追加幂等且绝不截断；当前时跳过；卸载移除二进制。

**验证：**
- 类型检查、构建、测试。
- 在这台 Mac 上：把本地构建 `npx` 到临时 HOME（`HOME=$(mktemp -d -p .scratch)`），然后从全新 shell 确认 `command -v claude-mem` 和 `claude-mem --version`。

**阶段 3 后停止。**

---

## 阶段 4：把 hooks 切换到 exec 形式，并让 bash 无法回归

**目标：** 每个运行时 Claude Code hook 变成 `{"command":"claude-mem","args":["hook","claude-code","<event>"]}`，事件、匹配器、`async` 和超时与今天相同（阶段 0 来源中的表格：`build-hooks.js` L125–150）。

1. **生成器：** 在 `hook-shell-template.ts` 中 `buildShellCommand`