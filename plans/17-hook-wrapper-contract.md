<!-- GitHub Issue #3605 的镜像。Issue 是公开跟踪器；本文档是设计。
     若两者不一致，设计以文档为准，状态以 Issue 为准（oh-my-issues 健康检查）。 -->

> **跟踪器：** https://github.com/thedotmack/claude-mem/issues/3605

# [plan-17] Hook 包装器契约 — 轻量、低成本、在所有宿主上失败开放

> **跟踪器：** #3605 · **设计文档：** `plans/17-hook-wrapper-contract.md` · **系列：** plan-14 #3602 · plan-15 #3603 · plan-16 #3604 · plan-17 #3605 · plan-18 #3606 · plan-19 #3607 · plan-20 #3608 · plan-21 #3609 · plan-22 #3610 · plan-23 #3611

## 缺陷

`hooks/hooks.json` 中的七个 Hook 条目（由 `src/build/hook-shell-template.ts` 生成）既不轻量也不失败开放。包装器的两个结构性特征导致了整个症状族：

**它在任何记忆代码运行之前就在 shell 中做昂贵且脆弱的工作。** 每次调用都会启动*第二个*登录 shell（`export PATH="$($SHELL -lc 'echo $PATH' 2>/dev/null):$PATH"`）——在 macOS/Linux 上耗时 250–420 ms，在 Git Bash 上耗时 1–10 s（其 profile 链会加载 nvm 等）；插件根目录发现是一个 `{ printf; ls -dt; printf; } | while read … break` 管道，其生产者在 `break` 关闭读端后仍在写入（MSYS 上出现 `printf: write error: Permission denied`，被 Claude Code 呈现为阻塞性 Hook 错误）；登录 shell 与 Hook 载荷共享 fd 0，因此 profile 脚本可以抽干 stdin（`[bun-runner] empty stdin payload received — issue #2188`，现在 darwin 上也出现）；嵌套的 `bash.exe` 登录 shell 正是 Windows 上每次工具调用弹出控制台窗口的原因，并使并发 MSYS fork 突发翻倍，可能导致 Git Bash 系统级卡死；当 Git Bash 不在 PATH 首位时，`"shell": "bash"` 会解析到 WSL 启动器；旧形态还在全部四个事件上运行 `worker-service start`。在 PostToolUse `*` 匹配器上，这项开销每天运行约 550–3,500 次。

**当 worker 不可达时，它阻塞用户而非降级。** 失败开放路径返回 `{continue:true, reason:"worker_unreachable"}`，但 `bC()` 先递增一个*全局、跨会话*计数器（`state/hook-failures.json`），超过 `CLAUDE_MEM_HOOK_FAIL_LOUD_THRESHOLD`（默认 3，使用 `>=` 所以永不解除触发）后写入 stderr 并 `exit 2`，Claude Code 将其视为 UserPromptSubmit 上的阻塞、PreToolUse:Read 上的拒绝，以及 Stop 上的重新唤醒循环；Stop 重入守卫（`stopHookActive`）只为 Codex 接线；`PreToolUse` 命令缺少 SessionStart 具有的 `|| echo '{"continue":true}'` 回退；`summarize` 在转录缺失时抛出未捕获异常。SessionStart 和 UserPromptSubmit 是同步的，`timeout: 60`，而内部预算（`HOOK_READINESS_WAIT` 10 s + `API_REQUEST` 30 s，Windows 上均 ×1.5）合计恰好 60 s，因此慢但存活的 worker 会导致 Hook 被杀而非返回回退；`HealthMonitor.httpRequestToWorker()` 的 `fetch()` 没有 `AbortSignal`，因此卡住的监听器会消耗整个预算；VS Code 扩展自身的 60 s 初始化截止时间仅被 SessionStart 就超过了。

修复方案是一份契约：包装器不解析任何可缓存的内容，不启动任何不需要的进程，从不写入已关闭的管道，从不触碰 stdin，从不显示窗口，并且**不能**因 worker 可用性故障返回阻塞性退出码。

## 子项

