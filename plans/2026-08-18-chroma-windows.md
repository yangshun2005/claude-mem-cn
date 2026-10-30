# Windows 上的 Chroma — 进程树归属

状态：可执行
目标：针对 `thedotmack/claude-mem:main` 的**一个** PR
相关计划主单：#3610 [plan-22] Chroma Sidecar 契约、#3602 [plan-14] 子进程归属、#3603 [plan-15] Worker 端口与存活检测

## 问题（已验证，非假设）

Chroma 目前在 Windows 上**能启动**。cmd.exe 的 `>`/`<` 参数被篡改问题（#2954、#3121）已在 `main` 上修复——`ChromaMcpManager` 直接 spawn `uvx`，没有 shell 包装。

坏掉的是**进程树归属**。Chroma 以 4 层原生链运行：

    worker → uvx.exe → uv.exe → python.exe → chroma-mcp

Windows 没有 POSIX 进程组，Node 的 `process.kill(pid, 'SIGTERM')` 只强制终止一个 PID。除一条路径外，所有清理路径都按 PID 杀进程，因此后代进程存活。存活进程随后引发全部三个 Windows 主要 bug：

| 存活进程影响 | Issue | 症状 |
|---|---|---|
| 孤儿进程继承 worker 的监听 socket | #3482 | 端口 37777 卡死，834 次健康检查失败，Hook 硬阻塞 |
| `uv` 在构建中途被杀，临时目录永不回收 | #3540 | `builds-v0/.tmp*` 泄漏——实测 144.21 GB / 696 个目录 |
| 外部 `VIRTUAL_ENV` 被继承进 uvx 沙箱 | #3552 | numpy ABI 冲突，语义同步**静默**停止 |

正确的修复已存在于仓库中，但未被共享：
`ChromaMcpManager.killProcessTree()`（`src/services/sync/ChromaMcpManager.ts:1039-1154`）
执行 POSIX 后代遍历 + Windows `taskkill /PID n /T /F`。没有其他地方使用它。

## 允许的 API（已对照源码验证——不要臆造）

- 通过 `execFileAsync` 调用 `taskkill /PID <n> /T /F`——**这就是** Windows 树杀。先例：`ChromaMcpManager.ts:1044-1047`。
- `pgrep -P <pid>` 用于 POSIX 后代遍历。先例：`ChromaMcpManager.ts:1124-1154`。
- `process.platform === 'win32'`——全仓库使用的检测惯用法。
- `path.join` / `pathToFileURL` / `fileURLToPath`——已是常规做法。
- `getSupervisor().registerProcess/unregisterProcess`——`src/supervisor/index.ts`，Chroma 已注册为 `'chroma-mcp'`。

**反模式——这些不存在，不要使用：**
- Windows Job Objects。Node 不暴露 Job Object API；添加它需要原生插件或 FFI。超出范围。
- 在 Windows 上期望 `process.kill(pid, 'SIGTERM')` 优雅关闭。它始终是强制终止。
- 任何新的 npm 依赖。此 PR 必须零依赖。
- `shell: true` 或将 `uvx` 包装进 `cmd.exe`。这会重新引入 #2954/#3121。

---

## 阶段 1 — 提取共享树杀辅助函数

**实现：** 将可用的实现从 ChromaMcpManager 移到共享模块。

- 创建 `src/shared/kill-process-tree.ts`，导出 `killProcessTree(pid, opts?)` 和 `collectDescendantPids(pid)`。
- 从 `src/services/sync/ChromaMcpManager.ts:1039-1154` **逐字复制**函数体。不要重新设计——
  它经过实战检验（POSIX：先叶子后根 SIGTERM，500ms 等待，重新收集，SIGKILL 并集；
  Windows：`taskkill /T /F`）。
- 让 `ChromaMcpManager` 导入共享辅助函数，并删除其私有副本。

**验证：**
- `grep -n "killProcessTree" src/services/sync/ChromaMcpManager.ts` → 仅导入，无本地定义。
- `npm run build` 干净。
- macOS 上的 Chroma 清理行为不变（现有测试无回归）。

**约束：** 本阶段不要改变算法。纯提取，以便 diff 可审查。

---

## 阶段 2 — 让每个 Windows 杀进程点都走辅助函数

**实现：** 用 `killProcessTree` 替换单 PID 杀进程。审计找到的确切位置：

| 文件:行 | 当前 | 为何在 Windows 上失败 |
|---|---|---|
| `src/supervisor/process-registry.ts:319` | `process.kill(record.pid, 'SIGTERM')` | cmd.exe 包装进程死亡，Claude 子进程成为孤儿 |
| `src/supervisor/process-registry.ts:353` | `process.kill(record.pid, 'SIGKILL')` | 回收器杀 1 个 PID，然后仍删除注册表 |
| `src/supervisor/process-registry.ts:482` | `proc.kill('SIGKILL')` | 仅杀 `.cmd` 包装进程 |
| `src/supervisor/process-registry.ts:770` | `process.kill(record.pid, 'SIGTERM')` | 重复 SDK 清理使后代成为孤儿 |
| `src/supervisor/shutdown.ts:186` | `process.kill(pid, signal)` | 根进程退出，存活扫描永远到不了 taskkill 分支 |
| `src/shared/worker-utils.ts:514` | `process.kill(stalePidInfo.pid, 'SIGKILL')` | **价值最高**——版本回收让整个 uvx→chroma 链存活并持有 socket |
| `src/server/runtime/ServerService.ts:363` | `process.kill(existing.pid, 'SIGTERM')` | 跳过 DB/队列/HTTP 清理处理器 |

