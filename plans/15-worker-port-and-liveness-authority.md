<!-- GitHub Issue #3603 的镜像。Issue 是公开跟踪器；本文档是设计文档。
     若两者不一致，设计以文档为准，状态以 Issue 为准（oh-my-issues 健康检查）。 -->

> **跟踪器：** https://github.com/thedotmack/claude-mem/issues/3603

# [plan-15] Worker 端口与存活判定权威 —— 对"是否有 worker 在服务此端口？"给出唯一经过验证的答案

> **跟踪器：** #3603 · **设计文档：** `plans/15-worker-port-and-liveness-authority.md` · **系列：** plan-14 #3602 · plan-15 #3603 · plan-16 #3604 · plan-17 #3605 · plan-18 #3606 · plan-19 #3607 · plan-20 #3608 · plan-21 #3609 · plan-22 #3610 · plan-23 #3611

## 缺陷

"worker 是否存活？"由五个相互独立、彼此矛盾的信号回答 —— `worker.pid`、`spawn.lock`、`chroma/.claude-mem-chroma-writer.lock`、`supervisor.json`，以及"端口是否被占用？" —— 而它们没有一个与实际持有监听套接字的进程核对过。因此：

- 监听套接字句柄可继承；chroma-mcp 进程树（以及在 Windows 上 Bun 下由 Hook 派生的 worker 子进程）在 worker 死亡后仍保持 `:37777` 绑定。内核在幽灵监听器上完成 TCP 握手，于是探测看到"端口被占用"，客户端要等满整个 `CLAUDE_MEM_API_TIMEOUT_MS`。
- 每个启动器都把"端口被占用"当作 worker 存活的证据（`Port already in use, refusing to start duplicate`、`Worker PID file points to a live process, skipping duplicate spawn`、`Another launcher holds the spawn lock`），且从不回收：`spawn.lock` 没有持有者存活检查，`validateWorkerPidFile` 会删除 pid 文件，导致 `ensureWorkerRunning` 再也无法识别所有者，`EADDRINUSE` 分支直接 `exit(1)` 且无补救，`pidFileStatus === 'alive'` 永远返回 `warming`。
- 健康探测没有超时（`HealthMonitor.httpRequestToWorker()` 的 `fetch()` 没有 `AbortSignal`，`isPortInUse()` 同样），因此一个卡死但仍绑定的 worker（DarkWake 100% CPU、CLI 探测挂起、写入锁过期、数据目录不可写）会让每个 Hook 停顿 30–60 秒，让每个启动器堆积（296+ 个 Hook 客户端，16+ 个 bun.exe）。
- 恢复路径本身也在说谎：`startWithBun` 预写 PID 文件并对引导 PID 做健康检查（守护进程健康时却报 `Process died during startup`），优雅关闭失败后报 `Restart successor spawned {pid=0}`，健康超时后仍记录 `Worker available {}`，冷却标记（`.worker-start-attempted`，120 秒）挡住了本可成功的重试，而在 Windows 上守护进程可能在启动一秒后以 0 退出，既无标记也无日志。
- 守护进程永远写入其启动日期的日志文件，而 `/api/logs` 读取今天的文件，因此数周此类故障都不可见。

架构层面的修复是设立单一的存活判定权威：一份带启动令牌的所有者记录，健康端点回显该令牌；"存活"意味着*该*令牌在有界探测内应答了 `/api/health`；绑定到端口的其他任何东西都可回收。

## 子 Issue

