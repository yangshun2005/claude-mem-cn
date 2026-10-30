# 安装器失败透明化 — 跨 IDE 矩阵

**目标：** 阻止通用安装器（`npx claude-mem install`）在所有 12 个 IDE 上静默吞掉真实失败并谎报"安装成功"。将每个错误抑制点转换为由显式分类法驱动的单一 `installerError(severity, ctx)` 决策点。让 `tree-sitter` ERESOLVE 冲突和缺失 `uv` 大声失败，并给出平台特定的修复指引。新增 12 IDE × 4 失败模式验证矩阵，以及受 v12.6.2 `tree-sitter-swift` 修复启发的 CI postinstall 回归防护。

**净效果：**
- 仅当所有 ABORT 级依赖都满足时才打印"Installation Complete"。部分成功时显示黄色"Installation Partial"标题及修复块。
- `runNpmInstallInMarketplace()` 先严格运行；仅在确认出现 `ERESOLVE` 标记时才应用 `--legacy-peer-deps`，并大声宣告回退。
- 自动安装尝试后仍缺失 `uv` = ABORT，平台特定说明作为主消息呈现（而非埋在"版本探测失败"的包装行下）。用户已选择退出向量搜索时，降级为 WARN_CONTINUE。
- Postinstall 回归防护：任何新增的带 `scripts.postinstall` 或 `scripts.install` 的传递依赖，若不在显式允许列表中，则构建失败，防止 v12.6.1 `tree-sitter-swift` 挂起重演。
- 跨 IDE 测试矩阵：12 IDE × 4 场景（正常 / ERESOLVE / 缺失 uv / 缺失 bun）= 48 个单元格，每个断言退出码、摘要文本和修复指引存在性。

**范围外（推迟到后续计划）：**
- 替换 `bun-runner.js`（其自身运行时问题；跟踪于 `plans/2026-04-29-installer-streamline.md`）。
- 将 `bufferConsole` 重构为结构化事件流（本计划仅修复数据丢失；完整流式 UX 留待后续）。
- 国际化新的修复消息（暂仅英文）。
- 将 `openclaw/install.sh` 从 bash 迁移到 TypeScript（仅审计；修复为原地加固）。

---

## 问题陈述（含行号引用）

当前存在的具体被吞错误：

| # | 文件 | 行号 | 当前行为 | 为何重要 |
|---|---|---|---|---|
| 1 | `src/npx-cli/commands/install.ts` | 1126–1135 | 捕获*所有* `npm install` 错误，打印 `console.warn`，返回误导性任务消息 `Dependencies may need manual install ⚠`。周围安装仍以 `installed successfully!` 结束。 | 真正的 `ERESOLVE`（或任何 npm 崩溃）变成用户立即忽略的黄色提示。 |
| 2 | `src/npx-cli/commands/install.ts` | 565–581 | `runNpmInstallInMarketplace` 总是使用 `npm install --omit=dev --legacy-peer-deps`。该标志无条件掩盖真实 peer 冲突。 | 下次 tree-sitter peer 范围收紧时，`--legacy-peer-deps` 会静默安装损坏的 tree，我们只能在运行时失败中看到。 |
| 3 | `src/npx-cli/install/setup-runtime.ts` | 206–219 | 若自动安装后 `getUvVersion()` 返回 null，抛出"uv installed but version probe failed."。`runInstallCommand` 未用平台特定说明包装；用户在 clack spinner 期间看到被包装的错误，可能被覆盖。 | 在正常路径上兑现 CLAUDE.md 的"缺失时自动安装 uv"承诺，但失败时退化为令人困惑的一行。 |
| 4 | `src/npx-cli/commands/install.ts` | 163–169, 328–347 | 每个 IDE 的失败通过 `bufferConsole`（第 43–64 行）推入 `pendingErrors[]`。`installStatus`（第 1197 行）只读 `failedIDEs.length > 0`，因此 `bufferConsole` 返回 0 后抛出的 IDE 不可见。摘要行"Failed: …"是唯一信号。 | 单个 IDE 失败产生黄色注释，在绿色"installed successfully!"结尾上方滚出屏幕。 |
| 5 | `src/npx-cli/commands/install.ts` | 1131 | `console.warn('[install] npm install error:', …)` — 错误被记录但未分类、重试或在摘要中呈现。 | 与 #1 同根因：stderr 消失，退出码保持 0。 |
| 6 | `src/npx-cli/commands/install.ts` | 1161–1166 | `disableClaudeAutoMemory` 失败今天被归类为"WARN_CONTINUE"（严重性正确），但实现是临时的。 | 不一致 — 此文件中每个其他 catch 使用不同的日志形态。 |
| 7 | `openclaw/install.sh` | 36 处 `2>/dev/null` / `\|\| true`（例如第 169、224–229、251、255、289、293、405、435、471、495、572、612、631、670、1076、1155、1161、1185 行） | curl/jq/find/健康检查流水线上的 bash 级错误抑制。许多是正确的（尽力探测），但若干掩盖了真实安装失败。 | 一些 `\|\| true` 模式隐藏了缺失的 `bun` 或不可写的插件目录。 |
| 8 | `src/services/integrations/*.ts` | 7 个文件（Codex、Cursor、Gemini、OpenCode、OpenClaw、Windsurf、MCP）中 50+ 个 catch 块 | 每个集成安装器有自己的临时错误处理。错误返回非零，被 `bufferConsole` 缓冲，然后丢弃。 | IDE 矩阵有 12 条不同的失败 UX 路径。 |
| 9 | `scripts/build-hooks.js` | 生成 `plugin/package.json`，包含所有 tree-sitter 依赖和 `trustedDependencies: ['tree-sitter-cli']`。没有 CI 防护阻止在此允许列表之外添加带 `scripts.postinstall` 的新包。 | v12.6.1 的确切根因 — 任何编辑此文件的人都能重演。 |

