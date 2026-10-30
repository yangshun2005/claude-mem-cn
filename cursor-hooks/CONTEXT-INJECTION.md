# Cursor Hooks 中的上下文注入

## 解决方案：自动更新的规则文件

上下文通过 Cursor 的 **Rules** 系统自动注入：

1. **安装**：`claude-mem cursor install` 创建初始上下文文件
2. **Stop Hook**：`session-summary.sh` 在每次会话结束后更新上下文
3. **Cursor**：自动在所有对话中包含 `.cursor/rules/claude-mem-context.mdc`

**结果**：上下文出现在每次对话开头，与 Claude Code 一致！

## 工作原理

### 安装时创建初始上下文

```bash
claude-mem cursor install
```

该命令会：
1. 将 Hook 脚本复制到 `.cursor/hooks/`
2. 创建 `hooks.json` 配置
3. 从 claude-mem 获取现有上下文并写入 `.cursor/rules/claude-mem-context.mdc`

### 上下文在三个时机更新

每次会话中上下文刷新**三次**，确保最新：

1. **提交提示词前**（`context-inject.sh`）：确保你以之前会话的最新上下文开始
2. **摘要完成后**（worker 自动更新）：摘要保存后，worker 立即更新上下文文件
3. **会话结束后**（`session-summary.sh`）：兜底更新，防止 worker 更新遗漏

### 提交提示词前的 Hook 更新上下文

提交提示词时，`context-inject.sh` 执行：

```bash
# 1. Ensure worker is running
ensure_worker_running "$worker_port"

# 2. Fetch fresh context
context=$(curl -s ".../api/context/inject?project=...")

# 3. Write to rules file (used immediately by Cursor)
cat > .cursor/rules/claude-mem-context.mdc << EOF
---
alwaysApply: true
---
# Memory Context
${context}
EOF
```

### Stop Hook 更新上下文

每次会话结束后，`session-summary.sh` 执行：

```bash
# 1. Generate session summary
curl -X POST .../api/sessions/summarize

# 2. Fetch fresh context (includes new observations)
context=$(curl -s ".../api/context/inject?project=...")

# 3. Write to rules file for next session
cat > .cursor/rules/claude-mem-context.mdc << EOF
---
alwaysApply: true
---
# Memory Context
${context}
EOF
```

### 规则文件

位置：`.cursor/rules/claude-mem-context.mdc`

```markdown
---
alwaysApply: true
description: "Claude-mem context from past sessions (auto-updated)"
---

# Memory Context from Past Sessions

[Your context from claude-mem appears here]

---
*Updated after last session.*
```

### 更新流程

上下文在**三个时机**更新：

**每次提示词前：**
1. 用户提交提示词
2. `beforeSubmitPrompt` Hook 运行 `context-inject.sh`
3. 上下文文件刷新为之前会话的最新观察
4. Cursor 读取更新后的规则文件

**摘要完成后（worker 自动更新）：**
1. 摘要保存到数据库
2. Worker 检查项目是否已注册到 Cursor
3. 若已注册，立即写入包含新观察的上下文文件
4. 不涉及 Hook——在 worker 进程中完成

**会话结束后（兜底）：**
1. Agent 完成（循环结束）
2. `stop` Hook 运行 `session-summary.sh`
3. 更新上下文文件（确保无遗漏）
4. 为下次会话做好准备

## 项目注册表

运行 `claude-mem cursor install` 时，项目会注册到 `~/.claude-mem/cursor-projects.json`。这样，每当生成新摘要时，worker 都能自动更新你的上下文文件——即使摘要来自 Claude Code 或另一个 IDE 处理同一项目。

查看已注册项目：
```bash
cat ~/.claude-mem/cursor-projects.json
```

## 与 Claude Code 对比

| 功能 | Claude Code | Cursor |
|---------|-------------|--------|
| 上下文注入 | ✅ Hook 输出中的 `additionalContext` | ✅ 自动更新的规则文件 |
| 注入时机 | 即时（同一提示词） | 提示词前 + 摘要后 + 会话后 |
| 持久性 | 仅会话内 | 基于文件（重启后仍保留） |
| 初始设置 | 自动 | `claude-mem cursor install` 创建初始上下文 |
| MCP 工具访问 | ✅ 完全支持 | ✅ 完全支持 |
| Web 查看器 | ✅ 可用 | ✅ 可用 |

## 首次会话行为

运行 `claude-mem cursor install` 时：
- 若 worker 正在运行且已有记忆 → 生成初始上下文
- 若无现有记忆 → 创建占位文件

之后上下文会自动刷新：
- 每次提示词前（确保包含最新观察）
- 每次会话结束后（捕获本次会话的新观察）

## 其他访问方式

### 1. MCP 工具

在 Cursor 中配置 claude-mem 的 MCP 服务器以使用搜索工具：
- `search(query, project, limit)`
- `timeline(anchor, depth_before, depth_after)`
- `get_observations(ids)`

### 2. Web 查看器

在 `http://localhost:37777` 手动访问上下文

### 3. 手动请求

向 Agent 提问："Check claude-mem for any previous work on authentication"

## 文件位置

上下文文件创建于：
```
<workspace>/.cursor/rules/claude-mem-context.mdc
```

该文件默认纳入版本控制。若不想提交，添加到 `.gitignore`：
```
.cursor/rules/claude-mem-context.mdc
```