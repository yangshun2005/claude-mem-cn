# Claude-Mem ↔ Cursor 集成架构

## 概述

本集成将 claude-mem 的持久化记忆系统接入 Cursor 的 Hook 系统，实现：
- 自动捕获 Agent 操作（MCP 工具、Shell 命令、文件编辑）
- 从历史会话中检索上下文
- 生成会话摘要供后续参考

## 架构

```
┌─────────────┐
│   Cursor    │
│   Agent     │
└──────┬──────┘
       │
       │ 事件（MCP、Shell、文件编辑、提示词）
       │
       ▼
┌─────────────────────────────────────┐
│      Cursor Hook 系统                │
│  ┌────────────────────────────────┐ │
│  │ beforeSubmitPrompt             │ │
│  │ afterMCPExecution              │ │
│  │ afterShellExecution            │ │
│  │ afterFileEdit                  │ │
│  │ stop                           │ │
│  └────────────────────────────────┘ │
└──────┬──────────────────────────────┘
       │
       │ HTTP 请求
       │
       ▼
┌─────────────────────────────────────┐
│   Hook 脚本（Bash）                  │
│  ┌────────────────────────────────┐ │
│  │ session-init.sh               │ │
│  │ context-inject.sh             │ │
│  │ save-observation.sh          │ │
│  │ save-file-edit.sh             │ │
│  │ session-summary.sh            │ │
│  └────────────────────────────────┘ │
└──────┬──────────────────────────────┘
       │
       │ HTTP API 调用
       │
       ▼
┌─────────────────────────────────────┐
│   Claude-Mem Worker 服务            │
│   （端口 37777）                     │
│  ┌────────────────────────────────┐ │
│  │ /api/sessions/init            │ │
│  │ /api/sessions/observations    │ │
│  │ /api/sessions/summarize       │ │
│  │ /api/context/inject          │ │
│  └────────────────────────────────┘ │
└──────┬──────────────────────────────┘
       │
       │ 数据库操作
       │
       ▼
┌─────────────────────────────────────┐
│   SQLite 数据库                      │
│   + Chroma 向量数据库                │
└─────────────────────────────────────┘
```

## 事件流

### 1. 提示词提交流程

```
用户提交提示词
    ↓
beforeSubmitPrompt Hook 触发
    ↓
session-init.sh
    ├─ 提取 conversation_id、项目名
    ├─ POST /api/sessions/init
    └─ 在 claude-mem 中初始化会话
    ↓
context-inject.sh
    ├─ GET /api/context/inject?project=...
    └─ 获取相关上下文（供后续使用）
    ↓
提示词继续传递给 Agent
```

### 2. 工具执行流程

```
Agent 执行 MCP 工具或 Shell 命令
    ↓
afterMCPExecution / afterShellExecution Hook 触发
    ↓
save-observation.sh
    ├─ 提取 tool_name、tool_input、tool_response
    ├─ 映射为 claude-mem 观察格式
    ├─ POST /api/sessions/observations
    └─ 将观察存入数据库
```

### 3. 文件编辑流程

```
Agent 编辑文件
    ↓
afterFileEdit Hook 触发
    ↓
save-file-edit.sh
    ├─ 提取 file_path、edits
    ├─ 创建 "write_file" 观察
    ├─ POST /api/sessions/observations
    └─ 存储文件编辑观察
```

### 4. 会话结束流程

```
Agent 循环结束
    ↓
stop Hook 触发
    ↓
session-summary.sh
    ├─ POST /api/sessions/summarize
    └─ 生成会话摘要供后续检索
```

## 数据映射

### 会话 ID 映射

| Cursor 字段 | Claude-Mem 字段 | 说明 |
|-------------|------------------|-------|
| `conversation_id` | `contentSessionId` | 跨轮次稳定，用作主会话标识符 |
| `generation_id` | （回退） | 当 conversation_id 不可用时使用 |

### 工具映射