### 参考事件（标准教训）

`CHANGELOG.md:93–110` 记录 v12.6.1 → v12.6.2：PR #2300 将 21 个 tree-sitter 语法从 devDependencies 移到 dependencies；`tree-sitter-swift` 的 postinstall 拉取嵌套的 `tree-sitter-cli`，后者下载 Rust 二进制并收到 SIGINT。**教训：** npm 不遵守 `trustedDependencies`（仅 Bun）。任何带网络 postinstall 的新传递依赖都可能挂起 `npx claude-mem install`。Phase 7 将其转为 CI 防护。

---

## Phase 0 — 文档发现

以下每个实现阶段按行号引用这些事实；不要重新推导。

### 可复制的允许 API / 模式

| 项目 | 位置 | 复制内容 |
|---|---|---|
| 现有 clack `runTasks` / `bufferConsole` 模式 | `src/npx-cli/commands/install.ts:32–64` | 任务返回字符串；编排器处理 spinner。复用，但每个错误都通过 `installerError` 路由。 |
| `describeExecError`（stdout/stderr 提取器） | `src/npx-cli/install/setup-runtime.ts:100–112` | 已是 child_process 错误的规范。移到共享模块。 |
| 部分状态的标记写入模式 | `src/npx-cli/install/setup-runtime.ts:262–275` | 为新的 `~/.claude-mem/last-install-error.json` 使用相同 JSON 形状（`{ severity, component, phase, cause, …}`）。 |
| 插件缓存解析 | `src/npx-cli/utils/paths.ts`（`pluginCacheDirectory`、`marketplaceDirectory`） | 所有路径解析必须遵守 `CLAUDE_MEM_DATA_DIR`；复用而非发明。 |
| 现有 IDE 列表（规范 12 个） | `src/npx-cli/commands/ide-detection.ts:40–129` | claude-code、gemini-cli、opencode、openclaw、windsurf、codex-cli、cursor、copilot-cli、antigravity、goose、roo-code、warp。 |
| `trustedDependencies` 允许列表（postinstall 防护） | `scripts/build-hooks.js:106–108` 和根 `package.json:190–202` | Phase 7 强制执行的模式。 |
| 现有安装测试（扩展，不替换） | `tests/install-non-tty.test.ts`、`tests/setup-runtime.test.ts`、`tests/install-disable-auto-memory.test.ts` | 相同测试框架形状（mock spawn、隔离 TMPDIR HOME）。 |
| Docker 测试框架（干净 Linux） | `Dockerfile.test-installer` | 已支持在未预装 bun/uv 的情况下运行安装。Phase 6 为此分叉出矩阵运行器。 |
| CLAUDE.md 退出码契约 | `CLAUDE.md` "Exit Code Strategy" 节 | Hook：退出 0 = 成功，1 = 非阻塞，2 = 阻塞。安装器不是 hook — 可因 ABORT 退出 1 或 2。Phase 8 交叉引用。 |
| 先前计划格式 | `plans/2026-04-29-installer-streamline.md`、`plans/2026-04-30-onboarding-ux-overhaul.md` | 分阶段布局、文件清单、反模式表。 |
| v12.6.2 事件文本 | `CHANGELOG.md:93–110` | Phase 7 在代码注释中逐字引用。 |

