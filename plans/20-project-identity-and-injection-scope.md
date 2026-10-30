<!-- 镜像自 GitHub Issue #3608。Issue 是公开跟踪器；本文档是设计。
     若两者不一致，设计以本文档为准，状态以 Issue 为准（oh-my-issues 健康检查）。 -->

> **跟踪器：** https://github.com/thedotmack/claude-mem/issues/3608

# [plan-20] 项目身份与注入范围 — 捕获、同步和检索共用一个解析器，注入配置能到达查询层

> **跟踪器：** #3608 · **设计文档：** `plans/20-project-identity-and-injection-scope.md` · **系列：** plan-14 #3602 · plan-15 #3603 · plan-16 #3604 · plan-17 #3605 · plan-18 #3606 · plan-19 #3607 · plan-20 #3608 · plan-21 #3609 · plan-22 #3610 · plan-23 #3611

## 缺陷

"这是哪个项目？"在写入路径（Hook 捕获、观察器、Chroma 同步）、读取路径（SessionStart/compact 上下文、PreToolUse 文件上下文、语义注入、时间线）和迁移路径上各自独立计算，规则互不相同，且没有用户可用的覆盖方式。每一处不一致都会让数千条观察结果被静默地判为"此项目尚无记忆"，或让记忆散落在五个键下：

- 写入路径在打包的 Hook 脚本中是 `basename(process.cwd())`（子代理/SDK 会话继承 TMPDIR → 项目 `T`；观察器会话 → 插件版本目录）；读取路径自 #2663 起是 `basename(gitToplevel(cwd) ?? cwd)`，其无 git 时的回退逻辑与捕获所用的父项目归属不同；13.4.0 从 cwd-basename 切换到 git-root，导致 monorepo 子目录被重新键控且无迁移；子模块（`.git/modules/`）被当作独立的叶子项目；工作树靠**一次性**的 `.cwd-remap-applied-v1` 标记来调和，因此之后创建的工作树和重新安置的仓库会留下 `merged_into_project = NULL` 的孤立行；键用 SQLite BINARY 排序规则比较，所以跨机器时 `pasteypal` ≠ `PasteyPal`；不同仓库中的同名目录会冲突；`<project>:dream` 命名空间永远匹配不上 `WHERE project = ?`；Chroma 文档打的是*守护进程*的 cwd 标签，所以 `/api/context/semantic?project=` 永远为空。
- 注入配置形同虚设：`CLAUDE_MEM_CONTEXT_OBSERVATION_TYPES/_CONCEPTS` 有文档、会持久化，却永远到不了 `loadContextConfig`；`CLAUDE_MEM_EXCLUDED_PROJECTS` 被捕获处理器遵守，但 SessionStart 的 `context` 处理器不遵守；注入查询没有 `agent_id IS NULL` 过滤，子代理的步骤日志会把持久记忆挤出近期窗口。
- 注入输出不确定、不幂等：SessionStart 和文件上下文头部中的分钟级时间戳每 60 秒就会击穿 Anthropic 前缀缓存；文件上下文 Hook 在每次读取未变更文件时都会重新注入同一块内容（#1629 的每会话一次门控在 13.13.0 中消失了）；文件夹 `CLAUDE.md` 写入器仍会在原本没有 `CLAUDE.md` 的目录中创建 43 字节的存根；时间线锚定使用跨项目的全局 LIMIT，且从不选择流键。

修复方案是：一个 `resolveProjectIdentity(cwd, env)` 供所有路径使用（支持 `CLAUDE_MEM_PROJECT_NAME` / 标记文件 / git remote 覆盖，大小写折叠的规范键，工作树/子模块 → 父项目，由 `merged_into_project` 支撑的别名表）；一个由设置 + 模式构建一次、传给每个注入查询的 `ContextScope` 对象；以及作为 (scope, data) 纯函数的注入渲染器。

## 子项

- #2834 — 修复项目 dream 命名空间的蒸馏记忆上下文注入
- #2842 — 子模块 cwd 解析为空叶子项目 — 尽管父仓库有历史，仍显示"尚无记忆"
- #2864 — 工作树观察结果被一次性 `.cwd-remap-applied-v1` 迁移孤立；工作树删除后无法访问
- #2872 — 注入上下文中的易变分钟级时间戳破坏 Anthropic 提示缓存
- #2882 — 项目检测使用 git root 而非工作目录，导致 monorepo/子目录出问题
- #2967 — 跟踪：项目名检测在 monorepo、工作树和子模块中解析到错误目录
- #2970 — 跟踪：注入上下文中的分钟级时间戳每 60 秒击穿 Anthropic 前缀缓存
- #2971 — 跟踪：蒸馏的 dream 命名空间记忆被排除在上下文注入之外
- #3025 — 摘要时间线回溯：按流前瞻 + 合并源语义
- #3062 — 重新安置 / 父级 `git init` 会孤立移动前的观察结果，并把子项目坍缩成一个 git-root 键
- #3194 — 上下文注入在 git 仓库外将项目解析为 basename(cwd)，而捕获归属到父项目
- #3274 — 上下文注入不应包含子代理观察结果（过滤 agent_id IS NULL，可配置）
- #3343 — Cursor：不同项目共用一个工作区（按提示的文件夹作为项目 / 手动指定项目）
- #3409 — CLAUDE_MEM_CONTEXT_OBSERVATION_TYPES / _CONCEPTS 有文档且会持久化，却永远到不了注入查询
- #3437 — project 是 basename(process.cwd())，所以子代理/SDK 观察结果归档到 TMPDIR 下 — 本存储的 26% 归属错误
- #3480 — PreToolUse `file-context` Hook 在每次读取同一未变更文件时重新注入相同的"先前观察"块
- #3511 — SessionStart 上下文注入不遵守 EXCLUDED_PROJECTS
- #3531 — 项目身份是大小写敏感比较的 basename(git-toplevel) — 两个检出会静默分叉到不同的记忆桶
- #3544 — 仍会在没有 CLAUDE.md 的目录中创建仅 43 字节存根的 CLAUDE.md（v13.14.0）
- #3594 — 为同名目录的仓库提供自定义项目名/别名