| Cursor 事件 | Claude-Mem 工具名 | 输入格式 |
|-------------|---------------------|--------------|
| `afterMCPExecution` | 事件中的 `tool_name` | `tool_input` 作为 JSON |
| `afterShellExecution` | `"Bash"` | `{command: "..."}` |
| `afterFileEdit` | `"write_file"` | `{file_path: "...", edits: [...]}` |

### 项目映射

| 来源 | 目标 | 说明 |
|--------|--------|-------|
| `workspace_roots[0]` | 项目名 | 工作区根目录的基名 |

## 使用的 API 端点

### 会话管理
- `POST /api/sessions/init` - 初始化新会话
- `POST /api/sessions/summarize` - 生成会话摘要

### 观察存储
- `POST /api/sessions/observations` - 存储工具使用观察

### 上下文检索
- `GET /api/context/inject?project=...` - 获取用于注入的相关上下文

### 健康检查
- `GET /api/readiness` - 检查 Worker 是否就绪

## 配置

### Worker 设置
位于 `~/.claude-mem/settings.json`：
- `CLAUDE_MEM_WORKER_PORT`（默认：37777）
- `CLAUDE_MEM_WORKER_HOST`（默认：127.0.0.1）

### Hook 设置
位于 `hooks.json`：
- Hook 事件名
- 脚本路径（相对或绝对）

## 错误处理

### Worker 不可用
- Hook 轮询 `/api/readiness`，重试 30 次（6 秒）
- Worker 不可用时，Hook 优雅失败（退出码 0）
- 观察采用即发即忘模式（忽略 curl 错误）

### 数据缺失
- `conversation_id` 为空 → 使用 `generation_id`
- `workspace_root` 为空 → 使用 `pwd`
- 工具数据缺失 → 跳过观察

### 网络错误
- 所有 HTTP 请求使用 `curl -s`（静默）
- 错误重定向到 `/dev/null`
- Hook 始终退出码 0，避免阻塞 Cursor

## 限制

1. **上下文注入**：Cursor 的 `beforeSubmitPrompt` 不支持修改提示词。上下文必须通过以下方式获取：
   - MCP 工具（claude-mem 提供搜索工具）
   - 从 Web 查看器手动检索
   - 未来：Agent SDK 集成

2. **对话记录访问**：Cursor Hook 不提供对话记录路径，摘要质量不如 Claude Code 集成。

3. **会话模型**：使用 `conversation_id`，可能与 Claude Code 的会话模型不完全一致。

4. **Tab Hook**：目前仅支持 Agent Hook。Tab（内联补全）Hook 可单独添加。

## 未来增强

- [ ] 通过 MCP 工具增强上下文注入
- [ ] 支持 `beforeTabFileRead` 和 `afterTabFileEdit` Hook
- [ ] 改进错误报告和日志
- [ ] 集成 Cursor 的 Agent SDK
- [ ] 支持阻塞/审批工作流
- [ ] 通过 Agent 消息实时注入上下文

## 测试

### 手动测试

1. **测试会话初始化**：
   ```bash
   echo '{"conversation_id":"test-123","workspace_roots":["/tmp/test"],"prompt":"test"}' | \
     ~/.cursor/hooks/session-init.sh
   ```

2. **测试观察捕获**：
   ```bash
   echo '{"conversation_id":"test-123","hook_event_name":"afterMCPExecution","tool_name":"test","tool_input":{},"result_json":{}}' | \
     ~/.cursor/hooks/save-observation.sh
   ```

3. **测试上下文检索**：
   ```bash
   curl "http://127.0.0.1:37777/api/context/inject?project=test"
   ```

### 集成测试

1. 在 Cursor 中启用 Hook
2. 提交提示词
3. 执行一些工具
4. 检查 Web 查看器：`http://localhost:37777`
5. 验证观察出现在数据库中

## 故障排查

详细排查步骤见 [README.md](README.md#troubleshooting)。