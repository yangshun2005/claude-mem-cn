<!-- GitHub Issue #3604 的镜像。Issue 是公开跟踪器；本文档是设计文档。
     若两者不一致，设计以本文档为准，状态以 Issue 为准（oh-my-issues 健康检查）。 -->

> **跟踪器：** https://github.com/thedotmack/claude-mem/issues/3604

# [plan-16] 规范安装标识与包完整性 — 一个可解析、依赖完整的插件根目录，以及不会杀死正常 worker 的回收机制

> **跟踪器：** #3604 · **设计文档：** `plans/16-canonical-install-identity-and-bundle-integrity.md` · **系列：** plan-14 #3602 · plan-15 #3603 · plan-16 #3604 · plan-17 #3605 · plan-18 #3606 · plan-19 #3607 · plan-20 #3608 · plan-21 #3609 · plan-22 #3610 · plan-23 #3611

## 缺陷

"安装并运行的是哪一份 claude-mem"没有唯一答案。Hook、supervisor 的后继进程 spawn、`npx claude-mem`（install/repair/doctor/status）、SDK 子进程、marketplace 自动更新器各自独立解析插件根目录，且都不校验该副本是否具备运行时依赖：

- Hook 前奏在 `~/.claude/plugins/cache/thedotmack/claude-mem/[0-9]*/` 下按 **mtime** 取最新（`ls -dt … | head -1`），会接受 1 字节的空壳桩和带 `.orphaned_at` 的目录，并回退到 `~/.claude/plugins/marketplaces/thedotmack/plugin/` —— 一个永远没有 `node_modules` 的检出目录。`resolveWorkerScriptPath()`/`resolveSuccessorScript()` 在代码中重复同样的 mtime 排序（hook 侧已由 #3371 在 13.12.x 修复；后继解析中仍存在，见 #3216/#3565）。
- `worker-service.cjs` 将 `zod`、`shell-quote`、`tree-sitter-*`、`ajv*`、`better-auth` 以及（13.9.x）`../sqlite/{SessionStore,observations/files}.js` 外部化；包需要构建从未产出的模块，以及插件更新 / marketplace 拉取从未安装的 `node_modules`（`Cannot find module 'zod/v3'`、`Cannot find module '../sqlite/observations/files.js'`）。`tree-sitter-cli` 的二进制由安装脚本获取，而该脚本被 `--ignore-scripts` 抑制；Windows 解析器检查的是无扩展名路径；语法声明了冲突的 `tree-sitter` peer 主版本（`ERESOLVE`）；npm 12 拒绝安装器传入的 `--allow-scripts` CLI 标志。
- 版本不匹配回收将 *marketplace 清单* 版本与 worker 的 `/api/health` 版本比较，并在后继存在**之前**杀死现任，然后从解析器返回的任意路径惰性 spawn —— 通常是 marketplace 副本或较旧的缓存目录 —— 并等待一个全新 worker 在清单过期时永远无法报告的插件版本匹配。结果：无界回收循环（86–449 次/小时），喂养 plan-14 的泄漏和 plan-15 的幽灵端口，且健康 worker 被一个注定失败的 worker 替换。
- `doctor` 检查 marketplace 的 `node_modules`，`repair` 安装到版本缓存，`status` 在健康 worker 旁边报告"未安装"；`.orphaned_at` 被 hook shell 遵守但运行中的守护进程不遵守；过期缓存版本从不 GC；marketplace 没有根 `.claude-plugin/marketplace.json`。
- 发布的包集合未经审计：DEP0190（`args` + `shell:true`）在 `dist/npx-cli` 中存活，`shell-quote` 和 sync-hub 传递依赖带有已知 CVE，一个 agent 指令模板随 `plugin/modes/` 发布，Windows CI 因与被测 PR 无关的原因而红。

## 子项

