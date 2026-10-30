# npx 安装器注册捕获（非 TTY 默认、延迟配对、轮询预算、分类、遥测）

分支：`fix/npx-signup-capture`。实测结果：`/workspace/npx-signup-metrics/FINDINGS.md`。

一句话概括问题：AI agent 在非 TTY shell 中运行 `npx claude-mem install`，触发缺少 provider 的中止，再用 `--provider claude` 重跑，而该路径跳过 OAuth，因此每天约 330 次安装结束时没有邮箱。另外，CLI 在 4 分钟时停止轮询，而服务端配对存活 30 分钟（16% 的 OAuth 用户看到超时），且 `oauth/start` 失败不发出任何事件。

已定决策（不再重新讨论）：

- 仅在**全新配置**上，非 TTY 默认使用 claude；否则复用已持久化的 provider。
- 延迟配对打印**仅登录**的 `authorizationUrl`，绝不打印 `checkoutUrl`。
- 延迟配对在 `CI` 下**跳过**，且**不**与 `DO_NOT_TRACK` 绑定。
- 延迟配对发送 `source: 'npx-installer-deferred'`；服务端目前忽略 `body.source`，将单独加入白名单。不要因此阻塞。
- 30 分钟 TTL 后，认领路由会跳转到 `/login`，因此不打印回退 URL。
- 交互式 `--provider claude` 和 grok-bot 隐式 `cmem` 路径保持不变。
- 向 `install_completed` 添加 `provider_source`。

构建机的内存约束：**一次只运行一个**测试文件，绝不运行整个套件。

---

## 阶段 0：文档发现（已整合，来自直接阅读）

已读来源：`src/npx-cli/commands/install.ts`、`src/npx-cli/index.ts`、
`src/npx-cli/installer-provider-choice.ts`、`src/npx-cli/cmem-pro-costs.ts`、
`src/npx-cli/cmem-memory-credentials.ts`、`src/npx-cli/install/error-taxonomy.ts`、
`src/npx-cli/install/error-reporter.ts`、`src/services/telemetry/cli-telemetry.ts`、
`src/services/telemetry/scrub.ts`、`docs/public/telemetry.mdx`、`docs/public/installation.mdx`、
`docs/public/cmem-pro-headless.mdx`、`tests/npx-cli/*.test.ts`、`tests/install-non-tty.test.ts`、
`tests/install-error-matrix.test.ts`。

### 允许使用的 API（现已存在；照抄，不要自创）

| 符号 | 位置 | 说明 |
|---|---|---|
| `isInteractive` | `install.ts:58` | `process.stdin.isTTY === true`，模块常量 |
| `log.info/success/warn/error` | `install.ts:161-164` | 感知 TTY 的日志器；所有输出都用它 |
| `getSetting(key)` | `install.ts:54` | 读取带默认值的持久化设置 |
| `readPersistedInstallerSettings()` | 用于 `install.ts:1081` | 原始持久化设置对象 |
| `mergeSettings(obj)` | 全文使用 | 原子写入设置，返回 bool |
| `installerError(severity, ctx, summary)` | `install/error-reporter.ts` | ABORT 抛出 `InstallAbortError` |
| `classifyError(cause, {component, phase})` | `install/error-taxonomy.ts` | 首个匹配生效；兜底最后 |
| `ERROR_CATEGORIES` | `install/error-taxonomy.ts` | 有序数组，元素为 `{id, severity, match, remediation}` |
| `captureCliEvent(event, props, {person?})` | `services/telemetry/cli-telemetry.ts:22` | 从不抛出；受同意门控；props 会被清洗 |
| `ALLOWED_PROPERTY_KEYS` | `services/telemetry/scrub.ts:10` | 允许的 prop 名集合；未知键被丢弃 |
| `startInstallerOAuthPairing()` | `install.ts:1457` | POST start；返回 `InstallerOAuthPairing \| null` |
| `parseInstallerOAuthStartBody(body)` | `install.ts:1399` | 纯解析器，已导出，有契约测试 |
| `InstallerOAuthPairing` | `install.ts:1340` | `{pairingId, secret, userCode, authorizationUrl, checkoutUrl, pollIntervalMs, delivered?}` |
| `waitForInstallerPairing(pairing, phase, version)` | `install.ts:1600` | 轮询循环；预算为 `:1334` 处的 `OAUTH_POLL_BUDGET_MS` |
| `requireInstallerOAuthLogin(version)` | `install.ts:1740` | 阻塞式登录；目前是 start 的唯一调用方 |
| `providerNeedsAccount(provider)` | `install.ts:1855` | `provider !== 'claude' && provider !== 'host'` |
| `validateNonInteractiveProvider(options, summary)` | `install.ts:1914` | 要替换的中止逻辑 |
| `promptProvider(options, pairing, version)` | `install.ts:1070` | 返回 `ProviderId`；claude 分支调用 `useSubscriptionAuth()` |
| `buildAnthropicMaxLocalSettings(settings)` | `cmem-memory-credentials.ts:308` | **清空 cloud-sync token/user/hub 和 Pro memory key** |
| `resolveInstallerProviderChoice({ide, provider})` | `installer-provider-choice.ts` | grok-bot 非 TTY -> `'cmem'` 哨兵值，在 `index.ts` 中应用 |
| `CMEM_INSTALLER_OAUTH_START_URL` | `cmem-pro-costs.ts` | start 端点 |

