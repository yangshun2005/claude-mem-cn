# SPIKE：Hook 直接调用 `claude-mem`（第一阶段证据）

**计划：** `plans/2026-10-02-claude-mem-on-path-hooks.md` · **日期：** 2026-10-02 · **基线：** `origin/main` @ `039c6160f`
**机器：** macOS 26.3 arm64，Claude Code 2.1.288，Bun 1.3.9。
**临时目录：** 所有内容位于 `.scratch/spike/`（被 `.gitignore:50` 忽略）。未改动产品代码。用户的真实安装（`~/.claude/plugins`、`~/.claude-mem`、`~/.claude/settings.json`）未被触碰。插件安装测试使用隔离的 `CLAUDE_CONFIG_DIR=.scratch/spike/cfg`。事后 grep 在真实 `settings.json`、`known_marketplaces.json` 和 `installed_plugins.json` 中找到 0 处 `spike` 匹配。

**门禁结果：第 1、2 项通过。exec 形式设计成立。没有任何事项阻塞第二阶段。**

计划未预料到三项发现。每项都会改变第二至第三阶段：
1. **`Setup` Hook 在插件安装或更新时从不触发**（第 8 项）。计划中的 Setup 自愈路线不可行。
2. **插件的 `bin/` 目录不在 Hook 的 PATH 上**（第 4 项）。但 `"command": "${CLAUDE_PLUGIN_ROOT}/bin/x"` 在 exec 形式下可用，即使路径包含空格和 emoji。这就是零 PATH 回退方案。
3. **`bun build --compile` 二进制默认从 cwd 自动加载 `.env`。** Hook 在用户项目目录中运行，因此启动器必须用 `--no-compile-autoload-dotenv` 编译（第 6 项）。

## 结论

| # | 项目 | 结论 |
|---|---|---|
| 1 | exec 形式无 shell 运行并接收 stdin | **通过** |
| 2 | exec 形式下 Hook 环境中有 `CLAUDE_PLUGIN_ROOT` | **通过** |
| 3 | 命令不在 PATH 上时会发生什么 | **通过**（已解答：非阻塞错误，退出码 1，会话继续） |
| 4 | 插件 `bin/` 可执行文件能否通过裸名称对 Hook 可见 | **失败**（不可见）。`${CLAUDE_PLUGIN_ROOT}/bin/<x>` 作为命令可用。 |
| 5 | macOS GUI 启动 PATH 是否包含 `~/.local/bin` | **部分**（有强间接证据表明是；未进行真实 Code 标签页启动） |
| 6 | 对约 30 行启动器执行 `bun build --compile` | **通过**（57.5 MiB，编译 0.13 秒，启动约 10 毫秒） |
| 7 | 编译后二进制上 `BUN_BE_BUN=1` 是否充当 `bun` CLI | **通过**（有注意事项，倾向选择 `findBun`） |
| 8 | `Setup` 何时触发 | **通过**（已解答：仅 `--init`、`--init-only`、`--maintenance`；安装、更新或正常启动时从不触发） |
| 9 | Windows：50 次工具调用中零可见窗口 | **待 Windows 测试者** |

以下每次 Claude Code 运行使用的通用测试装置：

```bash
S="$WORKTREE/.scratch/spike"            # WORKTREE = .../hooks-exec-form
cd "$S/project"; export PATH="$S/bin:$PATH" SPIKE_LOG="$S/logs/<run>.log"
claude -p "<prompt that triggers a Bash tool call>" \
  --plugin-dir "$S/plugin-<x>" --setting-sources project \
  --dangerously-skip-permissions --no-session-persistence \
  [--output-format stream-json --verbose --include-hook-events] [--debug-file "$S/logs/<run>.debug.log"]
```

`--setting-sources project` 将运行与用户插件隔离。调试日志仅显示 spike 插件和内置插件（`Found 2 plugins (2 enabled, 0 disabled)`：`spike-exec-form` 和 `cc-plugin-telemetry`）。用户的 claude-mem Hook **未**触发。

