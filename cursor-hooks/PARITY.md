# 功能对齐：Claude-Mem Hooks 与 Cursor Hooks

本文档对比 claude-mem 的 Claude Code hooks 与 Cursor hooks 实现，确保功能对齐。

## Hook 映射

| Claude Code Hook | Cursor Hook | 状态 | 备注 |
|-----------------|-------------|--------|-------|
| `SessionStart` → `context-hook.js` | `beforeSubmitPrompt` → `context-inject.sh` | ✅ 部分 | 已获取上下文，但 Cursor 中无法注入 |
| `SessionStart` → `user-message-hook.js` | （可选）`user-message.sh` | ⚠️ 可选 | 无 SessionStart 等价事件；可在 beforeSubmitPrompt 时运行 |
| `UserPromptSubmit` → `new-hook.js` | `beforeSubmitPrompt` → `session-init.sh` | ✅ 完成 | 会话初始化、隐私检查、斜杠剥离 |
| `PostToolUse` → `save-hook.js` | `afterMCPExecution` + `afterShellExecution` → `save-observation.sh` | ✅ 完成 | 工具观察捕获 |
| `PostToolUse` →（文件编辑） | `afterFileEdit` → `save-file-edit.sh` | ✅ 完成 | 文件编辑观察捕获 |
| `Stop` → `summary-hook.js` | `stop` → `session-summary.sh` | ⚠️ 部分 | 摘要生成（无法访问对话记录） |

## 功能对比

### 1. 会话初始化（`new-hook.js` ↔ `session-init.sh`）

| 功能 | Claude Code | Cursor | 状态 |
|---------|-------------|--------|--------|
| Worker 健康检查 | ✅ 75 次重试（15 秒） | ✅ 75 次重试（15 秒） | ✅ 一致 |
| 会话初始化 API 调用 | ✅ `/api/sessions/init` | ✅ `/api/sessions/init` | ✅ 一致 |
| 隐私检查处理 | ✅ 检查 `skipped` + `reason` | ✅ 检查 `skipped` + `reason` | ✅ 一致 |
| 斜杠剥离 | ✅ 剥离前导 `/` | ✅ 剥离前导 `/` | ✅ 一致 |
| SDK agent 初始化 | ✅ `/sessions/{id}/init` | ❌ 不需要 | ✅ 不适用（Cursor 特有） |

**状态**：✅ 完全对齐（SDK agent 初始化不适用于 Cursor）

### 2. 上下文注入（`context-hook.js` ↔ `context-inject.sh`）

| 功能 | Claude Code | Cursor | 状态 |
|---------|-------------|--------|--------|
| Worker 健康检查 | ✅ 75 次重试 | ✅ 75 次重试 | ✅ 一致 |
| 上下文获取 | ✅ `/api/context/inject` | ✅ `/api/context/inject` | ✅ 一致 |
| 输出格式 | ✅ 带 `hookSpecificOutput` 的 JSON | ✅ 写入 `.cursor/rules/` 文件 | ✅ 替代方案 |
| 项目名提取 | ✅ `getProjectName(cwd)` | ✅ `basename(workspace_root)` | ✅ 一致 |
| 自动刷新 | ✅ 每次会话启动 | ✅ 每次提交提示词 | ✅ 增强 |

**状态**：✅ 通过自动更新的规则文件实现完全对齐

**工作原理**：
- Hook 将上下文写入 `.cursor/rules/claude-mem-context.mdc`
- 该文件带有 `alwaysApply: true` frontmatter
- Cursor 在所有聊天会话中自动包含此规则
- 每次提交提示词时刷新上下文

### 3. 用户消息显示（`user-message-hook.js` ↔ `user-message.sh`）

| 功能 | Claude Code | Cursor | 状态 |
|---------|-------------|--------|--------|
| 带颜色的上下文获取 | ✅ `/api/context/inject?colors=true` | ✅ `/api/context/inject?colors=true` | ✅ 一致 |
| 输出通道 | ✅ stderr | ✅ stderr | ✅ 一致 |
| 显示格式 | ✅ 带 emoji 格式化 | ✅ 带 emoji 格式化 | ✅ 一致 |
| Hook 触发 | ✅ SessionStart | ⚠️ 可选（无 SessionStart） | ⚠️ Cursor 限制 |

**状态**：⚠️ 可选（Cursor 中无 SessionStart 等价事件）

**注意**：如需使用，可添加到 `beforeSubmitPrompt`，但可能过于冗长。

### 4. 观察捕获（`save-hook.js` ↔ `save-observation.sh`）

| 功能 | Claude Code | Cursor | 状态 |
|---------|-------------|--------|--------|
| Worker 健康检查 | ✅ 75 次重试 | ✅ 75 次重试 | ✅ 一致 |
| 工具名提取 | ✅ 从 `tool_name` | ✅ 从 `tool_name` 或 "Bash" | ✅ 一致 |
| 工具输入捕获 | ✅ 完整 JSON | ✅ 完整 JSON | ✅ 一致 |
| 工具响应捕获 | ✅ 完整 JSON | ✅ 完整 JSON 或输出 | ✅ 一致 |
| 隐私标签剥离 | ✅ Worker 处理 | ✅ Worker 处理 | ✅ 一致 |
| 错误处理 | ✅ 即发即忘 | ✅ 即发即忘 | ✅ 一致 |
| Shell 命令映射 | ✅ 不适用（独立 hook） | ✅ 映射到 "Bash" 工具 | ✅ 增强 |