### 影响设计的事实

- `index.ts` 在 `install.ts` 运行**之前**将非 TTY `--ide grok-bot` 解析为 `provider: 'cmem'`。
  因此在 `install.ts` 中，grok-bot 非 TTY 永远不会出现 `options.provider === undefined`。
  将所有新行为以 `options.provider === undefined`（默认）或
  `!providerNeedsAccount(options.provider)`（延迟配对）为键，可让 grok-bot 不受影响。
- `npx claude-mem update` 以**无 options** 调用 `runInstallCommand()`（`index.ts`，case
  `'update'`）。因此非 TTY update 今天也会触发同样的中止。若采用朴素默认，它会调用
  `useSubscriptionAuth()` 并清空 Pro 用户的 cloud sync。因此"仅全新配置"。
- 目前超时事件发送 `phase`，但 `phase` **不在** `ALLOWED_PROPERTY_KEYS` 中，
  因此被静默丢弃。`stage`、`outcome`、`version`、`duration_ms`、`interactive` 是允许的。
- `captureCliEvent` 不校验事件名；只有属性键会被清洗。
- `parseInstallerOAuthStartBody` 忽略未知字段；`expires_in` 可安全地可选添加。
- 轮询循环在 404/410 时返回 `{kind:'gone'}`，因此轮询到服务端 TTL 是安全的。
- 非 TTY 退出码：仅当 IDE 失败时才 `process.exitCode = 1`（`install.ts:~2437`）。
  延迟配对绝不能触碰退出状态。
- `promptTelemetryOptIn()` 已在 `process.env.CI` 下跳过（`install.ts:1818`）；照抄该门控。

### 固定源码字符串的测试（必须保留或有意更新）

| 测试 | 固定的字符串 / 顺序 | 影响 |
|---|---|---|
| `tests/npx-cli/provider-account-gate.test.ts` | `return provider !== 'claude' && provider !== 'host';`、`if (providerNeedsAccount(options.provider)) {`、正则 `providerNeedsAccount(...) {\n oauthPairing = await requireInstallerOAuthLogin(version);`、`if (options.provider !== 'gemini' && options.provider !== 'openrouter') return;`、`throw new Error('CMEM Pro requires a signed-in claude-mem account.');` | **全部逐字保留** |
| `tests/npx-cli/install-trial-contract.test.ts` | 首个 `indexOf('await requireInstallerOAuthLogin(version)')` < 首个 `indexOf('await promptProvider(options, oauthPairing, version)')`；`p.multiselect<ProviderChoice>`；对解析后的 start body 做 `toEqual` | 保留顺序；**为新字段更新 `toEqual`** |
| `tests/npx-cli/grok-bot-install-default.test.ts` | `initialValues: ['cmem']`、`'provider-cutover'`、index.ts 导入 | 保留 |
| `tests/install-non-tty.test.ts:96-102` | OAuth 调用前的 `validateNonInteractiveProvider(options, summary)`；精确字符串 `A provider must be explicit when stdin is not interactive.` | **重写此测试** |
| `tests/install-error-matrix.test.ts:75-82` | 组件 `'mystery'` -> `unknown-install-error` | 不受影响；添加用例 |

