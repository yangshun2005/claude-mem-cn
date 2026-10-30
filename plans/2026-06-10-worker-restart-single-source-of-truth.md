# Worker 重启：单一事实来源

**创建日期：** 2026-06-10
**根因分析：** 12 个 agent 诊断，经对抗性验证（workflow `wf_f07f3541-b05`）。摘要：worker 重启失败源于五个冗余的"谁是 worker"判定源，写入方互不协调；一个同步脚本"热重载"镜像，把版本 N 的代码写入版本 (N-1) 的缓存目录；一个只杀不启的重启端点，与 Hook 的懒启动竞争；以及一条构建链，触发两次不协调的重启且从不验证结果。6 月 8–10 日观察到 98 次版本回收乒乓事件和 6 次 EADDRINUSE 硬失败。

**执行模型：** 每个阶段自包含、独立落地（每阶段一个提交/PR，按顺序）。阶段 1–2 是高杠杆、低风险的部分。继续之前先运行 `npm test` 及该阶段的验证清单。

---

## 阶段 0：整合文档——允许的 API 与事实基准

*（发现工作已由三个事实提取子 agent 于 2026-06-10 完成；此处整合。执行者：信任这些引用，但编辑前重读所引行——行号会漂移。）*

### 允许的 API（已验证存在——只用这些，且用这些精确签名）

| API | 位置 | 签名 / 形态 |
|---|---|---|
| `ensureWorkerRunning()` | `src/shared/worker-utils.ts:293-381` | `(): Promise<boolean>` — Hook 懒启动 + PR #2768 版本回收 |
| `resolveWorkerScriptPath()` | `src/shared/worker-utils.ts:206-215` | 候选：`MARKETPLACE_ROOT/plugin/scripts/worker-service.cjs`，然后 `cwd()/plugin/scripts/worker-service.cjs` |
| `resolveBunRuntime()` | `src/shared/worker-utils.ts:217-235` | Hook 侧解析器；缺少 `~/.bun/bin` 回退 |
| `waitForWorkerPort` / `waitForWorkerReadiness` | `src/shared/worker-utils.ts:237-273` | 轮询 `GET /api/readiness`；预算环境变量 `CLAUDE_MEM_HOOK_READINESS_TIMEOUT_MS` |
| `ensureWorkerStarted(port, workerScriptPath)` | `src/services/worker-spawner.ts:70-153` | 返回 `'ready' \| 'warming' \| 'dead'`；无版本守卫 |
| `spawnDaemon(scriptPath, port, extraEnv?)` | `src/services/infrastructure/ProcessManager.ts:408-472` | 返回 PID 或 `undefined`；Unix 上 setsid，Windows 上 PowerShell |
| `resolveWorkerRuntimePath(options?)` | `src/services/infrastructure/ProcessManager.ts:63-125` | 完整 bun 解析链（BUN、BUN_PATH、`~/.bun/bin`、brew 路径、`which`） |
| PID 文件 API | `ProcessManager.ts:134-168, 508-520` | `writePidFile(info)`、`readPidFile(): PidInfo\|null`、`removePidFile()`、`touchPidFile()`、`cleanStalePidFile()` |
| `httpShutdown(port, reason?)` | `src/services/infrastructure/HealthMonitor.ts:94-114` | POST `/api/admin/shutdown?reason=restart` |
| `waitForPortFree(port, timeoutMs?)` | `HealthMonitor.ts:85-92` | 500ms 轮询 |
| `checkVersionMatch(port)` | `src/services/infrastructure/HealthMonitor.ts:~120-161` | 返回 `{matches, pluginVersion, workerVersion}`；ENOENT 时 fail-open |
| `performGracefulShutdown(config)` | `src/services/infrastructure/GracefulShutdown.ts:30-58` | 顺序关闭，无全局截止时间 |
| `flushResponseThen(res, payload, action)` | `src/services/server/flushResponseThen.ts:3-16` | 响应后于 'finish' 执行 action，始终 `process.exit(0)` |
| `writeJsonFileAtomic()` | `src/npx-cli/utils/paths.ts:124-205` | 仓库中唯一的原子写入辅助函数 |
| `MARKETPLACE_ROOT` | `src/shared/paths.ts:43` | `~/.claude/plugins/marketplaces/thedotmack` |
| `resolveDataDir()` / `DATA_DIR` | `src/shared/paths.ts:18-40` | 环境变量 `CLAUDE_MEM_DATA_DIR` 优先（第 19-20 行）；模块级常量——导入时计算 |
| Worker 端口默认值 | `src/shared/SettingsDefaultsManager.ts:91` | `37700 + (uid % 100)` — 绝不硬编码 37777 |
| `/api/health` 响应 | `src/services/server/Server.ts:212-233` | `{status, version, workerPath, uptime, pid, initialized, mcpReady, ...}` — 包含验证所需的一切 |
| `/api/readiness` 响应 | `Server.ts:235-247` | `{status: 'ready'\|'initializing', mcpReady}` |
| `/api/admin/restart` | `Server.ts:282-294` | 经 `flushResponseThen` 只杀不启 → `onRestart()` → `shutdown('restart')`；无任何重启（macOS/Linux） |
| `version: BUILT_IN_VERSION` | 经 `__DEFAULT_PACKAGE_VERSION__` esbuild define 烘焙 | `scripts/build-hooks.js:312,365,421,469,495,540` |

