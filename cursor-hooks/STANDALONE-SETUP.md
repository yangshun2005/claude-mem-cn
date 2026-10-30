# Claude-Mem for Cursor（无需 Claude Code）

> **为 Cursor 提供持久化 AI 记忆 — 不占用套餐额度运行**

## 概述

无需 Claude Code 订阅，即可在 Cursor 中使用 claude-mem 的持久化记忆。选择 Gemini 或 OpenRouter，让记忆处理不占用你的 Claude 套餐额度。

**你将获得**：
- **持久化记忆**跨会话保留 — 你的 AI 记得之前做过什么
- **自动捕获** MCP 工具、shell 命令和文件编辑
- **上下文注入**通过 `.cursor/rules/` — 每次对话都包含相关历史
- **Web 查看器** http://localhost:37777 — 浏览和搜索项目历史

✨ **30 天免费试用 — 记忆不占用套餐额度。让你的套餐用量提升最多 100%** → https://cmem.ai/pro?from=docs

**为什么重要**：每次 Cursor 会话都从零开始。Claude-mem 弥合这一差距 — 你的 AI 代理会逐步积累关于代码库、决策和模式的知识。

## 前置条件

### macOS / Linux
- Cursor IDE
- [Bun](https://bun.sh)（`curl -fsSL https://bun.sh/install | bash`）
- Git
- `jq` 和 `curl`：
  - **macOS**：`brew install jq curl`
  - **Linux**：`apt install jq curl`

### Windows
- Cursor IDE
- [Bun](https://bun.sh)（PowerShell：`powershell -c "irm https://bun.sh/install.ps1 | iex"`）
- Git
- PowerShell 5.1+（Windows 10/11 自带）

## 第 1 步：克隆 Claude-Mem

```bash
# Clone the repository
git clone https://github.com/thedotmack/claude-mem.git
cd claude-mem

# Install dependencies
bun install

# Build the project
bun run build
```

## 第 2 步：配置提供商（任选其一）

由于你没有 Claude Code，需要为 claude-mem 的摘要引擎配置一个 AI 提供商。

### 方案 A：Gemini（推荐 — 无需绑定计费）

Gemini 允许你在不启用计费的情况下创建 API 密钥。请求限额因模型和 Google Cloud 项目而异。

```bash
# Create settings directory
mkdir -p ~/.claude-mem

# Create settings file
cat > ~/.claude-mem/settings.json << 'EOF'
{
  "CLAUDE_MEM_PROVIDER": "gemini",
  "CLAUDE_MEM_GEMINI_API_KEY": "YOUR_GEMINI_API_KEY",
  "CLAUDE_MEM_GEMINI_MODEL": "gemini-2.5-flash-lite",
  "CLAUDE_MEM_GEMINI_RATE_LIMITING_ENABLED": true
}
EOF
```

**创建 API 密钥**：https://aistudio.google.com/apikey

### 方案 B：OpenRouter（100+ 模型）

OpenRouter 提供多种模型的访问，包括免费选项。

```bash
mkdir -p ~/.claude-mem
cat > ~/.claude-mem/settings.json << 'EOF'
{
  "CLAUDE_MEM_PROVIDER": "openrouter",
  "CLAUDE_MEM_OPENROUTER_API_KEY": "YOUR_OPENROUTER_API_KEY"
}
EOF
```

**获取 API 密钥**：https://openrouter.ai/keys

**可用免费模型**（ID 会变动；请查看 https://openrouter.ai/models）：
- `cohere/north-mini-code:free`（claude-mem 默认）
- `poolside/laguna-xs-2.1:free`

### 方案 C：Claude API（如果你有 API 访问权限）

如果你有 Anthropic API 额度但没有 Claude Code 订阅：

```bash
mkdir -p ~/.claude-mem
cat > ~/.claude-mem/settings.json << 'EOF'
{
  "CLAUDE_MEM_PROVIDER": "claude",
  "ANTHROPIC_API_KEY": "YOUR_ANTHROPIC_API_KEY"
}
EOF
```

## 第 3 步：安装 Cursor Hooks

```bash
# From the claude-mem repo directory (recommended - all projects)
bun run cursor:install -- user

# Or for project-level only:
bun run cursor:install
```

安装内容：
- Hook 脚本到 `.cursor/hooks/`
- Hook 配置到 `.cursor/hooks.json`
- 上下文模板到 `.cursor/rules/`

## 第 4 步：启动 Worker

```bash
bun run worker:start
```

Worker 在后台运行，负责：
- 会话管理
- 观察处理
- AI 摘要生成
- 上下文文件更新

## 第 5 步：重启 Cursor 并验证

1. **重启 Cursor IDE** 以加载新 Hook

2. **检查安装状态**：
   ```bash
   bun run cursor:status
   ```

3. **验证 Worker 正在运行**：
   ```bash
   curl http://127.0.0.1:37777/api/readiness
   ```
   应返回：`{"status":"ready"}`

4. **打开 Web 查看器**：http://localhost:37777

✨ **30 天免费试用 — 记忆不占用套餐额度。让你的套餐用量提升最多 100%** → https://cmem.ai/pro?from=docs

## 工作原理

1. **每次提示前**：Hook 初始化会话并确保 Worker 正在运行
2. **代理工作期间**：捕获 MCP 工具、shell 命令和文件编辑
3. **代理停止时**：生成摘要并更新上下文文件
4. **下次会话**：通过 `.cursor/rules/` 自动注入最新上下文

## 故障排查

### "No provider configured" 错误

确认设置文件存在且凭据有效：
```bash
cat ~/.claude-mem/settings.json
```

### Worker 无法启动

查看日志：
```bash
tail -f ~/.claude-mem/logs/worker-$(date +%Y-%m-%d).log
```

### Hook 未执行

1. 检查 Cursor 设置 → Hooks 标签页是否有错误
2. 确认脚本可执行：
   ```bash
   chmod +x ~/.cursor/hooks/*.sh
   ```
3. 检查 Cursor 中的 Hooks 输出通道

### 速率限制（Gemini 未绑定计费）

如果触发 Gemini 的无计费速率限制：
- 等到第二天
- 启用计费以获得更高限额
- 切换到 OpenRouter

## 后续步骤

- 阅读 [README.md](README.md) 了解详细的 Hook 文档
- 查看 [CONTEXT-INJECTION.md](CONTEXT-INJECTION.md) 了解上下文行为细节
- 访问 https://docs.claude-mem.ai 获取完整文档

## 快速参考

| 命令 | 用途 |
|---------|---------|
| `bun run cursor:install -- user` | 为所有项目安装 Hook（推荐） |
| `bun run cursor:install` | 仅为当前项目安装 Hook |
| `bun run cursor:status` | 检查安装状态 |
| `bun run worker:start` | 启动后台 Worker |
| `bun run worker:stop` | 停止后台 Worker |
| `bun run worker:restart` | 重启 Worker |

---

## Windows 安装

Windows 用户通过 PowerShell 脚本获得完整支持。安装程序会自动检测 Windows 并安装相应脚本。

### 启用脚本执行（如需要）

PowerShell 可能要求你启用脚本执行：

```powershell
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser
```

### Windows 分步指南

```powershell
# Clone and build
git clone https://github.com/thedotmack/claude-mem.git
cd claude-mem
bun install
bun run build

# Configure provider (Gemini example)
$settingsDir = "$env:USERPROFILE\.claude-mem"
New-Item -ItemType Directory -Force -Path $settingsDir

@"
{
  "CLAUDE_MEM_PROVIDER": "gemini",
  "CLAUDE_MEM_GEMINI_API_KEY": "YOUR_GEMINI_API_KEY"
}
"@ | Out-File -FilePath "$settingsDir\settings.json" -Encoding UTF8

# Interactive setup (recommended - walks you through everything)
bun run cursor:setup

# Or manual installation
bun run cursor:install
bun run worker:start
```

### Windows 上安装的内容

安装程序会将以下 PowerShell 脚本复制到 `.cursor\hooks\`：

| 脚本 | 用途 |
|--------|---------|
| `common.ps1` | 共享工具函数 |
| `session-init.ps1` | 在提示时初始化会话 |
| `context-inject.ps1` | 注入记忆上下文 |
| `save-observation.ps1` | 捕获 MCP/shell 使用 |
| `save-file-edit.ps1` | 捕获文件编辑 |
| `session-summary.ps1` | 在停止时生成摘要 |

`hooks.json` 文件配置为以 `-ExecutionPolicy Bypass` 调用 PowerShell，确保脚本无需额外配置即可运行。

### Windows 故障排查

**"Execution of scripts is disabled on this system"**

以管理员身份运行：
```powershell
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope LocalMachine
```

**PowerShell 脚本未运行**

确认 hooks.json 包含 PowerShell 调用：
```powershell
Get-Content .cursor\hooks.json
```

应显示类似命令：
```
powershell.exe -ExecutionPolicy Bypass -File "./.cursor/hooks/session-init.ps1"
```

**Worker 无响应**

检查端口 37777 是否被占用：
```powershell
Get-NetTCPConnection -LocalPort 37777
```

**杀毒软件拦截脚本**

某些杀毒软件可能拦截 PowerShell 脚本。如有需要，为 `.cursor\hooks\` 目录添加例外。