探针 `$S/bin/spike-hook` 是一个 `/bin/sh` 脚本。它将 `$0`、argv、`CLAUDE_PLUGIN_ROOT`/`DATA`、`CLAUDE_PROJECT_DIR`、`PATH`、`ps -o comm= -p $PPID` 和 stdin 追加到 `$SPIKE_LOG`。

---

## 1. 真实 Claude Code（macOS）中的 exec 形式：通过

Hook（`$S/plugin-a/hooks/hooks.json`）：
```json
{ "type": "command", "command": "spike-hook", "args": ["x", "${CLAUDE_PLUGIN_ROOT}", "has space"] }
```
命令：`claude -p "Use the Bash tool to run exactly: cat hello.txt . Then reply with its output." --plugin-dir "$S/plugin-a" --setting-sources project --dangerously-skip-permissions --no-session-persistence --debug-file "$S/logs/run1.debug.log"`

输出（`$S/logs/run1.log`，已裁剪）：
```
argv0=/Users/alexnewman/.superset/worktrees/claude-mem 🧠/hooks-exec-form/.scratch/spike/bin/spike-hook
argv[1]=x
argv[2]=/Users/alexnewman/.superset/worktrees/claude-mem 🧠/hooks-exec-form/.scratch/spike/plugin-a
argv[3]=has space
parent_comm=/Users/alexnewman/.local/bin/claude
grandparent_comm=/bin/zsh            <- the terminal that launched claude, not a hook shell
--- stdin ---
{"session_id":"a57b579d-…","hook_event_name":"PostToolUse","tool_name":"Bash","tool_input":{"command":"cat hello.txt",…},"tool_response":{"stdout":"hello",…},"tool_use_id":"toolu_01FSvnJAw71h49mNpLAsjpXe","duration_ms":4455}
```

这说明：
- **无 shell。** Hook 的父进程就是 `claude` 本身。在 shell 形式下会是 `sh`/`bash`。
- **精确 argv。** 带空格的参数保持为一个参数（`has space`），且 `${CLAUDE_PLUGIN_ROOT}` 被替换进 `args`，即使路径包含空格和 emoji。
- **Stdin。** 完整的 PostToolUse JSON 载荷通过 stdin 到达。
- **调试日志：** `Hook output does not start with {, treating as plain text`，因此按普通 Hook 处理。运行耗时 12.5 秒墙钟时间，包括模型回合。

## 2. Hook 环境中的 `CLAUDE_PLUGIN_ROOT`：通过

同一次运行（`$S/logs/run1.log`）：
```
CLAUDE_PLUGIN_ROOT=/Users/alexnewman/.superset/worktrees/claude-mem 🧠/hooks-exec-form/.scratch/spike/plugin-a
CLAUDE_PLUGIN_DATA=/Users/alexnewman/.claude/plugins/data/spike-exec-form-inline
CLAUDE_PROJECT_DIR=/Users/alexnewman/.superset/worktrees/claude-mem 🧠/hooks-exec-form/.scratch/spike/project
```
三者均在 exec 形式下作为环境变量导出。启动器可以优先依赖 `process.env.CLAUDE_PLUGIN_ROOT`，缓存和市场回退仍保留以增强健壮性。

## 3. 命令不在 PATH 上：通过（已解答）

Hook（`$S/plugin-b`）：`{ "type":"command", "command":"claude-mem-spike-does-not-exist", "args":["hook","claude-code","observation"] }`
命令：测试装置加 `--output-format stream-json --verbose --include-hook-events --debug-file "$S/logs/run3.debug.log"`，提示要求**两次** Bash 调用，然后回复 `DONE`。

流（`$S/logs/run3.stream.jsonl`），每次工具调用一次（2 次中的 2 次）：
```json
{"subtype":"hook_response","hook_name":"PostToolUse:Bash","exit_code":1,"outcome":"error",
 "stderr":"Error occurred while executing hook command: Executable not found in $PATH: \"claude-mem-spike-does-not-exist\""}
```
调试日志：
```
[ERROR] Hook command failed to spawn (PostToolUse:Bash): Executable not found in $PATH: "claude-mem-spike-does-not-exist"
[ERROR] "Hook PostToolUse:Bash (PostToolUse) error: status code 1\nstderr:\nError occurred while executing hook command: …"
```
最终结果：`"result":"DONE","is_error":false,"num_turns":3`。

