# 文件读取门禁评测

在真实 Claude Code 会话中、针对真实 claude-mem worker，验证文件读取门禁（`CLAUDE_MEM_FILE_READ_GATE_ENABLED`）：

- **门禁开启**（默认）：对 32 KB 及以上、且已有观察记录（observation）的完整代码文件执行 Read 会被拒绝，Claude 仍能回答关于该文件的问题并编辑它。它通过 `smart_outline` / `smart_unfold`、`get_observations`，或带 `offset` 和 `limit` 的 Read 获取所需内容。定向 Read 也能满足 Edit 的"编辑前先读"规则。较小的文件正常读取，并附带其时间线作为上下文。
- **门禁关闭**：同样的 Read 会放行，并附带文件的观察时间线作为上下文，不会拒绝任何操作。
- **节省**：对大文件，门禁开启时每次运行的成本低于门禁关闭。

从仓库根目录运行：

```bash
npm run build                                # 运行器会拒绝未构建门禁的 worker，以及异步 Read Hook
npm run eval:read-gate -- --preflight-only   # 不调用模型：用真实 Hook 对两个 worker 做检查
npm run eval:read-gate                       # 在 claude-sonnet-5-5 上跑 3 个用例 x 3 次
npm run eval:read-gate -- --runs 1 --case gate-on-edits-file
```

| 参数 | 默认值 | 作用 |
| --- | --- | --- |
| `--runs <n>` | `3` | 每个用例的运行次数 |
| `--model <id>` | `claude-sonnet-5-5` | Agent 模型；必须是明确的 Claude 模型 ID |
| `-j <n>` | `3` | 同时运行的任务数（1-8） |
| `--max-cost-usd <usd>` | `10` | 传给 `claude plugin eval`：按标价估算达到该值后不再启动新运行 |
| `--case <glob>` | 全部用例 | 原样传给 `--case`；未运行的用例其判定报告为未运行 |
| `--preflight-only` | 关闭 | 预检后停止；不调用模型 |

测试套件位于 `evals/read-gate/`，在 `plugin/` 之外，因为市场安装会从 git 复制 `plugin/`，而它们都不需要该套件。`claude plugin eval --eval-dir` 只读取插件下的目录，因此运行器每次运行会把套件复制到 `plugin/evals-read-gate/`（已 gitignore），运行结束后删除。

## 用例

| 用例 | 分组 | 文件 | 提示词 | 评分器 |
| --- | --- | --- | --- | --- |
| `gate-on-large-file` | 开启 | 大 | `carrier-tariffs.ts` 如何为承运商报价定价？请逐步说明，并给出锂电池附加费的每公斤费率和最低收费 | `read-blocked`（trace 中出现拒绝标记）、`answer-rate`、`answer-minimum` |
| `gate-off-large-file` | 关闭 | 大 | 与 `gate-on-large-file` 相同的问题 | `not-blocked`（无拒绝标记）、`read-used`、`answer-rate`、`answer-minimum` |
| `gate-on-edits-file` | 开启 | 大 | 查看该文件，然后把锂电池最低收费从 22.40 改为 24.75；其他不动 | `read-blocked`、`edited` 和 `old-value-gone`（运行后的文件） |
| `gate-on-small-file` | 开启 | 小 | `rate-calculator.ts` 如何为货运定价？请逐步说明，并给出 `calculateRemoteAreaSurcharge` 应用的准确费率和最低费用 | `not-blocked`、`read-used`、`answer-rate`、`answer-minimum` |
| `gate-off-small-file` | 关闭 | 小 | 与 `gate-on-small-file` 相同的问题 | `not-blocked`、`read-used`、`answer-rate`、`answer-minimum` |