### 反模式（已验证不存在——不要发明）
- Server.ts 中不存在 `/api/admin/status`，不存在 `/api/version` 路由（版本来自 `/api/health`），HTTP 重启路径中不存在任何重启。
- src/ 中不存在 flock/O_EXCL/`wx` 标志锁文件模式——阶段 4 引入第一个；写入纪律参照 `writeJsonFileAtomic`。
- `plugin/scripts/*.cjs` 是构建产物——绝不手改；用 `npm run build` 重建。
- `package-lock.json` 被 gitignore——不要提交。
- 测试用 `bun test`（`bunfig.toml` 预加载 `tests/preload.ts` 以 mock PostHog）。Mock 模式：`mock.module()` + 查询参数缓存破坏式全新导入（`tests/shared/worker-utils-version-recycle.test.ts:22-32`）。

### 必须持续通过的行为契约
- `tests/shared/worker-utils-version-recycle.test.ts`：版本不匹配时 `ensureWorkerRunning()` POST `/api/admin/restart` ≥1 次；匹配时 0 次。
- `tests/integrations/spawn-contract-windows.test.ts`：spawn 契约环境变量覆盖。
- 全套件：截至 v13.5.5 为 2203 通过 / 0 失败。

---

## 阶段 1：删除缓存镜像；让 CLI 重启成为唯一发起者

**为何优先：** 镜像（`sync-marketplace.cjs:164-173`）是版本分歧的最大人为来源（10 个缓存目录中 7 个内容差一）；双重重启（HTTP POST + sleep 1 + CLI 重启）是竞争生成器。两个修复都是删除。

### 实现内容
1. 在 `scripts/sync-marketplace.cjs` 中：
   - 删除缓存镜像块（约第 164-173 行：`INSTALLED_CACHE_PATH` + rsync + 向缓存目录 `bun install`）。
   - 删除 `detectInstalledVersion()`（约第 79-114 行）及其调用点——它只为镜像服务。
   - 删除 HTTP 重启触发块（约第 196-216 行：向 `/api/admin/restart` 的 `http.request` POST 及其成功/错误打印）。同步脚本的职责止于"文件已同步 + 在市场副本中 `bun install`"。
2. 根 `package.json` 约第 67 行，简化：
   - 从：`"build-and-sync": "npm run build && npm run sync-marketplace && sleep 1 && (cd ~/.claude/plugins/marketplaces/thedotmack && npm run worker:restart)"`
   - 到：`"build-and-sync": "npm run build && npm run sync-marketplace && (cd ~/.claude/plugins/marketplaces/thedotmack && npm run worker:restart)"`
   - （去掉 `sleep 1`——它存在是为了让现已删除的 HTTP 杀进程在 CLI 重启前落地）
3. 不要动 sync-marketplace.cjs 中其他位置现有的一次性旧缓存清理（若存在）——只动上述三块。

