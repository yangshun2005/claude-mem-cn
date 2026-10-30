# 安全策略

## 支持的版本

只有最新发布的 `claude-mem` 版本会收到安全更新。报告漏洞前请先升级到最新版本。

| 版本 | 支持 |
| ------- | ------------------ |
| 最新  | :white_check_mark: |
| 旧版本   | :x:                |

## 报告漏洞

如果你发现 claude-mem 的安全漏洞，请按以下方式报告：

1. **不要**创建公开的 GitHub Issue、Pull Request 或 Discussion
2. 发送邮件至 **alex@cmem.ai** 说明详情，或在 Security 标签页使用 GitHub 的"Report a vulnerability"按钮开启私密安全公告
3. 尽可能附上复现步骤、影响评估、受影响版本和建议修复方案

**范围：** 本策略覆盖 `claude-mem` 插件及其捆绑组件（Hook、worker 服务、SQLite/Chroma 同步、查看器 UI、搜索/规划技能）。上游依赖的问题应直接报告给对应项目，但也欢迎告知我们。

我们重视安全，会在 48 小时内确认有效报告，并争取在下一个发布版本中修复。

## 安全措施

### 命令注入防护

Claude-mem 会执行系统命令进行 git 操作和进程管理。我们已实施全面的命令注入防护：

#### 安全命令执行
- **数组参数：** 所有命令使用数组参数，防止 shell 解释
- **不经过 Shell：** 所有涉及用户输入的 spawn 操作都显式设置 `shell: false`
- **输入校验：** 所有用户可控参数在使用前都会校验

#### 安全模式示例
```typescript
// ✅ SAFE: Array-based arguments with validation
if (!isValidBranchName(userInput)) {
  throw new Error('Invalid input');
}
spawnSync('git', ['checkout', userInput], { shell: false });

// ❌ UNSAFE: Never do this
execSync(`git checkout ${userInput}`);
```

### 输入校验

所有用户可控输入都通过白名单和严格模式校验：

- **分支名：** 必须匹配 `/^[a-zA-Z0-9][a-zA-Z0-9._/-]*$/`，且不能包含 `..`
- **端口号：** 必须为数字且在 1024-65535 范围内
- **文件路径：** 所有路径使用 `path.join()` 拼接，防止路径穿越

### 进程管理

- **PID 文件保护：** 进程 ID 存储在用户数据目录（`~/.claude-mem/`）
- **端口校验：** worker 端口在绑定前校验
- **健康检查：** 处理请求前验证 worker 健康状态

### 隐私控制

Claude-mem 内置双标签系统保护内容隐私：

- `<private>content</private>` - 用户级隐私（阻止存储）
- `<claude-mem-context>content</claude-mem-context>` - 系统级标签（阻止递归存储）

标签在 Hook 层剥离，数据到达 worker/数据库前即被移除。

## 安全审计历史

### 2025-12-16：命令注入漏洞（Issue #354）
- **严重程度：** 严重
- **状态：** 已解决
- **受影响版本：** 修复前的所有版本
- **修复版本：** 当前版本
- **发现漏洞数：** 3
- **已修复漏洞数：** 3

**修复摘要：**
1. 将 `BranchManager.ts` 中的字符串插值替换为数组参数
2. 新增 `isValidBranchName()` 校验函数
3. 移除 `bun-path.ts` 中不必要的 shell 使用
4. 创建全面的安全测试套件

## 贡献者安全最佳实践

### 添加命令执行时

1. **绝不对用户输入使用 shell：**
   ```typescript
   // ❌ NEVER
   execSync(`command ${userInput}`);
   spawn('command', [...], { shell: true });

   // ✅ ALWAYS
   spawnSync('command', [userInput], { shell: false });
   ```

2. **始终校验用户输入：**
   ```typescript
   if (!isValidInput(userInput)) {
     throw new Error('Invalid input');
   }
   ```

3. **使用数组参数：**
   ```typescript
   // ❌ NEVER
   execSync(`git ${command} ${arg}`);

   // ✅ ALWAYS
   spawnSync('git', [command, arg], { shell: false });
   ```

4. **显式设置 shell: false：**
   ```typescript
   spawnSync('command', args, { shell: false });
   ```

### 添加用户输入时

1. **白名单校验**优先于黑名单
2. **严格正则模式**校验格式
3. **类型检查**校验预期数据类型
4. **范围校验**校验数值输入
5. **长度限制**限制字符串输入

### 代码审查清单

提交涉及命令执行或用户输入处理的 PR 前：