### 外部事实（引用）

| 主题 | 来源 / 规范参考 | 关键事实 |
|---|---|---|
| npm `ERESOLVE` 语义 | `npm install` 文档（npm v10+）和 npm RFC 0023 | `ERESOLVE` 在 stderr 上以确定性前缀 `npm error code ERESOLVE` 发出，后跟 `While resolving:` 块。`--legacy-peer-deps` 跳过 peer 依赖解析；`--force` 接受冲突树。它们不等价 — `--force` 更激进，*不是*我们想要的。 |
| Bun 安装错误 | `bun install` 源码 / 文档 | Stderr 行以 `error:` 开头。peer 依赖违规打印 `error: package "X" has unmet peer "Y"`。网络失败打印 `error: failed to resolve`。 |
| uv 安装脚本返回码 | `https://astral.sh/uv/install.sh` | 成功时退出 0，即使二进制落在非 PATH 目录（例如 `~/.local/bin` 尚未在 `PATH` 上）。版本探测必须在脚本运行后检查 `UV_COMMON_PATHS`。 |
| Claude Code hook 退出码契约 | `CLAUDE.md` "Exit Code Strategy" | Worker/hook 错误退出 0（Windows Terminal 卫生）。`npx claude-mem install` CLI 不是 hook，允许在 ABORT 时退出非零。 |

### 反模式 / 不存在的 API 方法（避免发明）

- 今天**没有**中央 `installerError` 函数。Phase 3 必须创建它。不要伸手去拿不存在的辅助函数。
- `--force` **不是** `--legacy-peer-deps` 的替代品。Phase 4 不得将回退"升级"为 `--force` — 那会掩盖比 ERESOLVE 更多的东西。
- npm 在 CLI 级别**没有** `--no-postinstall` 标志。正确的标志是 `--ignore-scripts`。不要发明。
- Bun 的 `trustedDependencies` **不被** npm 遵守。不要假设同一允许列表对两者都有效。Phase 7 强制执行单独的 npm 级防护。
- `process.exitCode = 1`（install.ts 第 1324 行）**不会**中止进行中的 `await` 链。Phase 3 的 `InstallAbortError` 必须抛出，而不仅仅是设置 `exitCode`。
- `bufferConsole` 包装器（install.ts:43–64）在缓冲区内部**吞掉** stderr；除非显式刷新，不要假设 stderr 在非交互模式下会到达终端。
- `clack` 的 `p.spinner()` 在 `.stop()` 时*覆盖*该行。spinner 期间通过 `console.warn` 发出的错误会丢失。Phase 3 的 WARN_CONTINUE 必须入队到摘要列表，而非实时记录。
- `ensureUv()` 已在失败时抛出 — 但抛出被上一级 clack 任务运行器捕获，后者在单行中显示消息。不要假设用户会读它；Phase 5 必须添加显式 ABORT 块。
- `install/public/install.sh` 和 `install/public/installer.js` 文件**已是弃用存根**（已验证 — 两者都只打印"use npx claude-mem install"）。不要浪费审计时间。
- `openclaw/install.sh` 是活跃的 shell 安装器（1653 行）。它在 Phase 1 有自己的 bash 级审计。

