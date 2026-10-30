<!-- 镜像自 GitHub Issue #3610。Issue 是公开跟踪器，本文档是设计文档。
     若两者不一致，设计以本文档为准，状态以 Issue 为准（oh-my-issues 健康检查）。 -->

> **跟踪器：** https://github.com/thedotmack/claude-mem/issues/3610

# [plan-22] Chroma Sidecar 契约 — 宿主安全的启动方式，固定依赖与干净环境，单写入者 upsert 同步，诚实的回退（或退役 Sidecar）

> **跟踪器：** #3610 · **设计文档：** `plans/22-chroma-sidecar-contract.md` · **系列：** plan-14 #3602 · plan-15 #3603 · plan-16 #3604 · plan-17 #3605 · plan-18 #3606 · plan-19 #3607 · plan-20 #3608 · plan-21 #3609 · plan-22 #3610 · plan-23 #3611

## 缺陷

向量存储是一个 Python Sidecar（`uvx … chroma-mcp`），通过 MCP stdio 驱动，而 worker 与索引之间的每一层都未针对用户实际环境做过测试。结果是语义搜索在数周或数月内静默降级（FTS5 回退、自信的空结果、冻结的水位线），存储无界增长：

- **启动参数对宿主不安全：** `chroma-mcp==0.2.6` 作为裸位置参数（uvx < 0.5.31 会拒绝；需要 `@`/`--from`），`--with onnxruntime>=1.20 --with protobuf<7` 经由 `cmd.exe /c` 传递，其中 `>`/`<` 是重定向符，再经 spawn 层双重引号包裹，导致 uvx 看到字面引号；预热路径重新实现了 spawn 逻辑却没有传输层的 `.cmd` 处理；启动 shell 的 `VIRTUAL_ENV`/`PYTHONPATH` 泄漏到子进程，使 `numpy` 从外部 venv 解析；`onnxruntime>=1.20` 约束不可覆盖（macOS 12 / cp313 不匹配时无 wheel）；`chromadb` 未固定版本，一次引擎升级导致持久化存储段错误长达 2.5 个月且零日志信号；预检忽略 `CLAUDE_MEM_CHROMA_UVX_PATH`；连接超时（30 秒，后改为 120 秒预热）短于冷 `uvx` 物化时间，且在构建中途杀死 uvx 会留下 100+ GB 的 `uv` 临时目录并永远重跑冷构建。
- **同步没有写入契约：** `ChromaSync` 使用仅追加的 `chroma_add_documents`，仅在完整写入成功时推进水位线，因此部分失败会在每个周期重新添加相同文档（27k 行占 74 GB，约 260 倍重复）；损坏的 HNSW 段被无限热重试，伴随数 GB 的内存尖峰；两个 worker（两个配置目录、两个插件版本或一个 Codex 安装）写入同一 `--data-dir` 且无仲裁 → 157 GB 的 `link_lists.bin` 和内核 panic；写入者锁记录的是 worker 的 PID 而非子进程的；文档以守护进程的 cwd 作为 `project` 标记；CJK 内容破坏回填 JSON 帧格式。
- **回退说谎：** `executeWithFallback` 仅在存在 `platformSource` 过滤器时才回退到 SQLite，因此项目范围的查询在 Chroma 中未命中时（约 32% 的行未同步）返回自信的空结果并标记 `usedChroma: true`；连接后 0.6 秒即死亡（99.4% 的连接）或因 SIGSEGV 崩溃的子进程被当作慢速处理；`doctor`/`status` 保持绿色，而未索引积压增长到数千。

维护者已暗示 Chroma 可能被废弃（#3138 评论）。两种解决方案都能终结这一簇问题：**(a)** 下述契约，或 **(b)** 移除 Sidecar 改用进程内索引，此时本计划的测试矩阵变为"永不启动 chroma-mcp 进程且搜索对等性成立"。本计划按 (a) 编写；第 0 步是决策。

## 子 Issue

- #2879 — uv < 0.5.31 时 Chroma 语义搜索永不连接 — 裸 `uvx <pkg>==<ver>` 被拒绝 → MCP -32000
- #2897 — chroma-mcp 30 秒连接超时在安装中途杀死 uvx，导致无限重试循环泄漏约 111GB uv 临时文件
- #2939 — chroma-mcp== 版本固定破坏 uvx — 改用 @ 而非 ==
- #2954 — Windows：Chroma 启动将 `>`/`<` 依赖说明符经由 cmd.exe /c 传递，破坏语义搜索
- #2961 — 跟踪：chroma-mcp 启动失败并报 MCP -32000 — uvx 版本语法错误与 Windows cmd.exe 重定向篡改
- #2979 — 语义注入始终返回空：Chroma 同步与 /api/context/semantic 之间的项目过滤器不匹配
- #3012 — ChromaDB 失控增长至 157 GB；同一数据目录上双 chroma-mcp 写入者；卸载后自动重装
- #3121 — [Windows] chroma-mcp 启动立即失败 — `--with` 说明符的引号包裹经 cmd.exe 双重引号（认证部分 → plan-19）
- #3199 — onnxruntime>=1.20 固定使语义搜索在 macOS 12 / 较旧 Intel Mac 上不可用（无覆盖）
- #3202 — 损坏的 HNSW 段使 ChromaSync 变成无界重试循环 — 数 GB 内存尖峰与冻结的水位线
- #3361 — 空语义结果返回自信的空而非回退到 SQLite：回退以 platformSource 为条件，而非以空为条件
- #3362 — chroma-mcp 已固定但 chromadb 浮动：一次引擎升级导致持久化存储段错误约 2.5 个月且零信号
- #3423 — ChromaDB 回填对 CJK 观察记录报 "JSON Parse error: Unrecognized token"
- #3552 — [Windows] chroma-mcp 继承启动进程的 Python 环境，破坏 uvx 隔离 — 语义同步静默停止
- #3577 — Chroma/uvx 预检忽略 CLAUDE_MEM_CHROMA_UVX_PATH 覆盖
- #3579 — chroma-mcp 子进程连接后约 0.6 秒死亡（99.4% 的连接）— 永久静默索引停滞
- #3591 — ChromaSync 水位线在部分写入时冻结而仅追加写入已落地 — 相同文档每周期重新添加（约 27k 行占 74 GB）

