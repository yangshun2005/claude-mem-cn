<!-- 镜像自 GitHub Issue #3602。Issue 是公开跟踪器；本文档是设计文档。
     若两者不一致，设计以本文档为准，状态以 Issue 为准（oh-my-issues 健康检查）。 -->

> **跟踪器：** https://github.com/thedotmack/claude-mem/issues/3602

# [plan-14] 子进程所有权 — worker 派生的每个进程都随 worker 一同终止

> **跟踪器：** #3602 · **设计文档：** `plans/14-child-process-ownership.md` · **系列：** plan-14 #3602 · plan-15 #3603 · plan-16 #3604 · plan-17 #3605 · plan-18 #3606 · plan-19 #3607 · plan-20 #3608 · plan-21 #3609 · plan-22 #3610 · plan-23 #3611

## 缺陷

worker 守护进程会派生长期存活的子进程——chroma-mcp sidecar（`uvx → uv → python`）、无头 `claude` 观察者子进程、`mcp-server.cjs`，以及回收期间的兄弟 worker——但从不接管它们的生命周期。具体表现：

- 监督器注册表（`supervisor.json`）用单一固定名称（`"chroma-mcp"`）登记 sidecar；每次（重新）派生都会覆盖该条目**而不杀死先前的 PID**，因此 `pruneDeadEntries()` 始终只能看到一个存活子进程，而实际已累积数十个。
- 子进程通过 `StdioClientTransport` 派生，未设置 `detached`/进程组，`close()` 只向 `uv` 包装进程发送 SIGTERM，而 `uv` 不会转发给它的 python 子进程。在 systemd 主机上，孤儿进程被重新挂到 `systemd --user` 下，因此 `PPID==1` 的清扫找不到任何东西。
- 每条异常退出路径都跳过清理：10 秒的 `gracefulDeadlineMs` 在 `sessionManager.shutdownAll()` 执行中途到期，导致 `chromaMcpManager.stop()` 永远无法到达（#3459）；`performGracefulShutdown` 抛出 `Server is not running.`，`process.exit(0)` 在注册表清扫之前执行（#3153）；`EADDRINUSE` 的“拒绝启动重复实例”路径在后台初始化已派生一对 chroma 进程之后才退出（#3216）；重启超时后的 SIGKILL 跳过一切。
- `ChromaMcpManager.callTool()` 的重试在清理之后重新进入 `ensureConnected()`，没有关闭闩锁，会重新派生它刚刚杀死的进程树；`connectInternal()` 的 catch 吞掉 `transport.close()` 失败，并把唯一指向存活约 500 MB python 子进程的句柄置空。
- 没有任何机制按资源用量限制或回收子进程（单个 chroma-mcp 达到 23 GB 占用，在 RSS 中不可见），启动时也不清扫孤儿进程。

由于版本不匹配回收循环（plan-16）每小时可重启 worker 数百次，这一泄漏一天内会变成数百个进程和数十 GB 交换空间。修复循环只能止住泵；只有所有权才能止住泄漏。

## 子 Issue

- #2896 — chroma-mcp 子进程泄漏：重连循环导致进程无界累积
- #2950 — macOS 上 chroma-mcp 子进程泄漏——僵尸 uv+python3.13 进程对跨会话累积
- #2959 — 跟踪：chroma-mcp 子进程泄漏——静默杀死失败和孤儿套接字继承导致僵尸 uv+python 进程对累积
- #3149 — 每个会话都泄漏 chroma-mcp 子进程（从不回收）+ chroma-mcp 启动忽略 CLAUDE_MEM_DATA_DIR
- #3153 — worker 每次重启泄漏一对 chroma-mcp 进程——supervisor.json 被覆盖，先前的 chroma 从不回收
- #3175 — chroma-mcp 进程在会话退出时成为孤儿 → 累积、持有 uv 缓存锁、膨胀 ~/.cache/uv
- #3191 — 进程内内存泄漏：单个存活的 chroma-mcp 子进程增长到 23 GB 占用——worker 应限制并回收它
- #3205 — chroma-mcp 进程泄漏：过期的 marketplace 版本触发无界 worker 回收循环，被回收的 worker 将 chroma-mcp 变成 PID 1 的孤儿
- #3216 — 过期版本回收循环泄漏 chroma-mcp 进程（759 个进程 / ~71 GB RSS），直到机器 OOM
- #3218 — 13.10.3 上仍存在 chroma-mcp 孤儿泄漏——约 20 分钟使 8GB 机器 OOM（#1063 / #1077 的回归）
- #3230 — chroma-mcp 子进程泄漏且从不回收——一天后 1000+ 孤儿进程 / ~48 GB RAM
- #3270 — 进程泄漏：一个会话期间累积 58+ 个 claude-mem 进程
- #3301 — 跨 worker 世代的 Chroma-mcp 孤儿泄漏：存活孤儿的注册表条目被覆盖而未杀死
- #3326 — chroma-mcp 服务器每 1-3 分钟泄漏一对新进程，从不回收旧的
- #3382 — 孤儿 chroma-mcp 进程累积——监督器跟踪 1 个，数百个泄漏
- #3404 — worker 守护进程从不回收无头 claude 观察者子进程——4 天累积 203 个
- #3413 — chroma-mcp 孤儿跨越 5 个 uv 构建世代（监督器跟踪 23 个中的 1 个）；孤儿被换出，因此泄漏是静默的
- #3459 — 优雅关闭也泄漏 chroma-mcp：10 秒期限在排空中途到期，因此 chromaMcpManager.stop() 永远无法到达
- #3540 — [Windows] chroma-mcp 重新派生清理与孤儿 uv builds-v0 临时目录相关——已确认 144GB