### 文件清单

| 文件 | 行数 | 处置 |
|---|---|---|
| `src/npx-cli/commands/install.ts` | 1371 | 大量编辑（Phase 1、3、4、5） |
| `src/npx-cli/install/setup-runtime.ts` | 288 | 编辑（Phase 5、7） |
| `src/npx-cli/install/error-taxonomy.ts` | 新建 | 创建（Phase 2） |
| `src/npx-cli/install/error-reporter.ts` | 新建 | 创建（Phase 3） |
| `src/services/integrations/CodexCliInstaller.ts` | ~360 | 编辑（Phase 3）— 每个 catch 路由到 `installerError` |
| `src/services/integrations/CursorHooksInstaller.ts` | ~530 | 编辑（Phase 3） |
| `src/services/integrations/GeminiCliHooksInstaller.ts` | ~310 | 编辑（Phase 3） |
| `src/services/integrations/OpenCodeInstaller.ts` | ~250 | 编辑（Phase 3） |
| `src/services/integrations/OpenClawInstaller.ts` | ~260 | 编辑（Phase 3） |
| `src/services/integrations/WindsurfHooksInstaller.ts` | ~395 | 编辑（Phase 3） |
| `src/services/integrations/McpIntegrations.ts` | ~220 | 编辑（Phase 3） |
| `openclaw/install.sh` | 1653 | 审计并选择性加固（Phase 1） |
| `scripts/build-hooks.js` | ~250 | 编辑（Phase 7）— postinstall 允许列表防护 |
| `scripts/check-postinstall-allowlist.js` | 新建 | 创建（Phase 7）— 预发布 CI 脚本 |
| `tests/install-error-matrix.test.ts` | 新建 | 创建（Phase 6）— 12 × 4 矩阵 |
| `tests/install-non-tty.test.ts` | 277 | 扩展（Phase 6） |
| `tests/setup-runtime.test.ts` | 135 | 扩展（Phase 5） |
| `Dockerfile.test-installer-matrix` | 新建 | 创建（Phase 6） |
| `docs/public/troubleshooting.mdx` | 新建或扩展 | 编辑（Phase 8） |
| `CLAUDE.md` "Exit Code Strategy" | 现有 | 编辑（Phase 8）— 交叉引用分类法 |
| `CHANGELOG.md` | — | **不要编辑** — 按 CLAUDE.md 自动生成 |

---

## Phase 1 — 审计每个错误抑制模式

**目标：** 生成安装器路径中每个 `catch`、`|| true`、`2>/dev/null` 和 `try {} catch {}` 的确定表。每行获得提议的 Phase 2 分类（ABORT / FAIL_LOUD_PER_IDE / WARN_CONTINUE / SILENT_RETRY）。

**交付物：** `plans/audit-installer-errors.csv`（与本计划一起提交），列：
`file, line, kind (catch | bash-or-true | bash-redirect), current_behavior, proposed_severity, proposed_remediation_text, notes`。

### 审计内容（精确 grep）

从仓库根运行这些 grep，将每个命中转为一行：

```bash
# TS catch blocks
grep -nE 'catch\s*(\(|\{)' src/npx-cli/ src/services/integrations/ -r

# TS empty catch
grep -nB1 'catch\s*\{\s*\}' src/npx-cli/ src/services/integrations/ -r

# TS console.warn after caught error
grep -nE 'catch.*\{' src/npx-cli/ src/services/integrations/ -r -A 3 | grep -A 0 'console\.warn\|log\.warn'

# Shell silent failures
grep -nE '\|\| true|2>/dev/null|2>&1.*\|\|' openclaw/install.sh

# Build / sync scripts
grep -nE 'catch|process\.exit\(0\)' scripts/build-hooks.js scripts/sync-marketplace.cjs

# Plugin hooks
grep -nE 'catch|exit 0' plugin/scripts/version-check.js plugin/scripts/bun-runner.js
```

### 已知计数（来自本计划内置的初始审计）

- `src/npx-cli/commands/install.ts`：**14** 个 catch 块（第 387、393、406、455