大文件是 `fixture-large/src/shipping/carrier-tariffs.ts`：941 行、49 KB，包含七家承运商的费率表和附加费函数，锂电池附加费位于中间。这使其超过门禁 32 KB 的拒绝大小，又远低于 Claude Code Read 工具 25,000 token 的上限——超过该上限时，整文件 Read 会直接失败，与门禁无关。小文件是 `fixture/src/shipping/rate-calculator.ts`：513 行、19 KB，低于拒绝大小。每个文件位于各自的项目中，各有自己的脚手架（`scaffold-large-gate-on.sh` / `scaffold-large-gate-off.sh` 和 `scaffold-gate-on.sh` / `scaffold-gate-off.sh`），因此一个用例只会看到其中一个。每个问题所问的数字只出现在某一个函数内。`seed-observations.json` 包含六条观察记录，四条关于小文件、两条关于大文件，其标题和事实描述结构，但从不给出这些数字，因此正确答案必须来自代码。

小文件用例正是拒绝大小存在的原因。在此之前，门禁也会拒绝小文件，同样的问题在门禁开启时 Sonnet 便宜 6%，但 Opus 贵 20%。Opus 先运行 `smart_outline`，然后通过定向 Read 读取了约 70% 的文件，因此拦截几乎没省下什么，还多花了四轮，每轮都重新读取整个上下文。在大文件上，同样的拦截让 Sonnet 节省 35%、Opus 节省 32%。现在小文件在门禁开启时正常读取，这对用例检查其成本与门禁关闭时相同。

提示词要求逐步说明文件，并在编辑前查看它，因为一个针对性的问题从不需要 Read。第一版只问两个数字（"calculateRemoteAreaSurcharge 应用什么费率和最低费用？"、"把最低费用从 4.85 改为 5.25"）。Claude 随后用 2 到 4 轮、一次 Grep 就回答了，其上下文行展示了整个 8 行函数，两个分组都没有运行尝试整文件 Read。Grep 无法被扣留：无论用例的 `allowed_tools` 列出什么，`claude plugin eval` 总是授予只读工具（Read、Glob、Grep、Task 等）。关于整个文件如何工作的问题，正是整文件 Read 所服务的。因此门禁开启的运行会遇到拒绝，门禁关闭的运行会正常读取文件。

## 运行如何隔离

`scripts/eval-read-gate.ts` 在 `.scratch/read-gate-eval/<timestamp>/{on,off}/data` 下创建两个数据目录。每个目录有自己的空闲端口、一个门禁开启或关闭的 `settings.json`，以及一份用六条观察记录播种的数据库副本，日期设为一天前。然后为每个目录启动一个 worker。每个用例的脚手架（`scaffold-gate-on.sh` / `scaffold-gate-off.sh`，都调用 `scaffold-workspace.sh`）把 fixture 复制到运行的工作区，日期设为 2026-01-01，使其早于观察记录，并把运行的 `~/.claude-mem` 链接到其分组的数据目录。因此一次运行的 Hook、MCP 服务器和 worker 共享一个数据目录，与真实安装一致，你自己的 `~/.claude-mem` 和默认 worker 端口绝不会被触碰。`claude plugin eval` 在用例目录内解析 `scaffold_script`，沿符号链接找到本 README 旁的脚本，并以运行的工作区为 cwd 执行它。

worker 的设置让评测只关注门禁：

- `CLAUDE_MEM_SKIP_TOOLS` 列出会话可调用的所有工具（包括 claude-mem 的 MCP 工具），因此没有运行会排队观察者工作。
- Chroma 关闭，语义注入关闭，遥测关闭，日志在 `data/logs/` 中为 DEBUG 级别。

真实 MCP 服务器运行（`--mocks off`）。运行器用 `--allow-tools` 授予 `mcp__plugin_claude-mem_mcp-search__*`、`Edit` 和 `ToolSearch`。授权适用于每个用例，因此问题用例也能看到 Edit，以及随 Edit 一起授予的 Write 和 NotebookEdit。它们不会被要求编辑。

在调用任何模型之前，预检检查四件事。第一是 D9 的前提条件，按构建后的 Hook 的求值方式。`plugin/scripts/worker-service.cjs` 解析到的 tree-sitter CLI 必须存在、能响应 `--version`，且是插件预置的副本。本机 PATH 上碰巧有的 `tree-sitter` 不算数；`npm run` 会把仓库的 `node_modules/.bin` 放上去。否则处于休眠状态的门禁会报告"tree-sitter CLI 未预置"，而不是缺少拒绝。其余三项把 PreToolUse Read 载荷通过 `node plugin/scripts/bun-runner.js plugin/scripts/worker-service.cjs hook claude-code file-context` 管道送入每个 worker：