**状态**：✅ 完全对齐（增强支持 shell 命令）

### 5. 文件编辑捕获（不适用 ↔ `save-file-edit.sh`）

| 功能 | Claude Code | Cursor | 状态 |
|---------|-------------|--------|--------|
| 文件路径提取 | 不适用 | ✅ 从 `file_path` | ✅ 新增 |
| 编辑详情 | 不适用 | ✅ 从 `edits` 数组 | ✅ 新增 |
| 工具名 | 不适用 | ✅ "write_file" | ✅ 新增 |
| 编辑摘要 | 不适用 | ✅ 从编辑生成 | ✅ 新增 |

**状态**：✅ 新功能（Cursor 特有，Claude Code 中无）

### 6. 会话摘要（`summary-hook.js` ↔ `session-summary.sh`）

| 功能 | Claude Code | Cursor | 状态 |
|---------|-------------|--------|--------|
| Worker 健康检查 | ✅ 75 次重试 | ✅ 75 次重试 | ✅ 一致 |
| 对话记录解析 | ✅ 提取最后消息 | ❌ 无法访问对话记录 | ⚠️ Cursor 限制 |
| 摘要 API 调用 | ✅ `/api/sessions/summarize` | ✅ `/api/sessions/summarize` | ✅ 一致 |
| 最后消息提取 | ✅ 从对话记录 | ❌ 空字符串 | ⚠️ Cursor 限制 |
| 错误处理 | ✅ 即发即忘 | ✅ 即发即忘 | ✅ 一致 |

**状态**：⚠️ 部分对齐（Cursor 中无法访问对话记录）

**注意**：摘要生成仍可工作，但缺少最后消息可能降低准确性。Worker 根据会话期间存储的观察生成摘要。

## 实现细节

### Worker 健康检查
- **Claude Code**：75 次重试 × 200ms = 15 秒
- **Cursor**：75 次重试 × 200ms = 15 秒
- **状态**：✅ 一致

### 错误处理
- **Claude Code**：即发即忘并记录日志
- **Cursor**：即发即忘并优雅退出（exit 0）
- **状态**：✅ 一致（适配 Cursor 的 hook 系统）

### 隐私处理
- **Claude Code**：Worker 执行隐私检查，hooks 遵循 `skipped` 标志
- **Cursor**：Worker 执行隐私检查，hooks 遵循 `skipped` 标志
- **状态**：✅ 一致

### 标签剥离
- **Claude Code**：Worker 处理 `<private>` 和 `<claude-mem-context>` 标签
- **Cursor**：Worker 处理标签（hooks 无需剥离）
- **状态**：✅ 一致

## 缺失功能（Cursor 限制）

1. ~~**直接上下文注入**~~：**已解决**，通过自动更新的规则文件
   - Hook 将上下文写入 `.cursor/rules/claude-mem-context.mdc`
   - Cursor 自动包含带 `alwaysApply: true` 的规则
   - 每次提示词时刷新上下文

2. **对话记录访问**：Cursor hooks 不提供对话记录路径
   - **影响**：摘要生成准确性降低
   - **变通方案**：Worker 根据观察生成

3. **SessionStart Hook**：Cursor 没有会话启动事件
   - **影响**：用户消息显示必须设为可选
   - **变通方案**：如需使用，可在 `beforeSubmitPrompt` 时运行

4. **SDK Agent 会话**：Cursor 不使用 SDK agent 模式
   - **影响**：无需调用 `/sessions/{id}/init`
   - **状态**：✅ 不适用（Cursor 特有）

## 增强功能（Cursor 特有）

1. **Shell 命令捕获**：将 shell 命令映射到 "Bash" 工具观察
   - **状态**：✅ 超越 Claude Code 的增强

2. **文件编辑捕获**：专用于文件编辑的 hook
   - **状态**：✅ 新功能

3. **MCP 工具捕获**：单独捕获 MCP 工具使用
   - **状态**：✅ 超越 Claude Code 的增强

## 总结

| 类别 | 状态 |
|----------|--------|
| 核心功能 | ✅ 完全对齐 |
| 会话管理 | ✅ 完全对齐 |
| 观察捕获 | ✅ 完全对齐（增强） |
| 上下文注入 | ✅ 完全对齐（通过规则文件） |
| 摘要生成 | ⚠️ 部分（无对话记录） |
| 用户体验 | ⚠️ 部分（无 SessionStart） |

**总体**：Cursor hooks 实现与 claude-mem 的 Claude Code hooks 达到**完全功能对齐**：
- ✅ 会话初始化
- ✅ 上下文注入（通过自动更新的 `.cursor/rules/` 文件）
- ✅ 观察捕获（MCP 工具、shell 命令、文件编辑）
- ⚠️ 摘要生成（可工作，但无法访问对话记录）