- #2891 — claude-mem worker 不可达 —— 会话开始后每次工具调用 Hook 都静默失败
- #2893 — [Windows] 僵尸端口清理器（#1721）漏掉持有继承套接字的孤儿进程
- #2899 — [Windows] 插件更新使 worker 端口成为孤儿（幽灵 LISTENING 套接字），阻塞所有 Claude Code 提示
- #2907 — chroma-mcp 子进程在 Hook 父进程退出后占用端口，阻塞重新派生
- #2926 — Windows 上 worker 从不启动 —— 端口 37777 被孤儿进程占用；UserPromptSubmit Hook 循环失败
- #2963 — 跟踪：Windows worker 端口在崩溃/更新后成为孤儿 —— 幽灵 LISTEN 套接字阻塞重新派生，Hook 循环失败
- #2992 — [Windows] Hook 在 29 秒后超时 —— localhost 先解析到 ::1，但 worker 只监听 127.0.0.1
- #2996 — Windows：过期的端口持有者 + 激进的派生冷却使提示阻塞约 15 分钟
- #3031 — Windows：版本升级时 worker 回收失败，留下僵尸进程占用端口 37777；过期的 supervisor.json PID 阻塞每次重新派生
- #3052 — Linux/WSL2：worker-cli.js start 误报"Process died during startup" —— 预写 PID 后自我退出问题仍存在
- #3073 — [Windows] 常驻 worker 守护进程没有关闭钩子 —— 阻塞更新/重启；手动停止留下幽灵 LISTEN 套接字
- #3085 — MCP STDIO 连接在 59 秒后关闭，使 claude-mem 不可用
- #3109 — Stop Hook 没有耗时日志 —— 根因：过期的 spawn.lock（持有者已死）+ Windows 上守护进程 stdout 被吞
- #3128 — Windows：版本不匹配回收使继承监听套接字的子进程成为孤儿 —— 端口在死 PID 下持续绑定数小时
- #3171 — worker 可能在持有端口时卡死；已接受的套接字泄漏进 CLOSE_WAIT；对端会话重试风暴
- #3204 — Windows：worker 重启留下孤儿 LISTENING 套接字，永久阻塞 worker 端口 —— 每个 Hook 挂起 60–120 秒
- #3300 — worker 守护进程将监听套接字泄漏给子 worker → 端口永不释放 → 无限懒派生抖动
- #3340 — worker-service.cjs 在 macOS 睡眠/唤醒后 100% CPU 挂起；HTTP 监听器停止响应
- #3386 — 插件版本升级时 worker 回收在 server.close() 失败后仍保持端口绑定（维护者称已在 13.12.4 修复 —— 验证并关闭）
- #3398 — worker 在 Claude CLI 子进程探测/挂起后对健康检查无响应，拖住同步 Hook
- #3415 — 长期运行的 worker 守护进程永远写入其启动日期的文件；GET /api/logs 读取今天的文件
- #3420 — 过期的 chroma 写入锁（死 PID）永久卡死 worker 守护进程（维护者：过期锁部分已在 main 修复 —— 验证）
- #3448 — 当已验证的存活所有者始终不健康时，worker 永远卡在 'warming' —— 有界已验证所有者回收提案
- #3450 — 杀死 worker 后端口仍通过继承的套接字句柄绑定；Hook 客户端挂起并堆积（296+ 进程）
- #3469 — [Windows] worker 从不绑定：在启动缓存后静默挂起，即使在空闲端口上
- #3482 — Windows：孤儿 chroma 子进程继承 worker 的监听套接字，永久阻塞端口 37777
- #3484 — Windows：尽管 worker 健康，Hook 健康检查从不成功（僵尸端口与配置端口不匹配）
- #3496 — 数据目录不可写导致每个 Hook 耗时 48 秒而非快速失败；派生锁失败开放，导致 worker 堆积
- #3546 — 13.14.0 上 worker 进程无界累积（12 个进程 / 2.17 GB RSS，仍在攀升）
- #3557 — Windows 上 worker 在启动约 1 秒后干净退出（代码 0）—— 从不绑定端口，不写 CAPTURE_BROKEN 标记
- #3596 — worker 不可达时 UserPromptSubmit Hook 阻塞所有输入（过期 :37777 监听器上的不可恢复死锁）