- 门禁开启，整文件：Hook 以 `Full-file Read blocked by claude-mem` 标记拒绝，并列出播种的观察记录。
- 门禁开启，`offset: 40, limit: 20`：无决定，时间线作为上下文到达。
- 门禁关闭，整文件：无决定，时间线作为上下文到达。

它记录每次 Hook 调用的实际耗时。任何检查失败都会在调用模型前停止运行。

两个 worker 总会被停止，包括 Ctrl-C 时。`--keep-temp` 保留的沙箱在 trace 复制到报告后会被删除。

## tree-sitter CLI（D9）

门禁只在 `smart_outline` 能解析的地方拒绝。这需要 `tree-sitter` 可执行文件，由 tree-sitter-cli 的 `install.js` 下载（决策 D9，见 `plans/2026-10-05-file-read-gate-restore.md`）。`npm run build` 安装插件依赖时关闭了生命周期脚本，因此构建后 `plugin/node_modules/tree-sitter-cli` 没有可执行文件。构建后的 Hook 和构建后的 MCP 服务器都优先解析该副本，因此门禁会保持休眠，`smart_outline` 也无法解析。

在启动任何 worker 之前，运行器按真实安装的方式预置该副本。它调用 `src/services/smart-file-read/tree-sitter-cli-provision.ts` 中的 `ensureTreeSitterCliBinary(plugin/)`——安装器和 worker 使用的同一函数——该函数运行包的 `install.js`（从 tree-sitter 发布页下载，因此需要网络），除非可执行文件已能响应 `--version`；且仅当下载的 SHA-256 与 `tree-sitter-cli-checksums.ts` 中固定的摘要匹配时才安装。失败时运行器带着原因停止，并把版本记录到 `summary.json` 和摘要头部。

## 成本

每个评分器都是 `regex` 或 `tool_used`，因此不调用评判模型。开销来自 agent 运行：默认 15 次，5 个用例 x 3 次，模型为 `claude-sonnet-5-5`。`--max-cost-usd` 为其设上限，摘要报告评测的实际成本。

## 阅读 `reports/read-gate/<timestamp>/summary.md`

- **判定**：这些决定退出状态。门禁开启、大文件（`gate-on-large-file` 和 `gate-on-edits-file`）：Claude 尝试整文件 Read 并在至少 2/3 的运行中收到拒绝（否则从未尝试的运行会空洞地通过其他门禁开启判定，休眠的门禁也会如此）；没有运行读取整个文件；每次尝试整文件 Read 的运行都收到拒绝；编辑在至少 2/3 的运行中留下 `24.75` 且无 `22.40`。门禁开启、小文件：任何运行都无拒绝标记；至少 2/3 的运行中整文件 Read 放行（否则运行从不会向门禁提出拒绝大小所要回答的问题）；答案评分器在至少 2/3 的运行中通过。门禁关闭（两个文件）：任何运行都无拒绝标记；每次运行至少有一次成功读取该文件；小文件的答案评分器在至少 2/3 的运行中通过。大文件：每个分组的答案评分器在至少 2/3 的运行中通过；门禁开启收到拒绝、门禁关闭读取整个文件，各在至少 2/3 的运行中发生，否则比较毫无意义；且门禁开启每次运行成本低于门禁关闭（对所有报告了成本的运行取均值；当 `--case` 漏掉一个分组时不运行）。每次运行的 Read 按其自身用例的文件计数。当 `claude plugin eval` 以非零退出或提前停止，或请求的用例（全部用例，或 `--case` 选中的用例）在结果中的运行次数少于 `--runs` 要求，或有运行以错误结束或被 mock 中止时，运行器也以非零退出。**问题**会列出每个此类用例及原因。
- **运行**：每次运行一行。整文件 Read 显示为尝试 / 拒绝 / 返回文件。定向 Read 显示为成功 / 拒绝；被拒绝的定向 Read 会阻止 Edit。该行还给出对 `smart_outline`、`smart_unfold` 和 `get_observations` 的调用、是否出现拒绝标记、失败的评分器、轮数、token（缓存写入 / 缓存读取 / 输出，来自运行的 `result` 事件）和成本。
- **同一问题，门禁开启 vs 关闭**：对每个 fixture，给出其门禁开启和门禁关闭问题用例的平均轮数、token 和成本，以及门禁开启相对门禁关闭的成本变化。仅供参考，但大文件的比较也是判定项。
- **预检与 Hook 延迟**：先是 D9 的 tree-sitter 检查，然后每次预检 Hook 调用的实际耗时（含进程启动）。门禁开启：大文件的整文件 Read 被拒绝，其定向 Read 获得上下文，小文件的整文件 Read 获得上下文。门禁关闭：大文件的整文件 Read 获得上下文。仅供参考；所有检查必须通过评测才能运行。头部行给出插件使用的 tree-sitter 版本。

