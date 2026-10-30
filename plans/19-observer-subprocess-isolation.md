<!-- 镜像自 GitHub Issue #3607。Issue 是公开的跟踪器；本文档是设计文档。
     若两者不一致，设计以本文档为准，状态以 Issue 为准（oh-my-issues 健康检查）。 -->

> **跟踪器：** https://github.com/thedotmack/claude-mem/issues/3607

# [plan-19] 观察器子进程隔离 —— 为无头生成器显式指定 env、cwd、配置、工具集与认证来源，并让降级行为显式可见

> **跟踪器：** #3607 · **设计文档：** `plans/19-observer-subprocess-isolation.md` · **系列：** plan-14 #3602 · plan-15 #3603 · plan-16 #3604 · plan-17 #3605 · plan-18 #3606 · plan-19 #3607 · plan-20 #3608 · plan-21 #3609 · plan-22 #3610 · plan-23 #3611

## 缺陷

无头观察生成器是 worker 的一个 `claude` CLI 子进程，但它的启动方式却像是用户的交互式会话：它继承（或被拒绝）worker 碰巧继承到的一切，加载用户完整的 `~/.claude` 配置（包括第三方插件 Hook），在一个可写的 cwd 中运行，保留对等消息工具，只解析一次 CLI 二进制路径且此后不再重新解析，并从一条硬编码的凭据链中选取凭据，无视用户实际配置的内容。以上每一项都是独立的启动属性；没有任何一项被显式声明，且其中任何一项失败时，生成器返回的文本会被 plan-18 的分类器静默丢弃。

- **配置/Hook 继承：** SDK 子进程加载所有插件 Hook；第三方 SessionStart Hook 注入 "you MUST invoke the skill" 并劫持 INIT（空响应、零观察）。旧版打包中 `--setting-sources` 被发出但无值（`Invalid setting source: --permission-mode`）。
- **工具面：** `disallowedTools` 列出了 Bash/Read/Write……但没有 `SendMessage`；观察器会话曾向用户的主会话发送伪造指令。`CLAUDE_MEM_MAX_CONCURRENT_AGENTS=0` 无法表示（`parseInt(...) || 2`）。
- **cwd 与身份：** 在 Windows 上，`cross-spawn` 将 SDK 启动路由为 `cmd.exe /d /s /c "<argv joined>"`，因此转录文本中的 `>` 会在*用户的项目目录*中创建 0 字节文件（并被 `git add -A` 提交）；观察器转录落在 `~/.claude/projects/`，Claude Desktop 会把它们列为会话（共 452 个）；观察器启动的 SDK 会话会被 Hook 重新捕获，并归档到以插件版本目录（`13.12.4`）或插件缓存基名命名的项目下。
- **Env：** `sanitizeEnv()` 会剥离 `HTTP_PROXY`/`HTTPS_PROXY`（代理后返回 403）——自 13.8.0 起为有意行为；`~/.claude-mem/.env` 只能重新注入五个键，且不含 `CLAUDE_CODE_OAUTH_TOKEN`；*已过期*的钥匙串凭据会提前返回，永远不会回退到有效的 `CLAUDE_CODE_OAUTH_TOKEN` 环境变量；当血统源自桌面应用时钥匙串查找失败（搜索列表不含登录钥匙串），在 Linux 上 `.credentials.json` 令牌为空时同样失败。
- **二进制解析：** `findClaudeExecutable()` 在同一长生命周期进程中通过 `execFileSync` 探测；原生 CLI 自动更新后，缓存的 realpath 被解除链接 → 永久 ENOENT → `setup_required` 卡死直到手动重启；当 `claude` 仅作为 VS Code 扩展二进制安装时永远找不到 —— 3,063 次 PostToolUse Hook、0 条观察，唯一痕迹是日志文件中的一条 WARN。

## 子项