### 文档引用
- 三块的逐字引用：阶段 0 表格 + 重启/关闭提取（sync-marketplace.cjs:79-114, 164-173, 196-216；package.json:64-79）。

### 验证清单
- `grep -n "INSTALLED_CACHE_PATH\|detectInstalledVersion\|admin/restart" scripts/sync-marketplace.cjs` → 无匹配。
- `grep -n "sleep 1" package.json` → build-and-sync 中无匹配。
- `npm run build-and-sync` → 完成；worker 重启（仅经 CLI 路径）；`curl -s http://127.0.0.1:$PORT/api/health` 显示刚构建的版本。（`$PORT` 解析为 `37700 + uid%100` 或从 `~/.claude-mem/settings.json` 读取。）
- `npm test` → 通过。
- 陈旧缓存回归检查：`ls ~/.claude/plugins/cache/thedotmack/claude-mem/*/package.json | xargs grep -h '"version"'` — 记录当前值；下一个发布版本后，复查无缓存目录内容变化（镜像过去会改动它们）。

### 反模式守卫
- 不要通过改为针对 `workerPath` 来"修复"镜像——该功能是移除，不是修复。
- 不要在此添加新的重启机制；阶段 2 加固现有 CLI 重启。

---

## 阶段 2：重启后验证——重启自证或退出 1

**为何：** `restart` 目前在 `spawnDaemon` 返回 PID 后即退出 0——那是 fork 成功，不是启动成功。四条静默退出 0 的守护进程路径意味着 worker 已死/陈旧却显示"✓"。

### 实现内容
全部在 `src/services/worker-service.ts`：
1. 在 `case 'restart'`（约第 956-973 行）：
   - 在 `httpShutdown` 之前，捕获旧 worker：`GET /api/health`（2s 超时）→ 保存 `oldPid`（无 worker 时可为 null）。
   - 将 `waitForPortFree(port, 5000)` 替换为 `waitForPortFree(port, getPlatformTimeout(15000))`——与 `stop`（约第 946 行）对齐。
   - 在 `spawnDaemon` 之后，添加验证循环（新辅助函数 `verifyRestartedWorker(port, oldPid, deadlineMs)`）：每 500ms 轮询 `GET /api/health`，直到 `health.pid !== oldPid && health.version === EXPECTED_VERSION`，其中 `EXPECTED_VERSION` 是本进程自身烘焙的 `__DEFAULT_PACKAGE_VERSION__`（build-and-sync 运行市场副本，故其烘焙版本就是刚同步的版本）。截止时间：`getPlatformTimeout(30000)`。
   - 成功时：记录 `Worker restart verified {pid, version}` 并退出 0。超时时：`console.error` 输出最后观察到的 health 载荷（或连接错误）并**退出 1**。
2. 在 `--daemon` 块（约第 1167-1208 行）：只改退出路径 4（通用启动失败，约第 1204-1206 行），从 `process.exit(0)` 改为 `process.exit(1)`。路径 1-3 是合法的重复抑制，保持退出 0。
3. Spawn 目标：将 `spawnDaemon(__filename, port)`（约第 965 行）改为优先市场脚本——复制 `resolveWorkerScriptPath()`（`src/shared/worker-utils.ts:206-215`）的候选模式，无市场副本时回退到 `__filename`（开发树、CI）。

### 文档引用
- `/api/health` 形态：`Server.ts:212-233`（`pid`、`version` 字段已确认）。
- `getPlatformTimeout`：用于 `worker-service.ts:946`。
- worker-service bundle 内 `__DEFAULT_PACKAGE_VERSION__` 可用性：`scripts/build-hooks.js:312-313`。

### 验证清单
- 新测试 `tests/services/worker-restart-verify.test.ts`：mock `global.fetch`（复制 `tests/shared/worker-utils-version-recycle.test.ts:34-50` 的 fetchLog 模式）；断言 `verifyRestartedWorker` 在 health 翻转为 `{pid: newPid, version: expected}` 时返回成功，陈旧 pid 失败，版本错误失败，超时失败。
- 手动：`npm run build-and-sync` → 输出包含 `Worker restart verified`；然后在重启窗口中途 `kill -9 <worker pid>` 并重跑，看到响亮的退出 1 路径（或在测试中将验证循环指向死端口模拟）。
- `npm test` → 通过，含版本回收契约。