- [ ] 没有使用字符串插值或模板字面量的 `execSync`
- [ ] 涉及用户输入时没有 `shell: true`
- [ ] 所有 spawn/spawnSync 调用使用数组参数
- [ ] 所有用户可控参数都有输入校验
- [ ] 为新攻击向量添加安全测试
- [ ] 代码遵循上述安全模式

## 依赖

我们定期审计依赖漏洞：

- **npm audit：** 每次发布前运行
- **Dependabot：** 已启用自动安全更新
- **人工审查：** 关键依赖每季度审查

## 数据存储

Claude-mem 将数据存储在本地 `~/.claude-mem/`：

- **数据库：** SQLite3，位于 `~/.claude-mem/claude-mem.db`
- **向量存储：** Chroma，位于 `~/.claude-mem/chroma/`
- **日志：** `~/.claude-mem/logs/`
- **设置：** `~/.claude-mem/settings.json`

所有 claude-mem 状态文件（数据库、向量存储、日志、设置、supervisor 和 PID 文件）都写入本地用户目录。本地存储不代表没有数据离开设备：

- **遥测（默认开启，可退出）：** 使用分析和脱敏后的错误消息及堆栈跟踪会发送至 PostHog，以随机安装 UUID 标识。要同时禁用分析和异常报告，运行 `npx claude-mem telemetry disable` 或设置 `DO_NOT_TRACK=1`。收集字段、设置优先级及独立的错误报告退出方式见[遥测](docs/public/telemetry.mdx)。
- **可选云同步：** 配置后，云同步会将观察记录、会话摘要和用户提示上传至你 cmem.ai 账户下的同步中心。当中心支持 `work_state` 时，还会上传 agent 待办列表和工作状态。设置和隐私详情见[云同步](docs/public/cloud-sync.mdx)。

Claude-mem 还会调用上游模型提供商和可选集成来工作，因此观察/转录/提示内容可能通过这些渠道离开本机：

- **Claude Agent SDK**（默认摘要/观察路径）：将提示和转录上下文发送至 Anthropic 的 API。
- **替代提供商**（`gemini`、`openrouter`）：配置后，将相同上下文发送至这些提供商。
- **`openai-compatible` 提供商**：配置后，将相同上下文发送至 `CLAUDE_MEM_OPENAI_COMPAT_PRESET` / `CLAUDE_MEM_OPENAI_COMPAT_BASE_URL` 指定的任意 OpenAI 兼容端点——第三方主机（NVIDIA NIM、DeepSeek、Groq、Together）或本机服务器（Ollama、LM Studio、vLLM）。与 OpenRouter 路径不同，它不发送归属头，也绝不复用 OpenRouter 或 cmem.ai 网关凭据。
- **Chroma MCP / `chroma-mcp`**：启用后，通过配置的嵌入后端计算嵌入，具体是否为远程 API 取决于用户的 chroma-mcp 配置。
- **OAuth / 钥匙串读取**：claude-mem 在 spawn 时从平台原生凭据存储读取 Claude Code OAuth token。该 token 会注入 worker 子进程，但 claude-mem 不会传输它。
- **GitHub releases / npm registry**：版本检查和自更新流程从公共 registry 获取元数据。

发送敏感内容前，请检查 `~/.claude-mem/settings.json` 和 `~/.claude-mem/.env` 中的提供商/Chroma 配置。使用 `<private>...</private>` 标签将特定内容排除在本地存储之外。

## 权限

Claude-mem 需要：

- **文件系统：** 读写 `~/.claude-mem/` 和 `~/.claude/plugins/`
- **网络：** localhost 上的 HTTP 服务器（默认端口 37777）
- **进程管理：** 启动 worker 进程、管理 PID

无需提权（root/管理员）。

## 安全默认值

- **Worker 主机：** 默认绑定 `127.0.0.1`（仅 localhost）
- **Worker 端口：** 用户可配置，校验范围 1024-65535
- **日志级别：** 默认 INFO（日志中不含敏感数据）
- **隐私标签：** 存储前自动剥离私密内容

## 更新

安全补丁在发现后尽快发布。用户应：

1. 保持 claude-mem 更新至最新版本
2. 关注 GitHub releases 的安全公告
3. 查看 [CHANGELOG.md](./CHANGELOG.md) 中的安全相关变更

## 疑问？

安全相关问题（非漏洞），请：

1. 查看安全关键文件中的代码注释
2. 在 GitHub Discussion（非 Issue）中提出一般安全问题
3. 敏感问题请发送邮件至 **alex@cmem.ai**

---

**最后更新：** 2026-09-09
**最后审计：** 2025-12-16（Issue #354）
**下次计划审计：** 2026-09-16