待评估/变基的相关 PR：#3302（覆盖前杀死）、#3465（关闭回收兜底 + 终态闩锁）、#3462（connectionGeneration 守卫）、#3192（内存看门狗）、#3232（按 PID 的注册表键 + 孤儿清扫）、#3405（注册表验证回收）、#2536、#2920、#1175。

## 修复顺序

1. **注册表按 PID 而非角色键控。** `ProcessRegistry.register()` 每次派生记录 `{pid, pgid, startToken, cmdlineFingerprint, dataDir}`；在任何新的 chroma/观察者派生之前，杀死记录的前驱进程树（通过 cmdline 指纹验证），然后注销。覆盖而不杀死在构造上变得不可能。
2. **每个子进程在其自己的进程组中派生**（POSIX 上 `detached: true` + `unref()`；Windows 上 Job Object / `CREATE_NEW_PROCESS_GROUP` + `taskkill /T`），使 `kill(-pgid)` 能到达 `python`，而不仅是 `uv`。Windows 杀死在 `/F` 之前使用优雅步骤，以便 uv 能清理 `builds-v0/.tmp*`（#3540）。
3. **单一清理路径始终执行。** `runShutdownSequence` 获得有界回收兜底：`process.on('exit'|'SIGTERM'|'SIGINT'|'uncaughtException')` → 注册表级联杀死，自带期限；优雅顺序变为 `chroma stop → observer abort → http close → db`，使昂贵的 `shutdownAll()` 不再能饿死子进程清理；`ERR_SERVER_NOT_RUNNING` 和 `EADDRINUSE` 重复路径都走同一级联。
4. **关闭闩锁。** `chromaMcpManager.stop({terminal:true})` 设置闩锁；之后 `ensureConnected()`/`callTool()` 抛出 `ChromaUnavailableError`。`stop()` 被记忆化。`connectInternal()` 失败时在置空 transport 之前杀死已派生的 PID。
5. **启动时和周期性清扫，独立于注册表。** worker 启动时及每个监督器 tick：杀死任何 `chroma-mcp --data-dir <ours>` / `claude … --resume <observer-id>` / `mcp-server.cjs` 进程，其父进程不是存活的 claude-mem worker（按 cmdline + data-dir 遍历，而非 `PPID==1`）。
6. **每个子进程的资源上限。** 每 5 分钟汇总子进程树的占用（PSS/私有，而非 RSS）；超过 `CLAUDE_MEM_CHROMA_MEMORY_LIMIT_MB`（默认 2048）时通过同一“派生前杀死”路径回收。同一看门狗中止空闲超过期限的观察者子进程。
7. **Chroma 启动遵循 `CLAUDE_MEM_DATA_DIR`** 作为 `--data-dir`（#3149），使多个配置目录不会意外共享一个存储。

## 测试矩阵

| 主机 | 触发 | 必需行为 |
|---|---|---|
| macOS / Linux（PID 1）/ Linux（systemd --user subreaper）/ Windows（Bun） | `worker restart` ×20 | 每次之后恰好一个 `chroma-mcp` 进程树存活；`supervisor.json` 有一个存活的 chroma 条目 |
| 全部 | SIGKILL worker，重启 | 启动清扫在新 worker 健康后 5 秒内杀死孤儿进程树 |
| 全部 | 优雅关闭，有 40 秒待处理的 summarize 工作 | chroma 进程树在进程退出前消失；`stop()` 之后不再派生 |
| 全部 | `EADDRINUSE` 重复启动 | 重复实例退出时零子进程残留 |
| 全部 | chroma 子进程占用 > 限制 | 回收一次，不循环；旧进程树消失 |
| 全部 | 4 小时空闲 worker | 观察者子进程数 == 活跃会话数 |
| Windows | `taskkill` 路径 | 没有超过 10 分钟的新的 `%LOCALAPPDATA%\uv\cache\builds-v0\.tmp*` 目录 |

该矩阵位于 CI 中（`tests/infrastructure/process-ownership.test.ts` + 一个 Docker `systemd --user` 作业 + Windows 作业）。回归必须在用户提交之前使 CI 失败。

## 范围之外

幽灵 LISTEN 套接字、过期 PID/spawn/writer 锁及回收（plan-15）。驱动泄漏的版本不匹配回收循环和过期路径后继派生（plan-16）。Chroma 派生参数、版本固定和同步水位（plan-22）。