- #2976 — 跟踪：观察器 Hook 启动会累积无限 .jsonl 转录文件并混入跨会话上下文
- #2999 — ENV_PROXY_VARS 在 sanitizeEnv() 中被剥离而非保留
- #3074 — 无头观察生成器继承交互式插件 Hook → 第三方 SessionStart Hook 劫持生成
- #3105 — 订阅/CLI 认证：SDK 在干净安装上返回 'Failed to authenticate. API Error: 403'，仅限 worker 的实时会话
- #3290 — Claude Code CLI 自动更新后 worker 卡在 setup_required —— 观察静默停止直到手动重启
- #3317 — 观察生成器总是失败：--setting-sources 传参无值 → Claude CLI 退出码 1（旧版打包；残留问题：启动失败时记录子进程 stderr）
- #3357 — 以转录 token 命名的 0 字节文件被写入用户工作目录（未加引号的 `>` 到达 shell）
- #3406 — macOS：由 Claude Code 桌面应用启动的 worker 无法读取登录钥匙串 —— 每条观察被静默丢弃
- #3417 — 观察器转录累积在 ~/.claude/projects/ 并淹没 Claude Desktop 会话列表
- #3425 — 观察器 SDK 会话在以下项目名下自我捕获：以插件版本目录命名（"13.12.4"）
- #3566 — 观察器会话通过 SendMessage 向对等会话发送伪造指令
- #3582 — `claude` 不在 PATH 时静默无操作（VS Code 扩展安装）：3,063 次 PostToolUse Hook、0 条观察、无用户可见警告

同时承载 #3169 的认证部分（Windows 上 `.env` 的 `CLAUDE_CODE_OAUTH_TOKEN` 被忽略）和 #3121（过期令牌提前返回阻断环境变量回退），以及 #3424 的认证部分。

待评估/变基的相关 PR：#3291（`ClaudeExecutableUnspawnableError` + 有界自重启）、#2942、#2919（转录累积）、#2704（apiKeyHelper 等价物，在 plan-12 路线图上）。

## 修复顺序

1. **`ObserverSpawnSpec` —— 一个显式声明的对象。** `{binary, argv, cwd, env, settingSources: [], disallowedTools（含 SendMessage/对等工具）, permissionMode, sessionDir, authSource}` 在一处构建，并在每次启动时以 INFO 级别记录（脱敏）；SDK 子进程以 `settingSources: []`（或隔离的 `CLAUDE_CONFIG_DIR` shim）运行，从而不加载任何用户或插件 Hook；为 Hook 作者记录 `CLAUDE_MEM_INTERNAL=1`。
2. **worker 与子进程之间不经过 shell。** 以 argv 数组直接启动 `claude`（`windowsVerbatimArguments: false`，不走 `cross-spawn` 的 cmd.exe 路径）；观察内容通过 stdin/stream-json 传输，绝不作为参数；cwd 为 `~/.claude-mem/observer-sessions/<id>`（或 `--no-session-persistence` + claude-mem 自有的转录目录），从而不向用户项目写入任何内容，也不出现在 `~/.claude/projects/` 中；非零退出时将子进程 stderr 捕获到 worker 日志。
3. **自捕获防护。** 启动时携带观察器标记（env + cwd 位于插件/数据目录内）的会话会被 `session-init`/`observation`/`summarize` 处理器跳过；项目推导拒绝位于插件安装目录或数据目录内的 cwd。
4. **显式认证来源。** `authSource ∈ {env:CLAUDE_CODE_OAUTH_TOKEN, keychain, credentials.json, apiKey, apiKeyHelper}` 按文档化顺序解析，其中*可用的*环境变量令牌始终优先于已过期的平台凭据；`.env` 允许列表包含 `CLAUDE_CODE_OAUTH_TOKEN`；代理变量被保留（或 `CLAUDE_MEM_PRESERVE_PROXY` 默认 true）并记录 `NO_PROXY`；钥匙串查找检测桌面血统并回退到 `credentials.json`，同时检查令牌有效性；认证失败表现为 ERROR + `/api/health.auth=failed` + 一行 SessionStart 通知，并暂停（plan-18 保留路径）而非重新启动。
5. **可恢复的二进制解析。** `findClaudeExecutable()` 在 ENOENT 时探测 `realpathSync(candidate)`，搜索已知的编辑器扩展 glob，抛出类型化的不可启动错误以触发一次有界后继交接（plan-15），并将 `setup_required` 显示在 SessionStart 上下文和 `doctor` 中 —— 绝不只出现在日志文件里。
6. **设置解析。** `CLAUDE_MEM_MAX_CONCURRENT_AGENTS` 及其同类项使用 `??`/NaN 检查解析，使 `0` 表示"不启动生成器"（仅搜索模式）。