### 反模式守卫
- 不要轮询 `/api/version`（不存在）或发明新 health 字段。
- 不要为 `EXPECTED_VERSION` 比对磁盘上的市场 `package.json`——烘焙常量才是"我正在运行的代码"的事实；磁盘读取会重新引入第二个判定源。
- 验证截止时间不得永久阻塞：硬上限，然后退出 1。

---

## 阶段 3：自替换重启——旧 worker 生成其后继；回收路径停止向尸体 spawn

**为何：** `/api/admin/restart` 只杀不启；POST 它的 Hook 随后懒启动，与垂死 worker 竞争（乒乓）。若旧 worker 在端口关闭后作为最后动作生成后继，新旧永不共存，也无第三方向尸体 spawn。

### 实现内容
1. `src/services/worker-service.ts` `shutdown()`（约第 671-699 行）：
   - 重入守卫：`isShuttingDown` 字段（约第 188 行）目前只写；让 `shutdown()` 在入口检查并设置（`if (this.isShuttingDown) return; this.isShuttingDown = true;`）。
   - 硬截止：用 `getPlatformTimeout(10000)` 计时器将 `performGracefulShutdown(...)` 包在 `Promise.race` 中；超时时记录 `Graceful shutdown deadline exceeded — proceeding` 并继续（不要挂在无界会话排空上——排空今天可能跑 35-40s）。
   - 后继 spawn：当 `reason === 'restart'`，在优雅关闭完成/超时后：解析市场脚本（同阶段 2 步骤 3 的候选模式），`await waitForPortFree(port, 5000)`，然后 `spawnDaemon(marketplaceScript, port)`。若端口始终不释放或 spawn 返回 `undefined`，大声记录（`logger.error`）——下一个 Hook 的懒启动是安全网。注意 `flushResponseThen`（flushResponseThen.ts:3-16）在 action 完成后调用 `process.exit(0)`，故 spawn 必须在 action 内 await。
2. `src/shared/worker-utils.ts` `ensureWorkerRunning()` 回收路径（约第 305-330 行）：
   - POST `/api/admin/restart` 后，不要立即懒启动。改为轮询 `GET /api/health`（500ms 间隔，`HOOK_READINESS_TIMEOUT_MS` 预算）等待后继：健康且 `version === pluginVersion`（已从 `checkVersionMatch` 在手）。仅当后继始终不出现时才落入现有懒启动。
   - 放大器修复：本函数中任何位置 `waitForWorkerReadiness` 成功后，经 `/api/health` 再查一次版本；若仍不匹配，记录警告（不要在同一次调用中循环/再回收——每个 Hook 事件一次回收，下一个 Hook 重试；此处无界循环会重建风暴）。

### 文档引用
- `onRestart` 接线：`worker-service.ts:255`；路由：`Server.ts:282-294`；`flushResponseThen.ts:3-16`。
- 排空时序 + 无界 await：`GracefulShutdown.ts:30-75`。
- `HOOK_READINESS_TIMEOUT_MS`：`worker-utils.ts:45-48`。
- 要镜像的信号路径守卫：`src/supervisor/index.ts:56-60`。

### 验证清单
- `tests/shared/worker-utils-version-recycle.test.ts` 仍通过（不匹配时仍 POST 重启——变化在 POST 之后发生的事）。
- 新测试：shutdown 重入——调用 `shutdown()` 两次，断言 `performGracefulShutdown` 只运行一次（经 `mock.module` mock）。
- 新测试：回收不向尸体 spawn——fetch mock 中 `/api/admin/restart` 返回 200 且 `/api/health` 在第 N 次轮询返回新版本：断言无 spawn 尝试；health 始终不恢复时：断言懒启动回退触发。
- 手动端到端：worker 运行时，`curl -X POST http://127.0.0.1:$PORT/api/admin/restart`；约 15s