- #2831 — worker-service.cjs 有外部 zod require — node_modules 缺失时崩溃
- #2910 — smart_search/smart_outline/smart_unfold 在干净的 Windows 安装上返回 0 个符号 — tree-sitter-cli 二进制从未下载（--ignore-scripts），无扩展名 existsSync
- #2922 — 共享 Windows CI 噪音（托管 runner 上的 tree-sitter-css node-gyp）
- #2941 — DEP0190 在 13.6.1 上仍可复现：npx-cli 包中的 args + shell
- #2952 — shell-quote 1.8.3 有 CVE-2026-9277 — 建议升级
- #2964 — 跟踪：分发包中缺少插件运行时依赖 — zod 外部化、tree-sitter-cli 未下载、hook 崩溃
- #2974 — 跟踪：共享 Windows CI 噪音 — GitHub 托管 runner 上 tree-sitter-css node-gyp 重建失败
- #3004 — marketplace.json 位置与 Claude Code 插件发现不兼容
- #3035 — ERESOLVE peer 依赖冲突：安装时 tree-sitter 版本不匹配
- #3054 — 添加 CLAUDE_MEM_WORKER_SCRIPT_PATH 以固定 worker 包
- #3056 — SettingsDefaultsManager loadFromFile 测试在运行 claude-mem 的开发机上失败（环境变量泄漏）
- #3072 — 原生 tree-sitter 语法在 Windows 上的 bun 下失败 — worker 总是用 bun 启动，其 Node-API 不完整
- #3091 — v13.9.1：observation → Chroma 向量同步对每个 observation 都失败（缺少 `../sqlite/observations/files.js`）
- #3100 — [Windows] 版本不匹配 worker 自杀 + 端口僵尸链（13.9.x）
- #3107 — Chroma 向量同步在 13.9.x 上损坏：worker-service.cjs 需要构建从未产出的 ../sqlite/*.js
- #3117 — doctor 必需的 'Marketplace deps' 检查无法被其自身提示治愈 — repair 只安装到版本缓存
- #3126 — Backfill 因缺少模块 '../sqlite/observations/files.js' 崩溃；搜索查询参数被忽略
- #3134 — Linux：worker 路径在不同调用上下文中解析到不同的磁盘副本 → 两个并发守护进程；过期缓存版本从不 GC
- #3152 — smart_search/smart_outline 对所有语言损坏 — tree-sitter-cli 二进制从未安装
- #3155 — hooks.json 插件根解析可能在自动更新窗口期间选中空壳桩或回退到 marketplace 副本
- #3253 — 全新安装 claude-mem @ 13.11.0 抛出 "allowScripts" 错误
- #3392 — CI：'typecheck · build · test · bundle-size' 作业不稳定（顺序依赖的 mock、CORS 端口冲突）
- #3424 — 全新安装损坏：加载时 marketplace 缓存未命中，然后 worker 压缩 spawn 的订阅认证失败
- #3438 — 将"发布文件中无 agent 指令"规则结构化 — law-study-CLAUDE.md 仍在发布
- #3456 — .orphaned_at 仅是 hook 过滤器 — 运行中的守护进程和 chroma-mcp 继续运行，使手动禁用路径失效
- #3478 — osv-scanner：workers/sync-hub/bun.lock 中 2 个高危传递漏洞
- #3534 — observations 从不压缩；`doctor`/`status` 在成功安装后立即报告"未安装"
- #3545 — 插件更新不安装声明的依赖 — worker 失败于 Cannot find module 'zod/v3'
- #3565 — 版本不匹配回收从 `marketplaces/…/plugin/` 重新 spawn worker，该目录永远没有 `node_modules` — 必然 spawn 失败

待评估/变基的相关 PR：#3371（semver 解析器，已合并）、#3095（tree-sitter 运行时供应，仅社区边缘）、#3146（内联 `parseFileList`）、#2887（打包 zod，仅社区边缘）、#3055（`CLAUDE_MEM_WORKER_SCRIPT_PATH`）、#2918、#2710、#2597、#2531、#3066、#2945、#2598、#1677、#2699。

## 修复顺序