这说明：
- 启动器缺失是**非阻塞错误**，报告为退出码 1，会话继续。
- 它在**每次 Hook 调用**时报告，而非一次。根据文档，交互式记录会显示 `<hook name> hook error` 通知，并附 stderr 第一行（https://code.claude.com/docs/en/hooks，"exit codes other than 2"）。
- 它响亮且重复，但从不阻塞。消息会指出缺失的可执行文件，因此故障排查页面可以以 `Executable not found in $PATH: "claude-mem"` 为关键。
- 交互式 UI 渲染未视觉确认，因为本次运行仅无头。

## 4. 插件 `bin/` 对 Hook 可见：失败（不可见）；`${CLAUDE_PLUGIN_ROOT}/bin/x` 可用

插件 `$S/plugin-c/bin/spike-bin-probe` 已确认不在 PATH 上（`command -v spike-bin-probe` 无返回）。

**4a. 裸名称：** `{ "command":"spike-bin-probe", "args":["from-plugin-bin"] }` 加上同一匹配器中的 PATH 探针（`spike-hook`）。
```
hook_response: {"exit_code":1,"outcome":"error","stderr":"Error occurred while executing hook command: Executable not found in $PATH: \"spike-bin-probe\""}
hook_response: {"exit_code":0,"outcome":"success"}        <- PATH probe; its logged PATH has no plugin-c entry
Bash tool, same session:  command -v spike-bin-probe -> …/.scratch/spike/plugin-c/bin/spike-bin-probe
                          PATH entry 40 = …/.scratch/spike/plugin-c/bin
```
因此 `bin/` 仅在 **Bash 工具**的 PATH 上。这与文档一致（"files there go on the Bash tool's PATH"）。它**不**在 Hook 的 PATH 上。

**4b. 插件根路径：** `{ "command":"${CLAUDE_PLUGIN_ROOT}/bin/spike-bin-probe", "args":["via-plugin-root-path"] }`
```
hook_response: {"exit_code":0,"outcome":"success"}
argv0=…/.scratch/spike/plugin-c/bin/spike-bin-probe
argv[1]=via-plugin-root-path
parent_comm=/Users/alexnewman/.local/bin/claude
```
exec 形式接受 `${CLAUDE_PLUGIN_ROOT}` 相对绝对路径作为 `command`，无需 PATH 查找，无需 shell。文档也推荐完全相同的模式：`"command":"node","args":["${CLAUDE_PLUGIN_ROOT}/scripts/format.js"]`。这**不是**发布版 `hooks.json` 中的每用户绝对路径，因为占位符在运行时替换。

## 5. macOS GUI 启动 PATH：部分