### 需避免的反模式

- 不要把 claude 默认放进 `index.ts`；它会与 grok-bot 解析冲突。
- 不要从延迟路径调用 `requireInstallerOAuthLogin`（测试顺序 + 它会阻塞）。
- 不要在延迟路径打印 `checkoutUrl` 或设备码。
- 不要在非 TTY 路径上添加任何新的 `p.*` 提示。
- 不要发送不在 `ALLOWED_PROPERTY_KEYS` 中的遥测 props；先添加键。
- 不要用 `DO_NOT_TRACK` 门控配对；只有遥测被门控，且 `captureCliEvent` 已处理。
- 不要不带文件参数运行 `bun test`。

---

## 阶段 1：全新配置上的 provider 默认 + 复用已持久化 provider

**文件：** `src/npx-cli/commands/install.ts`（`validateNonInteractiveProvider`、`promptProvider`、`InstallOptions`）、`src/npx-cli/index.ts`（仅帮助文本）。

1. 向 `InstallOptions` 添加 `providerSource?: 'flag' | 'default' | 'persisted' | 'prompt'`
   （`install.ts:1858`）。`index.ts` 在给出 `--provider` 时设置 `providerSource: 'flag'`
   （grok-bot 隐式 `cmem` 也设为 `'flag'`；它是显式的产品默认）。
2. 在 `validateNonInteractiveProvider`（`install.ts:1914`）中，替换 `!options.provider` 中止：
   - 读取 `persisted = readPersistedInstallerSettings()`。
   - 若 `persisted.CLAUDE_MEM_PROVIDER` 是有效的 `ProviderId`（`'claude' | 'gemini' | 'openrouter'`），
     设置 `options.provider = that`、`options.providerSource = 'persisted'`，并
     `log.info('Non-interactive run: keeping the configured provider (<id>).')`。
     然后在 gemini/openrouter key 检查**之前** `return`（持久化的 key 已存在，
     且持久化的 Pro 配置绝不能触发 `configuredCmemKey` 中止）。
   - 否则设置 `options.provider = 'claude'`、`options.providerSource = 'default'`，并
     `log.info('No --provider given on a non-interactive run: defaulting to your Anthropic plan (local memory).')`。
   - 对显式 flag 情况保留现有的 gemini/openrouter key 校验不变
     （固定行 `if (options.provider !== 'gemini' && options.provider !== 'openrouter') return;` 必须逐字保留）。
3. 在 `promptProvider`（`install.ts:1108`）中：当 `options.providerSource === 'persisted'` 时，
   返回 `options.provider`，**不**调用 `useSubscriptionAuth()`、`persistClaudeProvider()`
   或任何 `mergeSettings`。将此检查放在最前，在 `if (options.provider)` 分支之前。
   注释说明原因：持久化的 Pro 配置必须在非 TTY update 运行中存活。
4. 交互流程：多选运行后 `providerSource = 'prompt'`。无其他改动。
5. `index.ts` 帮助：将 `--provider` 行改为说明它在非交互下可选，且在全新安装时默认
   为 `claude`。
6. 更新账户门控上方的注释块（`install.ts:~2205-2213`），使其不再
   说 `--provider claude` "never touches cmem.ai"（阶段 2 会改变）。