1. **一个解析器，三个消费者。** `resolvePluginRoot()`（共享 TS，经 `hook-shell-template.ts`、`worker-utils.ts` 和 `npx-cli` 编译进 hook 前奏）返回满足以下条件的最高 **semver** 缓存目录：(a) 有超过大小下限的 `worker-service.cjs`，(b) 无 `.orphaned_at`，(c) 有 `node_modules/zod/v3`（或包的外部清单所列的任何内容），优先遵守 `CLAUDE_MEM_WORKER_SCRIPT_PATH`。marketplace 检出目录**永远**不是 spawn 目标。存在 `installed_plugins.json` 的 `installPath` 时优先使用。
2. **包不欠它看不见的文件系统任何东西。** 内联所有纯 JS 外部依赖（zod、shell-quote、ajv、`sqlite/observations/files`）；为任何必须保持外部的模块发布 `plugin/sqlite/`；添加构建时守卫，扫描 `plugin/scripts/*.cjs` 中的 `require("../…")`/外部 id，除非目标在 tarball 中否则失败。CI 中断言 `npm pack --dry-run` 差异。
3. **依赖随副本落地。** 插件更新 / 首次 hook / `repair` 在版本目录内运行 `bun install --production`（或 vendored `node_modules`），成功**之后**写入 `.install-version` 标记；tree-sitter-cli 二进制由显式的非生命周期步骤获取，Windows 上解析 `.exe`；语法对齐到单一 `tree-sitter` 主版本或移到可选步骤之后；`allowScripts` 在 `package.json`/`.npmrc` 中声明，绝不用 CLI 标志。新版本目录原子发布（临时 + 重命名）。
4. **只回收进已验证的后继。** 版本比较是针对*解析出的已安装根目录*的 semver 严格更新，而非 marketplace 清单；后继先在探测端口上 spawn，必须用自己的 token 应答 `/api/health`，然后要求现任交接（plan-15 交接）；失败则保留现任并开启每次启动的熔断器（最多 3 次回收/小时，记录日志）。`.orphaned_at` 由守护进程在启动时和每个 supervisor tick 检查 → 优雅关闭。
5. **doctor / status / repair 一致。** 三者都使用 `resolvePluginRoot()`；`doctor` 报告解析出的根目录、其版本及其依赖完整性；`repair` 修复*那个*根目录；仅当解析器返回空时才报告"未安装"。Marketplace 注册在宿主发现的根目录写入 `.claude-plugin/marketplace.json`。
6. **GC 与审计。** 升级成功时，删除早于前一版本的缓存目录（保留 N=2）；CI 对所有 lockfile 运行 `osv-scanner`，对 `dist/` 运行 `check-spawn-env-discipline.cjs`（DEP0190），以及打包 lint，对白名单外被跟踪的 `*CLAUDE.md` 失败。Windows CI 固定到工具链可用的 runner（或跳过原生语法构建）以使其可信；不稳定的 mock/端口测试（#3392、#3056）被隔离，以便矩阵可以以绿色为门禁。

## 测试矩阵

| 宿主 | 安装路径 | 场景 | 必需行为 |
|---|---|---|---|
| macOS / Linux / Windows | marketplace | 会话中途 `claude plugin update` N → N+1 | hook 仅在其依赖存在后才解析 N+1；现任 N 继续服务直到后继健康；交接后记录零次回收 |
| 全部 | marketplace | 清单版本 ≠ 缓存版本（过期清单） | 不回收；`doctor` 解释偏差 |
| 全部 | marketplace | 带 `.orphaned_at` / 1 字节桩 / 无 `node_modules` 的缓存目录 | 永不被选中；来自孤立目录的守护进程自行关闭 |
| 全部 | `npx claude-mem install`（npm 10/11/12，Node 20/22/24/26） | 全新安装 | 退出码 0；`doctor` 绿色；`smart_search` 在 `.ts` 文件上返回符号 |
| Windows | Bun worker | tree-sitter 语法 | 要么在 Node 下加载，要么按语言降级并给出 WARN，绝不静默 `0 symbols` |
| 全部 | tarball | `npm pack` | `plugin/scripts/*.cjs` 中每个 `require()` 目标都在 tarball 内可解析 |
| 全部 | 仓库 | CI | osv-scanner 干净、DEP0190 lint 干净、打包 lint 干净、Windows 作业在空 PR 上绿色 |

矩阵位于 CI 中（`tests/infrastructure/plugin-distribution.test.ts`、按 Node/npm 版本的全新安装 Docker 作业，以及 Windows 作业）。回归必须在用户提交之前使 CI 失败。

## 范围之外

被杀死现任的子进程会发生什么（plan-14），以及端口如何交接/回收（plan-15）。Hook shell 前奏开销和 fail-open 语义（plan-17）。