待评估/变基的相关 PR：#3405（注册表验证回收 + 故障转移端口）、#3416（端口迁移）、#3431、#3309（监听套接字继承）、#2895（真实端口占用检测）、#2980、#3009、#3112、#2998、#3099、@dajiaohuang 的有界 fetch 超时 PR（#3553 讨论串）、#3231、#3219。

## 修复顺序

1. **不可继承的监听器。** 在 Windows 上用 `WSA_FLAG_NO_HANDLE_INHERIT` / `SetHandleInformation(HANDLE_FLAG_INHERIT, 0)`、在 POSIX 上用 `FD_CLOEXEC` 创建 HTTP 监听器；派生每个子进程时显式指定 `stdio`（绝不用 `inherit`），使任何后代都无法持有端口。添加 CI 测试：杀死 worker 后断言在子进程存活的情况下端口可在 1 秒内绑定。
2. **所有者记录 + 启动令牌。** worker 将 `{pid, startToken, port, scriptPath, version, bootId}` 写入 `worker-port.json`；`/api/health` 回显 `startToken` 和 `bootId`。每个存活判定（`ensureWorkerRunning`、`worker-cli start/stop/restart`、Hook 的就绪等待、版本不匹配回收）都读取该记录，并用 **5 秒** `AbortSignal` 调用 `/api/health`；令牌匹配是"存活"的唯一定义。
3. **回收策略（有界、已验证）。** 端口被绑定 + 无令牌匹配 ⇒ 若记录的 PID 的命令行是 claude-mem worker，则对其发 SIGTERM → 等待 → SIGKILL → 清扫注册表中的子进程（plan-14 清扫）→ 重新探测可绑定性；若仍被无主进程绑定，则**迁移**到下一个空闲端口并持久化（`CLAUDE_MEM_WORKER_PORT` 回退文件），使 Hook 和 MCP 跟随。每次启动器调用只尝试一次回收；绝不动应答 `/api/health` 的 PID。
4. **带存活检查的锁。** `spawn.lock`、`worker.pid` 和 chroma 写入锁都携带 `{pid, startToken, createdAt}`；获取时检查 `process.kill(pid,0)` **和**命令行；持有者已死 ⇒ 回收，而非等待。锁写入因 EACCES/EROFS/ENOSPC 失败时大声中止而非失败开放（#3496）：启动时先运行 `access(dataDir, W_OK)`，并将失败写入 stderr 和 runner-errors 日志。
5. **不会说谎的启动。** `startWithBun` 仅在 `/api/health` 返回新令牌后才写 PID 文件；就绪判定用 `/api/readiness`，绝不用引导 PID；到达启动缓存但从不绑定的 worker 以**非零**退出，记录原因并写入 `CAPTURE_BROKEN`（#3557）；Windows 上（`Start-Process` 路径）守护进程的 stdout/stderr 重定向到日志文件，使静默退出不再静默；`--help` 绝不启动 worker。
6. **卡死看门狗。** 进程内心跳：若事件循环阻塞 > 15 秒或 `/api/health` 自探测失败 3 次（DarkWake、CLI 探测挂起、写入锁卡住），worker 关闭已接受的套接字并以非零自我终止，使下一个 Hook 的启动器派生新 worker；睡眠/唤醒（`SIGCONT`、单调时钟跳变）重新初始化监听器。空闲连接超时关闭 CLOSE_WAIT 泄漏。
7. **冷却以证据为准，而非时间。** `.worker-start-attempted` / 派生冷却仅在上次尝试的失败*不是*"端口被死所有者绑定"时生效；可回收的端口总是允许立即重试一次。冷启动等待时间经过测量（近期启动的 p95），下限 15 秒，上限低于 Hook 超时（Hook 侧由 plan-17 负责）。
8. **绑定主机规范化。** 在 `SettingsDefaultsManager`/`getWorkerHost` 中将 `localhost` → `127.0.0.1`，使 worker 和探测永不跨地址族（#2992）。
9. **守护进程可观测性。** 每次写入时重新计算日志路径（日期滚动）或使用按大小轮转的 `claude-mem.log`；`/api/logs` 读取守护进程实际写入的文件；`worker-cli status` 打印所有者记录与实时健康状态，使不匹配一条命令即可发现。