需评估/变基的相关 PR：#2940、#2880（`--from`）、#2847（Windows 连接引号）、#3266（用 update 替代 delete+add，已发布）、#3192（内存看门狗 — 生命周期部分见 plan-14）、#3178。

## 修复顺序

0. **决策：契约还是退役。** 若退役 → 用移除 + 对等性矩阵替换步骤 1–5，并在不再存在 `chroma-mcp` 启动路径时关闭本 master。
1. **单一启动规格，不经 shell。** `ChromaSpawnSpec` 构建直接执行的 argv 数组（`uvx.exe`/`uvx` 只解析一次，遵循 `CLAUDE_MEM_CHROMA_UVX_PATH`，无 `cmd.exe`，无逐参数引号），`--from chroma-mcp==X --with chromadb==Y --with onnxruntime==Z --with protobuf==W` 全部精确固定并可通过设置覆盖（`CLAUDE_MEM_CHROMA_WITH`），净化环境（剥离 `VIRTUAL_ENV`、`PYTHONPATH`、`PYTHONHOME`、venv PATH 条目），`--data-dir` 来自解析后的数据目录；同一规格服务于连接、预热和预检。冷物化作为独立的、进度监控的 `uv tool install`/预热步骤运行，无墙钟杀死（改用停滞下载看门狗），因此 MCP 连接超时只衡量热启动。
2. **子进程健康是观察到的，而非假设的。** 捕获子进程的退出码/信号；SIGSEGV/立即死亡以 ERROR 级别抛出 `chroma_engine_crashed` 并附带固定版本，设置 `/api/health.chroma=down`，在 `doctor` 中显示崩溃计数器和未索引积压；连接后 N 秒内死亡的子进程打开带退避的熔断器，而非 1–3 分钟的重启节奏。
3. **单写入者，upsert 语义。** 写入者锁以*子进程*的 PID + 启动令牌为键（plan-15 存活规则），标记写入者版本以便旧 worker 拒绝写入；`chroma_upsert_documents`（或在每个添加路径上检查现有 ID）；水位线按文档推进；每个文档以 `resolveProjectIdentity()` 输出（plan-20）标记并测试 where 子句；JSON 帧格式对 CJK 的 UTF-8 安全。
4. **ChromaSync 中的失败分类。** 每个集合的确定性工具级失败（HNSW 应用错误）计入熔断器，停止重试、快照集合、重置水位线并从 SQLite 重建一次 — 若失败则冷却；大小合理性检查（chroma 字节 vs SQLite 行数，`link_lists.bin` ≪ `data_level0.bin`）在 `doctor` 中告警。
5. **诚实的回退。** `executeWithFallback` 在空结果（及不可用）时回退，无论过滤器如何，且每个结果携带 `strategy: chroma|sqlite` 及 `chromaCoverage`，使调用方和查看器能显示"语义搜索自 <日期> 起降级"。

## 测试矩阵

| 宿主 | uv / Python | 场景 | 要求行为 |
|---|---|---|---|
| macOS arm64、macOS 12 x86_64、Linux、Windows 10/11（uvx.exe、uvx.cmd shim、WinGet uv） | uv 0.5.7 / 0.5.29 / 0.11.x / 0.12.x；py 3.12 / 3.13 | 连接 | 子进程到达 `list` 探测；argv 被记录；无 cmd.exe |
| Windows | 任意 | 启动 shell 设置了 `VIRTUAL_ENV` | 子进程从 uvx 环境导入 numpy |
| 全部 | 任意 | 冷缓存 | 预热完成一次（受监控，不被杀死）；无超过 10 分钟的 `.tmp*` 构建目录 |
| 全部 | 任意 | chromadb 启动时崩溃（fixture） | ERROR 附带版本；`doctor` 红色；熔断器打开；SQLite 搜索仍可服务 |
| 全部 | 任意 | 部分批次失败 | 已写入文档的水位线推进；下一周期无重复（ID 计数断言） |
| 全部 | 任意 | 两个 worker，一个数据目录 | 第二个只读或拒绝；无双写入者 |
| 全部 | 任意 | 项目范围查询，行仅在 SQLite 中 | 返回结果且 `strategy=sqlite` |
| 全部 | 任意 | CJK 观察记录 | 回填成功 |
| 全部 | 任意 | （退役路径） | 无 `chroma-mcp` 进程；搜索对等性套件通过 |

矩阵位于 CI 中（`tests/services/sync/chroma-spawn-spec.test.ts` 配合假 `uvx`，加上每个 OS 上的真实 uvx 任务）。回归必须在用户提交前使 CI 失败。

## 范围之外

worker 死亡时回收 chroma 进程树并限制其内存（plan-14）。bundle 中缺失 `sqlite/` 模块（plan-16）。项目键派生本身（plan-20）。