`worker-utils.ts:514` 是 #3482 的直接原因——即使本阶段其他内容被裁剪，也要修它。

**验证：**
- `grep -rn "process\.kill(" src/ | grep -v kill-process-tree.ts` → 每个剩余命中都是仅 POSIX 或有意的；在 PR 正文中逐一说明理由。
- `npm run build` 干净；现有 supervisor 测试通过。

---

## 阶段 3 — 阻止 uv 构建目录泄漏

**实现：** 在构建中途树杀 `uv` 正是使 `builds-v0/.tmp*` 成为孤儿的原因（#3540）。

- 在 Chroma 清理路径中，先尝试优雅退出，仅在宽限期后才升级到 `taskkill /T /F`。
  复用现有的 500ms 等待惯用法，不要发明新的定时器方案。
- 为 uv 缓存 `builds-v0` 目录添加陈旧临时目录清扫，仅针对超过保守阈值的条目，
  在 Chroma 启动时运行（不是关闭时——关闭可能是硬杀）。
- 阅读 `src/shared/uvx-bin-dirs.ts` 了解 uv 目录如何定位。不要硬编码路径。

**验证：**
- 目录不存在时清扫为空操作（全新安装不得崩溃）。
- 清扫永不删除属于存活 `uv` PID 的目录。

**约束：** 不得递归删除已解析 uv 缓存目录之外的任何内容。在 unlink 前断言解析路径位于 uv 缓存根之下。

---

## 阶段 4 — 清理 uvx 子进程环境

**实现：** `uvx --python 3.13` 不得继承外部 Python（#3552）。

- 从传给 Chroma 子进程的环境中剥离 `VIRTUAL_ENV`、`PYTHONHOME`、`PYTHONPATH`、`CONDA_PREFIX`、`CONDA_DEFAULT_ENV`。
- 在构建子进程环境的单一位置应用——见 `getUvxPreflightEnv` /
  `effectiveUvxEnv`（`ChromaMcpManager.ts` 和 `src/services/worker/dependency-preflight.ts:75`）。
  这是两个并行实现；两者都清理或统一它们（DRY——优先统一）。

**验证：**
- 单元测试断言：在 `process.env` 被污染的情况下，构建出的环境对象不含这些键。

---

## 阶段 5 — 让 Windows CI 真正证明它（核心交付物）

这些 bug 持续发布的原因：`.github/workflows/windows.yml` 是**仅构建**
（`runs-on: windows-2022`，步骤 = 安装 → 构建 → 一个 Bun 解析器测试）。它从不 spawn Chroma。
没有本阶段，PR 无法验证——我们在 macOS 上，无法本地测试 Windows。

**实现：** 添加一个 Windows job，它：
1. 在 runner 上安装 uv（PowerShell 安装器，与 `setup-runtime.ts:200` 相同）。
2. 通过生产代码路径 spawn 真实的 chroma-mcp。
3. 往返一个文档：创建集合 → 添加 → 查询 → 断言结果返回。
4. 关闭 worker。
5. **断言零孤儿**：没有 `chroma-mcp`、`uv.exe` 或 `python.exe` 后代残留
   （`tasklist` / `Get-CimInstance Win32_Process` 按父链过滤）。
6. **断言无临时目录泄漏**：`builds-v0/.tmp*` 数量未增长。

步骤 5 和 6 是回归门禁——它们是全部重点。

**验证：** 该 job 必须在 `main` 上**失败**（证明它能捕获 bug），在应用阶段 1-4 后**通过**。
在 PR 描述中演示这一点。如果它在 `main` 上通过，测试就是错的——修正测试。

---

## 阶段 6 — PR + 跟进

- 从当前 `main` 拉出分支 `fix/chroma-windows-process-tree`。
- PR 正文必须说明：根因、7 个杀进程点、CI 证明（main 上失败 / 修复后通过），
  以及此 PR 取代或重叠哪些社区 PR。
- **重叠披露（已通过 `gh` 验证）：** `ChromaMcpManager.ts` 被 4 个开放 PR 修改——
  #3541（uv 宽限期）、#3567（环境清理）、#3286（Job Object）、#3292（广泛恢复）。
  它们互不引用。此 PR 用共享辅助函数覆盖 #3541 和 #3567 的范围，而不是四次独立编辑。
  在 PR 正文中明确致谢他们。
  相关开放 PR：#3309（socket 继承，当前 CONFLICTING/DIRTY）、#3416（端口重绑）、
  #3529（windowsHide）、#3321（where.exe PATH）。
- 跟进：关注 CI，处理审查意见，重跑直到绿色且可合并。

## 超出范围（在 PR 中明确说明，不要静默丢弃）

- 原生 Windows 上仅 Bash 的 Hook（`plugin/hooks/hooks.json` `"shell": "bash"`）——属于计划主单 #3605。
- `tree-sitter.exe` 查找未命中（`src/services/smart-file-read/parser.ts:362`）——真实 MAJOR bug，单独 PR。
- `~\` 波浪号展开（`src/shared/paths.ts:32`）——真实 MAJOR bug，单独 PR。
- 大小写敏感的路径包含检查（P5、P6）；`/dev/null` 与 `NUL`（P3、P4）——MINOR。
- 协调其他 31 个开放的 Windows/Chroma PR。