## 测试矩阵

| 主机 | 运行时 | 场景 | 要求行为 |
|---|---|---|---|
| Windows 10/11 | Bun、Node | chroma 进程树存活时杀死 worker | 端口 ≤ 1 秒可绑定；下一个 Hook 的启动器 ≤ 15 秒启动新 worker |
| Windows | Bun | 插件更新 / 版本不匹配回收 | 仅在后继令牌应答 `/api/health` 后才释放现任；无幽灵 LISTEN |
| Windows | Bun | 来自*无主*进程的幽灵监听器（模拟） | 迁移到下一个空闲端口；Hook + MCP 在一次调用内跟随 |
| macOS | Bun | 空闲 worker 睡眠/DarkWake | 唤醒后健康应答 ≤ 5 秒，或 worker 自我重启 |
| macOS / Linux | Bun | 卡死的事件循环（注入 30 秒阻塞） | 探测在 5 秒中止；看门狗重启 worker；Hook 在存活判定上等待不超过 5 秒 |
| Linux / WSL2 | Bun | 对健康 worker 执行 `worker-cli start` | 以 0 退出"已在运行"；绝不报"Process died during startup" |
| 全部 | 全部 | 死 PID 的过期 `spawn.lock` / `worker.pid` / 写入锁 | 首次接触即回收；不触发冷却 |
| 全部 | 全部 | `~/.claude-mem` 不可写 | Hook 在 < 1 秒内失败并给出 stderr 原因；不派生 worker |
| 全部 | 全部 | 冷启动时 30 个并发 Hook 启动器 | 恰好一个 worker；其他启动器附加或退出；无堆积 |
| 全部 | 全部 | 守护进程跨日期边界运行 | `/api/logs` 显示守护进程的 ERROR 行 |

该矩阵在 CI 中运行（`tests/infrastructure/liveness-authority.test.ts`，带可注入的假监听器/子进程，外加 Windows 任务）。回归必须在用户提交前使 CI 失败。

## 范围之外

杀死比 worker 活得更久的子进程（plan-14）。启动器从哪个磁盘副本派生、何时允许触发版本不匹配回收（plan-16）。当权威判定"未存活"时 Hook 做什么 —— 失败开放，不阻塞（plan-17）。


---

## 轮次：2026-09-12 oh-my-issues

- **#4059** — 一旦 `worker.pid` 消失，macOS 版本不匹配回收就无法杀死过期 worker。`verifyPidFileOwnership` 删除 PID 文件，懒派生撞上 `Port already in use, refusing to start duplicate`，而 `c0()` 的不匹配路径需要 PID 文件来识别进程，于是永久放弃。修复正是本计划的核心主张的具体化：直接解析监听器（`lsof -ti tcp:<port>` / `Get-NetTCPConnection`），验证命令行包含 `worker-service.cjs`，然后杀死它；并在 `EADDRINUSE` 分支上用新的所有者记录重新接管监听器。先启动后继再杀死现任由 plan-16（#3604）负责。
- **#4063** — `Logger.ensureLogFileInitialized()`（`src/utils/logger.ts:90`）在 `logFileInitialized` 闩锁*内部*读取日期，将 `logFilePath` 冻结在进程启动的 UTC 日；守护进程随后数天都追加到那里。修复：在查询闩锁之前计算日期并比较。验证注意事项：文件名是 UTC，日志行是本地时间 —— 与 `date -u +%F` 比较，否则每晚会有两小时误报。该编译方法每个发布版本随 7 个 bundle 发布。同一报告：完全没有日志轮转或保留（`rotateLog`、`pruneLogs`、`maxLogFiles`、`retentionDays` 均无匹配），已纳入本计划范围。