旁边还有：`summary.json`（相同数据，包含每次运行）、`preflight.json`、`traces/`（每次运行的 trace，以及编辑用例编辑后的文件）和 `eval/`（`claude plugin eval` 的 `aggregate-result.json` 和 `report.html`）。

## 最近一次运行

2026-10-06，Claude Code 2.1.291，tree-sitter 0.26.9，拒绝大小为 32 KB。两个模型上所有判定均通过。报告：`reports/read-gate/2026-10-06T09-54-38-735Z`（Sonnet）和 `reports/read-gate/2026-10-06T09-56-10-038Z`（Opus）。

**`claude-sonnet-5-5`，每用例 3 次运行，评测成本 $1.20**

| 用例 | 运行 | 整文件 Read 尝试 / 拒绝 / 返回文件 | 定向 Read | `smart_outline` / `smart_unfold` / `get_observations` | 通过的评分器 | 平均轮数 | 平均缓存写入 / 读取 | 平均成本 |
| --- | ---: | --- | ---: | --- | --- | ---: | --- | ---: |
| `gate-on-large-file` | 3 | 3 / 3 / 0 | 2 | 3 / 8 / 0 | `answer-rate` 3/3, `answer-minimum` 3/3, `read-blocked` 3/3 | 7.3 | 14,074 / 68,025 | $0.077 |
| `gate-off-large-file` | 3 | 3 / 0 / 3 | 3 | 0 / 0 / 0 | `answer-rate` 3/3, `answer-minimum` 3/3, `not-blocked` 3/3, `read-used` 3/3 | 3.0 | 35,231 / 58,591 | $0.124 |
| `gate-on-edits-file` | 3 | 3 / 3 / 0 | 3 | 0 / 0 / 0 | `edited` 3/3, `old-value-gone` 3/3, `read-blocked` 3/3 | 5.0 | 8,868 / 59,767 | $0.052 |
| `gate-on-small-file` | 3 | 3 / 0 / 3 | 0 | 0 / 0 / 0 | `answer-rate` 3/3, `answer-minimum` 3/3, `not-blocked` 3/3, `read-used` 3/3 | 2.0 | 17,682 / 20,389 | $0.073 |
| `gate-off-small-file` | 3 | 3 / 0 / 3 | 0 | 0 / 0 / 0 | `answer-rate` 3/3, `answer-minimum` 3/3, `not-blocked` 3/3, `read-used` 3/3 | 2.7 | 17,691 / 25,242 | $0.074 |

**`claude-opus-5-5`，每用例 2 次运行，评测成本 $1.59**

| 用例 | 运行 | 整文件 Read 尝试 / 拒绝 / 返回文件 | 定向 Read | `smart_outline` / `smart_unfold` / `get_observations` | 通过的评分器 | 平均轮数 | 平均缓存写入 / 读取 | 平均成本 |
| --- | ---: | --- | ---: | --- | --- | ---: | --- | ---: |
| `gate-on-large-file` | 2 | 2 / 2 / 0 | 6 | 2 / 0 / 0 | `answer-rate` 2/2, `answer-min