## 测试矩阵

| 宿主血统 | 认证 | 场景 | 要求行为 |
|---|---|---|---|
| 终端 / 桌面应用（macOS）/ VS Code 扩展 / Linux 桌面 | 订阅（钥匙串）、`.credentials.json`、环境变量令牌、API key、代理 | 启动 | 子进程在一次尝试内完成认证，或以 ERROR + 健康标志失败；绝不被归类为普通文本 |
| 全部 | 过期钥匙串 + 有效环境变量令牌 | 启动 | 使用环境变量令牌 |
| 全部 | 安装了 superpowers 风格的 SessionStart Hook | INIT | Hook 未加载；观察被存储 |
| Windows | 任意 | 转录文本包含 `> foo` | 项目 cwd 中创建零个文件 |
| 全部 | 任意 | 100 次观察器运行 | `~/.claude/projects/` 中零条目；转录位于 claude-mem 目录下，受保留策略约束 |
| 全部 | 任意 | 观察器启动 | 观察器自身无 `sdk_sessions` 行；无以版本目录命名的项目 |
| 全部 | 任意 | `claude` 二进制在运行中被替换/解除链接 | 在一次交接内恢复；无永久 `setup_required` |
| 全部 | 任意 | `claude` 仅在 VS Code 扩展目录中 | 找到，或 SessionStart 上下文显示 "claude-mem: setup required — claude CLI not found" |
| 全部 | 任意 | `MAX_CONCURRENT_AGENTS=0` | 不启动生成器；搜索正常工作 |
| 全部 | 任意 | 观察器带对等消息工具 | `SendMessage` 被阻止；无跨会话消息 |

该矩阵在 CI 中运行（`tests/services/worker/observer-spawn-spec.test.ts`，配合伪造的 `claude` 二进制 + 血统/env 夹具）。回归必须在用户提交 Issue 之前就让 CI 失败。

## 不在范围内

子进程返回内容的分类与历史预算（plan-18）。杀死/回收子进程（plan-14）。*用户*会话的项目命名（plan-20）。


---

## 轮次：2026-09-12 oh-my-issues

- **#4065** —— `CLAUDE_MEM_LLM_TIMEOUT_MS` 由 13.24.23 中的 #3794 修复引入，并在失败消息中被点名，但它只能从 `process.env` 读取。在 `~/.claude-mem/settings.json` 中设置它毫无效果，且是静默的：它不在设置的 `DEFAULTS` 中，而 `Settings.loadFromFile` 只让环境变量覆盖文件，从不把文件写入环境变量。其同类 `CLAUDE_MEM_API_TIMEOUT_MS` *在* `DEFAULTS` 中，且*通过*设置 getter 读取，因此两个相邻的超时使用两种不同机制，且只有一个是静默的。这与启动路径叠加：该值在模块导入时读取一次，而守护进程由 Claude Code 环境中的 PostToolUse Hook 重新启动，因此 shell 导出加 `worker-service.cjs restart` 不会生效。

  修复：(1) 将该键放入 `DEFAULTS`，与 `CLAUDE_MEM_API_TIMEOUT_MS` 共用同一 getter；(2) 每次启动时从生成器声明的配置中解析，而非在导入时解析一次；(3) 对 settings.json 中任何未知的 `CLAUDE_MEM_*` 键发出警告 —— 这一处加载器改动即可消除整类问题。(3) 与 plan-20 的装饰性配置发现重叠（`CLAUDE_MEM_CONTEXT_OBSERVATION_TYPES/_CONCEPTS` 从未到达 `loadContextConfig`）；哪个计划先交付设置注册表，就应由它为两者负责。