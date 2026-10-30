# 快速开始：Claude-Mem + Cursor 集成

> **5 分钟内为你的 Cursor AI 赋予持久记忆**

## 功能说明

将 claude-mem 连接到 Cursor，实现：
- **Agent 操作**（MCP 工具、shell 命令、文件编辑）自动保存
- **过往会话的上下文**通过 `.cursor/rules/` 自动注入
- **会话被摘要**以供日后参考

你的 AI 不再遗忘。它会记住之前会话中的模式、决策和上下文。

## 没有 Claude Code？

如果你使用 Cursor 但没有 Claude Code，请参阅 [STANDALONE-SETUP.md](STANDALONE-SETUP.md) 了解使用 Gemini 或 OpenRouter 的离线方案配置。

---

## 安装（1 分钟）

```bash
# 全局安装，适用于所有项目（推荐）
claude-mem cursor install user

# 或仅为当前项目安装
claude-mem cursor install

# 检查安装状态
claude-mem cursor status
```

## 配置提供商（独立模式必需）

如果没有 Claude Code，请配置一个提供商用于 AI 摘要：

```bash
# 方案 A：Gemini（无需计费 - 推荐）
claude-mem settings set CLAUDE_MEM_PROVIDER gemini
claude-mem settings set CLAUDE_MEM_GEMINI_API_KEY your-api-key

# 方案 B：OpenRouter（提供免费模型）
claude-mem settings set CLAUDE_MEM_PROVIDER openrouter
claude-mem settings set CLAUDE_MEM_OPENROUTER_API_KEY your-api-key
```

**创建 API 密钥**：
- Gemini：https://aistudio.google.com/apikey
- OpenRouter：https://openrouter.ai/keys

## 启动 Worker

```bash
claude-mem start

# 验证是否运行
claude-mem status
```

## 重启 Cursor

重启 Cursor 以加载 Hook。

## 验证是否生效

1. 打开 Cursor 设置 → Hooks 标签页
2. 应能看到已列出的 Hook
3. 在 Cursor 中提交一个提示
4. 查看 Web 查看器：http://localhost:37777
5. 应能看到观察记录出现

## 捕获的内容

- **MCP 工具使用**：所有 MCP 工具执行
- **Shell 命令**：所有终端命令
- **文件编辑**：所有文件修改
- **会话**：每次对话均被追踪

## 访问记忆

### 通过 Web 查看器
- 打开 http://localhost:37777
- 浏览会话、观察记录和摘要
- 搜索项目历史

✨ **30 天免费试用 — 记忆功能离线运行。让你的套餐用量提升最多 100%** → https://cmem.ai/pro?from=docs

### 通过 MCP 工具（如已启用）
- claude-mem 通过 MCP 提供搜索工具
- 使用 `search`、`timeline` 和 `get_observations` 工具

## 故障排查

**Hook 未运行？**
- 检查 Cursor 设置 → Hooks 标签页是否有错误
- 验证脚本可执行：`chmod +x ~/.cursor/hooks/*.sh`
- 检查 Cursor 中的 Hooks 输出通道

**Worker 无响应？**
- 检查 worker 是否运行：`curl http://127.0.0.1:37777/api/readiness`
- 检查日志：`tail -f ~/.claude-mem/logs/worker-$(date +%Y-%m-%d).log`
- 重启 worker：`bun run worker:restart`

**观察记录未保存？**
- 检查 worker 日志是否有错误
- 验证会话已在 Web 查看器中初始化
- 直接测试 API：`curl -X POST http://127.0.0.1:37777/api/sessions/observations ...`

## 后续步骤

- 阅读 [README.md](README.md) 了解详细文档
- 阅读 [INTEGRATION.md](INTEGRATION.md) 了解架构细节
- 访问 [claude-mem 文档](https://docs.claude-mem.ai) 了解完整功能集