- #2823 — 每个 Hook 都报告 "Failed with non-blocking status code: No stderr output"；13.4.0 不可用（维护者报告已在 main 上通过 #2888/#2889 修复 — 待验证）
- #2846 — summarize 事件上 Hook 无限重试循环（Permission denied）
- #2903 — [Windows] VS Code 扩展的 60s 初始化截止时间被 claude-mem 缓慢的 worker 冷启动超过
- #2914 — Hook 错误（`printf: write error` + 更新后 worker 不可达）
- #2962 — 跟踪：npx-cli 包和 Hook 脚本中的 DEP0190 与 bash PATH 探测回归（PATH 探测部分；DEP0190 → plan-16）
- #2966 — 跟踪：worker 不可达导致 Stop Hook exit-2 → 无限重试循环阻塞所有 Claude Code 提示
- #2973 — 跟踪：Windows VS Code 扩展在 60s 超时 — SessionStart Hook 在冷 worker 启动上阻塞
- #3106 — 添加环境变量以禁用 PreToolUse / PostToolUse Hook（Windows 控制台窗口焦点窃取）
- #3161 — worker 不可达时 Stop Hook 退出 2，阻塞停止并导致无限唤醒循环
- #3167 — transcript_path 文件缺失时 Stop Hook 'summarize' 抛出未捕获异常
- #3169 — v13.3.0：Stop Hook 打印 'printf: write error'，退出 2（`.env` OAuth 令牌部分 → plan-19）
- #3186 — 每个 Hook 都启动登录 shell（$SHELL -lc）探测 PATH — 每个 Hook 事件约 250-300ms 开销
- #3190 — Hook 每次调用都启动登录 shell — 在 Windows（Git Bash）上每次工具调用增加约 5–10s
- #3224 — Windows/Git Bash：Hook 以误导性的 'printf: write error: Permission denied' 退出 2；worker 无法自愈
- #3246 — Bug Report 一直报错（UserPromptSubmit 被 Hook 阻塞）
- #3248 — Windows：claude-mem 导致系统迟缓 — 诊断报告（进程泄漏 + Hook 开销）
- #3267 — worker 可能未运行时 UserPromptSubmit Hook 间歇性阻塞命令
- #3303 — SessionStart/UserPromptSubmit Hook 同步（async=false，timeout=60）；Windows 内部预算等于线上超时
- #3348 — Windows：每个 Hook 的登录 shell 启动使并发 bash fork 翻倍 — 可能破坏整个 Git Bash 系统
- #3349 — [macOS] Hook 收到空 stdin（与 #2188 相同的故障，现在在 darwin 上）— 静默破坏所有记忆捕获
- #3396 — [Windows] 13.11.0 Hook 格式（"shell": "bash"）重新引入每次 Hook 控制台闪烁回归
- #3412 — 默认 WSL 发行版为 docker-desktop 时 Windows 上 bash Hook 静默失败
- #3426 — Windows/Git Bash：Hook 间歇性失败并报 'printf: write error: Permission denied'（管道竞争）
- #3434 — worker 饱和时 UserPromptSubmit 会话初始化 Hook 阻塞长达 60s
- #3449 — Windows：Hook 包装器在任何记忆工作前每次调用消耗约 1.8s
- #3451 — Windows：每个 Hook 仅为读取 $PATH 就启动登录 shell（约 420ms × 约 550 次/天）
- #3481 — UserPromptSubmit 被 claude-mem 自身的 stderr 写入阻塞（worker 不可达通知）
- #3507 — PostToolUse Hook 每次工具调用启动登录 shell（Windows/Git Bash 上约 400ms）
- #3521 — [Windows] claude-mem Hook 弹出可见控制台窗口（每次工具调用约 1 个）— 其他插件的 Hook 不会
- #3523 — UserPromptSubmit Hook 阻塞所有提示 — worker 在 244 个 Hook 中不可达
- #3553 — UserPromptSubmit Hook 长时间休眠后超时（Windows）— fetch() 无超时
- #3559 — claude-mem Hook（PowerShell 上每个 Hook 可见 shell 窗口）
- #3563 — UserPromptSubmit Hook 每次运行都超时（Windows）；全部四个 Hook 事件上冗余的 worker-service start 启动
- #3568 — worker 不可达时 PreToolUse:Read 退出非零，导致 Claude Code 在会话剩余时间内拒绝 Read 工具

待评估/变基的相关 PR：#3519（在 `command -v node` 上守卫前奏）、#2917（移除阻塞性预热 Hook，已合并？）、#2892（永不阻塞宿主）、#2885（每会话哨兵）、#2865（printf 触发，已合并）、#2888/#2889（EADDRINUSE 失败方退出 0）、#3099（限制 Hook 调用）、#2945、#2598、#2715、#3069、来自 @dajiaohuang 的有界 fetch PR（#3553）。

## 修复顺序