**验证**
- `grep -n "A provider must be explicit" src/npx-cli/commands/install.ts` -> 无命中。
- `grep -n "providerSource" src/npx-cli/commands/install.ts src/npx-cli/index.ts` -> 在 index.ts 中设置，在 validate + promptProvider + install_completed（阶段 5）中读取。
- `bun test tests/npx-cli/provider-account-gate.test.ts` 原样通过。
- `bun test tests/npx-cli/grok-bot-install-default.test.ts` 原样通过。
- `tests/install-non-tty.test.ts` 在阶段 6 更新，在此之前不得运行。

**反模式防护：** 不添加 `p.select`/`p.confirm`；持久化分支上不调用 `mergeSettings`；grok-bot `'cmem'` 绝不进入默认分支（在阶段 6 用测试断言）。

---

## 阶段 2：非交互安装末尾的延迟仅登录配对

**文件：** `src/npx-cli/commands/install.ts`（新辅助函数 + 调用点）、`src/npx-cli/cmem-pro-costs.ts`（无需改动；复用 `CMEM_INSTALLER_OAUTH_START_URL`）。

1. 给 `startInstallerOAuthPairing` 添加 options 参数：`startInstallerOAuthPairing(opts?: { source?: string })`，
   默认 source 为 `'npx-installer'`。Body 变为 `{ source, device_name: hostname() }`。现有调用方不变。
2. 在 `requireInstallerOAuthLogin` 旁添加 `async function offerDeferredLogin(options: InstallOptions, version: string): Promise<void>`：
   - 若 `isInteractive`、`providerNeedsAccount(options.provider)` 或
     `process.env.CI` 已设置，立即返回（照抄 `promptTelemetryOptIn` 的精确门控，`install.ts:1819-1820`）。
   - `const pairing = await startInstallerOAuthPairing({ source: 'npx-installer-deferred' })`。
   - 若为 `null`：发出 start 失败事件（阶段 5）并静默返回（无日志行，退出状态不变）。
   - 成功时，用 `log.info` 按以下顺序打印，且**位于安装输出的最后**：
     ```
     Optional: sign in to claude-mem to unlock cloud sync and the CMEM Pro trial.
     Sign-in link: <pairing.authorizationUrl>
     AGENT: show this link to the user so they can finish signing in. Do not open it yourself; the install is already complete.
     ```
     然后 `await captureCliEvent('installer_oauth_deferred', { version, interactive: false })`。
   - 绝不轮询。绝不打印 `userCode` 或 `checkoutUrl`。
3. 调用点：在 `runInstallCommandInner` 中，非 TTY "Next Steps" 块之后、`console.log('\nclaude-mem installed successfully!')` / 失败分支之后，且在
   `captureCliEvent('install_completed', ...)`（`install.ts:~2445`）**之前**。用 `try {} catch {}` 包裹，
   使此处任何内容都不能改变退出状态或在摘要之后抛出。
4. 保持阶段 0 的固定源码字符串完整；新辅助函数不得包含文本
   `await requireInstallerOAuthLogin(version)` 或 `await promptProvider(options, oauthPairing, version)`。

**验证**
- `grep -n "npx-installer-deferred" src/npx-cli/commands/install.ts` -> 辅助函数中一处命中。
- `grep -n "checkoutUrl\|userCode" src/npx-cli/commands/install.ts` -> `offerDeferredLogin` 内无新命中。
- `grep -n "offerDeferredLogin" src/npx-cli/commands/install.ts` -> 定义 + 一个调用点，位于 `install_completed` 之前。
- `bun test tests/npx-cli/install-trial-contract.test.ts` 仍通过顺序测试。

**反模式防护：** 无轮询循环、无 `process.exit`、无 `process.exitCode` 写入、辅助函数中不读取 `DO_NOT_TRACK`、无 stdin 监听器。

---

## 阶段 3：基于 `expires_in` 的轮询预算

**文件：** `src/npx-cli/commands/install.ts`（`InstallerOAuthPairing`、`parseInstallerOAuthStartBody`、`waitForInstallerPairing`）。

1. 在 `OAUTH_POLL_BUDGET_MS`（`install.ts:1334`）旁添加常量：保留 `OAUTH_POLL_BUDGET_MS = 240_000`
   作为回退，并添加 `OAUTH_POLL_BUD