待评估/变基的相关 PR：#2827（git-remote 身份来源）、#2883、#2856、#2665、#2858（dream 命名空间）、#2886（时间戳）、#2701（git-root 切换）、#3358。

## 修复顺序

1. **`src/utils/project-name.ts` 中的 `resolveProjectIdentity()`，处处使用。** 顺序：`CLAUDE_MEM_PROJECT_NAME` 环境变量 → 向上查找 `.claude-mem-project` 标记 → `CLAUDE_PROJECT_DIR` → git remote slug（`org/repo`，按 #2827 默认启用）→ git toplevel（工作树和子模块解析到其超级项目；子目录保留 `subpath` 字段以供可选范围限定）→ 最近的 `.claude/` 祖先 → 最后是 `basename(cwd)`，并对 TMPDIR/主目录/插件目录的 basename 设拒绝列表。输出是大小写折叠的规范键加显示名。打包的 Hook 脚本、观察器摄取、Chroma 同步元数据和每个读取路径都调用这一个函数。
2. **幂等调和，而非一次性标记。** Worker 启动时，一个带版本的调和器为存储 cwd（从此保留）解析结果不同的行重新推导键，写入 `merged_into_project`；`claude-mem project alias|merge <from> <into>` 命令暴露同一张表；检索以 `COLLATE NOCASE` 匹配 `project IN (key, aliases…)`。
3. **`ContextScope` 构建一次。** 由设置 ∩ 模式得到 `{projects, excludedProjects, types, concepts, mainAgentOnly (默认 true), subagentTypesAllowlist, platformSource}`；`SettingsDefaults` 补上缺失的键；每个注入查询（`ObservationCompiler`、语义注入、文件上下文、时间线）都接收一个 `ContextScope`；SessionStart 的 `context` 处理器应用 `excludedProjects` 和 dream 命名空间包含（存在时优先 `<project>:dream` 而非原始名）。
4. **确定性、幂等渲染。** 头部只打日期粒度（或什么都不打）；文件上下文门控按 (session, file, latestObservationEpoch) 持久化到 SQLite，使未变更文件只注入一次；文件夹 `CLAUDE.md` 写入器绝不创建原本不存在的文件，也绝不写入空包装；时间线查询选择流键，并按流前瞻应用显式的合并源语义。
5. **Doctor 行。** `claude-mem doctor` 分别从读取和写入代码路径打印当前 cwd 的解析身份，两者不一致则失败。

## 测试矩阵

| 布局 | 路径 | 必需行为 |
|---|---|---|
| 普通仓库、monorepo 子目录、git 工作树、子模块、`$HOME` 下的非 git 子目录、TMPDIR 子代理 cwd、插件缓存目录、两个大小写不同的检出、两个同名目录的仓库 + `CLAUDE_MEM_PROJECT_NAME`、Cursor 多根 | 捕获 vs SessionStart 上下文 vs 语义注入 vs 时间线 vs Chroma 元数据 | 每条路径上的规范键相同（由运行全部五个解析器的表驱动测试断言） |
| 安装后创建、随后删除的工作树 | 调和 | 删除前后都能从父项目访问到行 |
| 仓库在磁盘上移动 / 父级 `git init` | 调和 | 旧行被别名化，而非孤立 |
| 被排除的项目 | SessionStart、compact、文件上下文、语义 | 不注入任何内容 |
| `CONTEXT_OBSERVATION_TYPES=bugfix` | 注入 | 只有 bugfix 行 |
| 子代理密集的项目 | 注入 | 窗口默认零 `agent_id != NULL` 行 |
| 同一会话中读取同一文件 20 次 | 文件上下文 | 只注入一次（仅在有新观察后再次注入） |
| 没有 CLAUDE.md 的目录 | 任意会话 | 不创建 CLAUDE.md |
| 相隔 61 秒的两次 SessionStart 注入 | 字节 | 完全相同 |

该矩阵在 CI 中运行（`tests/utils/project-identity.test.ts` 配合 fixture 仓库 + `tests/services/context/context-scope.test.ts`）。回归必须在用户提交之前就让 CI 失败。

## 范围之外

观察器自捕获 / 观察器 cwd（plan-19）。超出打对项目标签之外的 Chroma 同步机制（plan-22）。超出步骤 2 所需范围的 `merged_into_project`/别名 Schema（plan-21）。


---

## 轮次：2026-09-12 oh-my-issues

- **#4061** — SessionStart 上下文 Hook 在 `resume` 时注入时间线，且在一次 `claude --resume` 启动中触发两次，相隔一分钟，各 50 条观察结果，注入到一个已带有该块的对话中。`hooks/hooks.json`（13.24.23）将其注册在 `SessionStart` 下，匹配器为 `startup|resume|clear|compact`。这是缺陷在会话层面的注入幂等性那一半，与文件上下文 Hook 重新注入未变更文件、分钟级时间戳击穿前缀缓存并列。修复应放在 `ContextScope` 中：让注入决策成为 (source, scope, 会话已携带内容) 的纯函数 — 从匹配器中移除 `resume`（resume 时 worker 的 `start` 保留）*并*让注入按启动幂等，因为双重触发表明 `startup` 和 `resume` 都匹配了一次启动。最终状态：`resume` → 零注入；`startup`/`clear`/`compact` → 恰好一次；每次启动绝不两次。