我没有打开任何 GUI 应用。证据：
```
$ launchctl getenv PATH
                                  <- empty; launchd default for GUI apps is /usr/bin:/bin:/usr/sbin:/sbin
$ ps eww -o command= -p <Claude.app main pid 96695> | tr ' ' '\n' | grep ^PATH=
PATH=/usr/bin:/bin:/usr/sbin:/sbin
$ ps eww -o command= -p <Claude.app child "disclaimer" pid 97801, parent of an MCP node> | … | grep ^PATH=
PATH=/Users/alexnewman/.kimi-code/bin:/Users/alexnewman/.local/bin:…:/opt/homebrew/bin:/Users/alexnewman/.bun/bin:…
$ grep -h "\[CCD\].*\(Shell environment\|login-shell\)" ~/Library/Logs/Claude/main*.log | tail
2026-09-15 13:50:27 [warn] [CCD] Shell environment extraction failed (attempt 1/5, willRetry=true), using process.env: Error: Shell environment extraction timed out
2026-09-15 13:56:57 [info] [CCD] Resolved 45 login-shell env vars
… (20 "Resolved 45 login-shell env vars" lines, 2026-09-04 → 2026-10-01; 1 failure, which then succeeded on retry)
```
Claude Code CLI **不**自行修复 PATH。模拟 launchd 最小启动：
```
env -i HOME=$HOME USER=$USER LOGNAME=$USER SHELL=/bin/zsh TMPDIR=$TMPDIR PATH=/usr/bin:/bin:/usr/sbin:/sbin \
  ~/.local/bin/claude -p "…echo hi…" --plugin-dir "$S/plugin-d" --setting-sources project …
hook "spike-hook" (bare name)           -> exit 1 "Executable not found in $PATH: \"spike-hook\""
hook "$S/bin/spike-hook" (absolute)     -> logged PATH=/usr/bin:/bin:/usr/sbin:/sbin
```
结论：
- CLI 将其继承的 PATH 直接传给 Hook。它不为 Hook 做登录 shell 探测。（调试日志中的 `Spawn-env probe captured 98 keys` 属于 Bash 工具快照。）
- 桌面应用（`[CCD]`）解析**登录 shell 环境**并提供给它生成的子进程。此处该 PATH 包含 `~/.local/bin`，因为 `.zshrc` 和 `.zprofile` 导出了它。安装器的追加器写入 `.zshrc`（`install.ts` `detectShellConfigFile`），该提取显然读取它：子 PATH 包含仅 `.zshrc` 的条目，如 nvm 和 `~/Library/pnpm`。
- **仍有两个缺口：**
  - (a) 当提取失败或超时时，桌面应用记录 `using process.env`，即 `/usr/bin:/bin:/usr/sbin:/sbin`。在重试成功前，`~/.local/bin`、`/opt/homebrew/bin` 和 `/usr/local/bin` 都缺失。
  - (b) 此处未启动真实 Code 标签页会话，因此桌面应用记录的 Hook PATH 仍未观察。

**剩余：** 打开一个 Code 标签页会话，安装 `$S/plugin-d` 探针（例如通过临时市场），并将记录的 PATH 与 `~/.local/bin` 对比。

## 6. 对约 30 行启动器执行 `bun build --compile`：通过

源码：`$S/launcher/launcher.ts`，26 行。它处理 `--version`、`hook <platform> <event>`、`CLAUDE_PLUGIN_ROOT` 解析、stdin 缓冲、`spawnSync(process.execPath, [worker-service.cjs, …], {env:{BUN_BE_BUN:"1"}, windowsHide:true})`，以及失败开放退出 0。
```
$ /usr/bin/time -p bun build --compile launcher.ts --outfile claude-mem
   [9ms]  bundle  1 modules
 [111ms] compile  claude-mem
real 0.13
$ stat -f %z claude-mem         -> 60291808   (57.5 MiB)
$ file claude-mem               -> Mach-O 64-bit executable arm64
$ ./claude-mem --version        -> 1                                        (exit 0)
$ ./claude-mem                  -> usage: claude-mem hook <platform> <event> (exit 0)
$ CLAUDE_PLUGIN_ROOT=/nonexistent ./claude-mem hook claude-code observation </dev/null
claude-mem: plugin root not found (/nonexistent), continuing without memory   (exit 0)
$ for i in 1..5; time ./claude-mem --version          -> real 0.01 (x5)
$ for i in 1..3; time (echo '{}' | ./claude-mem hook …)  -> real 0.04 (x3; includes spawning the stub worker)
```
宿主目标从本地 Bun 离线编译，无下载。

**发现的坑：** 编译后二进制从 cwd 自动加载 `.env`（`--compile-autoload-dotenv`，默认 true）。
```
$ cd $S/dotenv-probe   # contains .env: SPIKE_DOTENV_LEAK=leaked-from-project-env
$ ./envprint-default   -> launcher-entry SPIKE_DOTENV_LEAK=leaked-from-project-env
$ ./envprint-nodotenv  -> launcher-entry SPIKE_DOTENV_LEAK=<unset>     (built with --no-compile-autoload-dotenv)
```
Hook 在用户项目目录中运行，因此第三阶段**必须**用 `--no-compile-autoload-dotenv` 编译。另请注意，`bun <script>` 子进程（`BUN_BE_BUN` 或真实 bun）仍会自动加载 cwd `.env`。这也是当前通过 `bun-runner.js` 的行为，因此不是回归。

## 7.