1. **前奏在常见路径上不做发现工作。** `command -v node >/dev/null 2>&1 || export PATH="$(…) </dev/null"` — 仅在 node 缺失时运行登录 shell，stdin 重定向自 `/dev/null`，stderr 静默；解析后的 PATH 和插件根缓存于 `$CLAUDE_MEM_DATA_DIR/resolved-env`（按插件版本失效），因此稳态下除 `node` 外不 fork 任何进程。插件根选择移入 `resolvePluginRoot()`（plan-16）；管道+`break` 生产者组替换为对数组的 `for` 循环（消费者退出后无写入者），每个剩余生产者都加 `2>/dev/null`。Windows 上模板显式解析 Git Bash（`CLAUDE_CODE_GIT_BASH_PATH` → `C:/Program Files/Git/bin/bash.exe`），检测 WSL 启动器并以 stderr 原因退出，且 `bun-runner.js`/`worker-service.cjs` 的每次启动（worker 启动、健康探测、分离守护进程）都设置 `windowsHide: true` 且不附加控制台 shell。
2. **可用性故障不能阻塞。** 在 Hook 处理器中引入类型化退出策略：`blocking_error`（exit 2）仅保留给来自宿主的畸形输入；`worker_unavailable`、`worker_wedged`、`setup_required`、`transcript_missing`、`data_dir_unwritable` 全部返回 `{continue:true, suppressOutput:true}` + exit 0，通知通过 `systemMessage`/`additionalContext` 传递（每会话一次，在 SessionStart 上），绝不通过 stderr+exit 2。每个 `hooks.json` 命令都以回退 echo 结尾。`stop_hook_active` 由 claude-code 适配器解析，使 Stop 无法重入。失败响亮计数器变为每会话、衰减，且只产生*消息*，绝不产生退出码。`CLAUDE_MEM_DISABLE_TOOL_HOOKS=1` / `CLAUDE_MEM_DISABLE=1` 在任何工作前短路。
3. **预算严格在线上超时之内。** SessionStart 为 `async: true`（单个非阻塞上下文 Hook，无 `worker-service start`）；UserPromptSubmit 保持同步（它创建下游 Hook 依赖的 `user_prompts` 行），但其就绪等待 + 请求预算上限为 10 s（`CLAUDE_MEM_SESSION_INIT_TIMEOUT_MS`），绝不乘以线上超时；每个 Hook 侧 `fetch` 和 `isPortInUse` 都携带 `AbortSignal.timeout()` ≤ 5 s；存活探测未命中（plan-15 权威）在 < 1 s 内短路到回退，而非等待冷启动窗口；PostToolUse 捕获在入队后即发即忘。
4. **stdin 只读取一次，防御性地。** `collectStdin()` 区分"不期望载荷"与"载荷丢失"，后者以 ERROR 级别记录父进程名；包装器中在 `bun-runner.js` 之前没有任何东西触碰 fd 0。
5. **Doctor 行。** `claude-mem doctor` 测量一次完整 Hook 往返（`SessionStart`、`PostToolUse`）并报告墙钟时间、fork 数以及控制台窗口抑制标志是否生效，使回归只需一条命令即可发现。

## 测试矩阵

| 宿主 | Shell | 场景 | 要求行为 |
|---|---|---|---|
| macOS / Linux | bash, zsh | node 在 PATH 上 | PostToolUse Hook 包装器开销 ≤ 60 ms；零登录 shell 启动（CI 中 strace/dtruss 断言） |
| Linux (distrobox) / macOS | bash，profile 读取 stdin | PostToolUse | 载荷完整；无 `empty stdin payload` |
| Windows 10/11 | Git Bash | 设置 `CLAUDE_PLUGIN_ROOT` | 无 `printf: write error`；包装器开销 ≤ 300 ms |
| Windows | Git Bash，PowerShell 宿主，Windows Terminal | 50 次工具调用 | 零可见控制台窗口（Windows 任务中 Win32 窗口枚举） |
| Windows | `bash` = WSL 启动器（docker-desktop） | 任意 Hook | exit 0 且 `{continue:true}`，一行 stderr 说明原因 |
| 全部 | 全部 | worker 已停止 | UserPromptSubmit / PreToolUse:Read / Stop 各自 exit 0 且 `{continue:true}`；提示被接受；Read 被允许；Stop 返回；通知在 SessionStart 上下文中出现一次 |
| 全部 | 全部 | worker 已绑定但卡住 | UserPromptSubmit 在 ≤ 10 s 内返回回退；PostToolUse 在 ≤ 5 s 内 |
| Windows | VS Code 扩展 | 无 worker 冷启动 | SessionStart 在 < 5 s 内完成；扩展初始化成功 |
| 全部 | 全部 | Stop 且转录文件缺失 | exit 0，不抛出，摘要跳过并记录 WARN |
| 全部 | 全部 | 连续 100 次 worker 不可达 Hook | 任何地方都无 exit 2；计数器永不阻塞 |

矩阵位于 CI 中（`tests/infrastructure/hook-wrapper-contract.test.ts` 在每个 shell 下运行真实的 `hooks.json` 命令 + 带窗口枚举的 Windows 任务）。回归必须在用户提交前使 CI 失败。

## 范围之外

判断 worker 是否存活并回收端口（plan-15）。包装器解析哪个插件根（plan-16 拥有 `resolvePluginRoot()`；本计划仅消费它）。Codex/OpenCode 宿主 Hook Schema（plan-23）。