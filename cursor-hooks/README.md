# Claude-Mem Cursor Hooks 集成

> **为 Cursor 提供持久化 AI 记忆 — 不占用套餐额度运行**

让 Cursor AI 跨会话拥有持久记忆。你的智能体自动记住做过的工作、做出的决策以及代码库中的模式。

### 为什么选择 Claude-Mem？

- **跨会话记住上下文**：无需每次重新解释代码库
- **自动捕获**：MCP 工具、shell 命令和文件编辑自动记录，无需手动操作
- **不占用套餐额度的提供商选项**：支持你的 Gemini 密钥或 OpenRouter，包括免费可用的模型
- **无论是否使用 Claude Code 均可工作**：两种方式功能完整

### 快速安装（5 分钟）

```bash
# Clone and build
git clone https://github.com/thedotmack/claude-mem.git
cd claude-mem && bun install && bun run build

# Interactive setup (configures provider + installs hooks)
bun run cursor:setup
```

---

## Cursor 用户快速开始

**使用 Claude Code？** 跳到[安装](#installation) — 一切自动生效。

**仅使用 Cursor（无 Claude Code）？** 参见 [STANDALONE-SETUP.md](STANDALONE-SETUP.md)，了解使用 Gemini 或 OpenRouter 的不占用套餐额度配置。

---

## 概述

这些 Hook 将 Cursor 的 Hook 系统桥接到 claude-mem 的 worker API，支持：
- **会话管理**：初始化会话并生成摘要
- **观察捕获**：记录 MCP 工具使用、shell 命令和文件编辑
- **Worker 就绪**：确保提交提示前 worker 正在运行

## 上下文注入

上下文通过 Cursor 的 **Rules** 系统自动注入：

1. **安装**：`claude-mem cursor install` 生成初始上下文
2. **Stop Hook**：每次会话结束后更新 `.cursor/rules/claude-mem-context.mdc` 中的上下文
3. **Cursor**：自动在所有聊天会话中包含此规则

**上下文在每次会话结束后更新**，因此下次会话能看到最新上下文。

### 其他访问方式

- **MCP 工具**：配置 claude-mem 的 MCP 服务器以使用 `search`、`timeline`、`get_observations` 工具
- **Web 查看器**：在 `http://localhost:37777` 访问上下文
- **手动请求**：让智能体搜索记忆

✨ **30 天免费试用 — 记忆不占用套餐额度。让你的套餐用量提升最多 100%** → https://cmem.ai/pro?from=docs

详见 [CONTEXT-INJECTION.md](CONTEXT-INJECTION.md)。

## 安装

### 快速安装（推荐）

```bash
# Install globally for all projects (recommended)
claude-mem cursor install user

# Or install for current project only
claude-mem cursor install
```

### 手动安装

<details>
<summary>点击展开手动安装步骤</summary>

**用户级**（推荐 — 适用于所有项目）：
```bash
# Copy hooks.json to your home directory
cp cursor-hooks/hooks.json ~/.cursor/hooks.json

# Copy hook scripts
mkdir -p ~/.cursor/hooks
cp cursor-hooks/*.sh ~/.cursor/hooks/
chmod +x ~/.cursor/hooks/*.sh
```

**项目级**（按项目配置 Hook）：
```bash
# Copy hooks.json to your project
mkdir -p .cursor
cp cursor-hooks/hooks.json .cursor/hooks.json

# Copy hook scripts to your project
mkdir -p .cursor/hooks
cp cursor-hooks/*.sh .cursor/hooks/
chmod +x .cursor/hooks/*.sh
```

</details>

### 安装之后

1. **启动 worker**：
   ```bash
   claude-mem start
   ```

2. **重启 Cursor** 以加载 Hook

3. **验证安装**：
   ```bash
   claude-mem cursor status
   ```

## Hook 映射

| Cursor Hook | 脚本 | 用途 |
|-------------|--------|---------|
| `beforeSubmitPrompt` | `session-init.sh` | 初始化 claude-mem 会话 |
| `beforeSubmitPrompt` | `context-inject.sh` | 确保 worker 正在运行 |
| `afterMCPExecution` | `save-observation.sh` | 捕获 MCP 工具使用 |
| `afterShellExecution` | `save-observation.sh` | 捕获 shell 命令执行 |
| `afterFileEdit` | `save-file-edit.sh` | 捕获文件编辑 |
| `stop` | `session-summary.sh` | 生成摘要 + 更新上下文文件 |

## 工作原理

### 会话初始化（`session-init.sh`）
- 每次提交提示前调用
- 使用 `conversation_id` 作为会话 ID 在 claude-mem 中初始化新会话
- 从工作区根目录提取项目名
- 输出 `{"continue": true}` 以允许提交提示

### 上下文 Hook（`context-inject.sh`）
- 确保会话前 claude-mem worker 正在运行
- 输出 `{"continue": true}` 以允许提交提示
- 注意：上下文文件由 `session-summary.sh`（stop hook）更新，不在此处更新

### 观察捕获（`save-observation.sh`）
- 捕获 MCP 工具执行和 shell 命令
- 映射为 claude-mem 的观察格式
- 发送到 `/api/sessions/observations` 端点（即发即忘）

### 文件编辑捕获（`save-file-edit.sh`）
- 捕获智能体所做的文件编辑
- 将编辑视为 "write_file" 工具使用
- 在观察中包含编辑摘要

### 会话摘要（`session-summary.sh`）
- 智能体循环结束时调用（stop hook）
- 请求 claude-mem 生成摘要
- **更新上下文文件** `.cursor/rules/claude-mem-context.mdc` 供下次会话使用

## 配置

Hook 从 `~/.claude-mem/settings.json` 读取配置：

- `CLAUDE_MEM_WORKER_PORT`：Worker 端口（默认：37777）
- `CLAUDE_MEM_WORKER_HOST`：Worker 主机（默认：127.0.0.1）

## 依赖

Hook 脚本需要：
- `jq` — JSON 处理
- `curl` — HTTP 请求
- `bash` — Shell 解释器

macOS 安装：`brew install jq curl`
Ubuntu 安装：`apt-get install jq curl`

## 故障排查

### Hook 未执行

1. 检查 Hook 是否在正确位置：
   ```bash
   ls .cursor/hooks.json  # Project-level
   ls ~/.cursor/hooks.json  # User-level
   ```

2. 确认脚本可执行：
   ```bash
   chmod +x ~/.cursor/hooks/*.sh
   ```

3. 检查 Cursor 设置 → Hooks 标签页中的配置状态

4. 检查 Cursor 中的 Hooks 输出通道是否有错误信息

### Worker 无响应

1. 确认 worker 正在运行：
   ```bash
   curl http://127.0.0.1:37777/api/readiness
   ```

2. 检查 worker 日志：
   ```bash
   tail -f ~/.claude-mem/logs/worker-$(date +%Y-%m-%d).log
   ```

3. 重启 worker：
   ```bash
   claude-mem restart
   ```

### 观察未被保存

1. 监控 worker 日志中的传入请求

2. 通过 Web 查看器 `http://localhost:37777` 确认会话已初始化

3. 直接测试观察端点：
   ```bash
   curl -X POST http://127.0.0.1:37777/api/sessions/observations \
     -H "Content-Type: application/json" \
     -d '{"contentSessionId":"test","tool_name":"test","tool_input":{},"tool_response":{},"cwd":"/tmp"}'
   ```

## 与 Claude Code 集成对比

| 功能 | Claude Code | Cursor |
|---------|-------------|--------|
| 会话初始化 | ✅ `SessionStart` hook | ✅ `beforeSubmitPrompt` hook |
| 上下文注入 | ✅ `additionalContext` 字段 | ✅ 自动更新 `.cursor/rules/` 文件 |
| 观察捕获 | ✅ `PostToolUse` hook | ✅ `afterMCPExecution`、`afterShellExecution`、`afterFileEdit` |
| 会话摘要 | ✅ 带转录的 `Stop` hook | ⚠️ `stop` hook（无转录） |
| MCP 搜索工具 | ✅ 完全支持 | ✅ 完全支持（若已配置 MCP） |

## 文件

- `hooks.json` — Hook 配置
- `common.sh` — 共享工具函数
- `session-init.sh` — 会话初始化
- `context-inject.sh` — 上下文/worker 就绪 Hook
- `save-observation.sh` — MCP 和 shell 观察捕获
- `save-file-edit.sh` — 文件编辑观察捕获
- `session-summary.sh` — 摘要生成
- `cursorrules-template.md` — `.cursorrules` 文件模板

## 另见

- [Claude-Mem 文档](https://docs.claude-mem.ai)
- [Cursor Hooks 参考](../docs/context/cursor-hooks-reference.md)
- [Claude-Mem 架构](https://docs.claude-mem.ai/architecture/overview)