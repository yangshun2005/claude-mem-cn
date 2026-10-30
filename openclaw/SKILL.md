# Claude-Mem OpenClaw 插件 — 安装指南

本指南介绍如何在 OpenClaw 网关上安装 claude-mem 插件。完成后，你的智能体将通过系统提示词上下文注入获得跨会话的持久记忆，还可选择将实时观察流推送到消息频道。

## 快速安装（推荐）

运行以下一行命令自动完成全部安装：

```bash
curl -fsSL https://install.cmem.ai/openclaw.sh | bash
```

安装程序以交互方式处理依赖检查（Bun、uv）、插件安装、记忆槽配置、AI 提供商设置、worker 启动以及可选的观察流配置。

### 带选项安装

预选 AI 提供商和 API 密钥以跳过交互提示：

```bash
curl -fsSL https://install.cmem.ai/openclaw.sh | bash -s -- --provider=gemini --api-key=YOUR_KEY
```

完全无人值守安装（默认使用 Claude Max Plan，跳过观察流）：

```bash
curl -fsSL https://install.cmem.ai/openclaw.sh | bash -s -- --non-interactive
```

升级现有安装（保留设置，更新插件）：

```bash
curl -fsSL https://install.cmem.ai/openclaw.sh | bash -s -- --upgrade
```

安装完成后，跳到[第 4 步：重启网关并验证](#step-4-restart-the-gateway-and-verify)确认一切正常。

---

## 手动安装

以下步骤适用于不想使用自动安装程序、或需要排查单个步骤的情况。

### 第 1 步：克隆 Claude-Mem 仓库

首先将 claude-mem 仓库克隆到 OpenClaw 网关可访问的位置。这样你就能获得 worker 服务源码和插件代码。

```bash
cd /opt  # 或你希望存放的任何位置
git clone https://github.com/thedotmack/claude-mem.git
cd claude-mem
npm install
npm run build
```

worker 服务需要安装 **bun**。如果没有：

```bash
curl -fsSL https://bun.sh/install | bash
```

### 第 2 步：启动 Worker

claude-mem worker 是运行在 37777 端口的 HTTP 服务。它存储观察记录、生成摘要并提供上下文时间线。插件通过 HTTP 与之通信——worker 在哪里运行不重要，只要能在 localhost:37777 访问即可。

#### 检查是否已在运行

如果本机也运行着安装了 claude-mem 的 Claude Code，worker 可能已在运行：

```bash
curl http://localhost:37777/api/health
```

**返回 `{"status":"ok"}`？** worker 已在运行。跳到第 3 步。

**连接被拒绝或无响应？** worker 未运行。继续往下看。

#### 如果 Claude Code 已安装 claude-mem

如果 claude-mem 作为 Claude Code 插件安装（位于 `~/.claude/plugins/marketplaces/thedotmack/`），从该安装启动 worker：

```bash
cd ~/.claude/plugins/marketplaces/thedotmack
npm run worker:restart
```

验证：
```bash
curl http://localhost:37777/api/health
```

**返回 `{"status":"ok"}`？** 已就绪。跳到第 3 步。

**仍不工作？** 查看 `npm run worker:status` 获取错误详情，或确认 bun 已安装并在 PATH 中。

#### 如果没有 Claude Code 安装

从克隆的仓库运行 worker：

```bash
cd /opt/claude-mem  # 你克隆的位置
npm run worker:start
```

验证：
```bash
curl http://localhost:37777/api/health
```

**返回 `{"status":"ok"}`？** 已就绪。进入第 3 步。

**仍不工作？** 调试步骤：
- 确认 bun 已安装：`bun --version`
- 查看 worker 状态：`npm run worker:status`
- 检查 37777 端口是否被占用：`lsof -i :37777`
- 查看日志：`npm run worker:logs`（如可用）
- 直接运行以查看错误：`bun plugin/scripts/worker-service.cjs start`

### 第 3 步：将插件添加到网关

将 `claude-mem` 插件添加到 OpenClaw 网关配置：

```json
{
  "plugins": {
    "claude-mem": {
      "enabled": true,
      "config": {
        "project": "my-project",
        "syncMemoryFile": true,
        "workerPort": 37777
      }
    }
  }
}
```

#### 配置字段说明

- **`project`**（string，默认：`"openclaw"`）— 项目名称，用于在记忆数据库中限定所有观察记录的范围。每个网关/用例使用唯一名称，避免观察记录混淆。例如，如果此网关运行编码机器人，使用 `"coding-bot"`。

- **`syncMemoryFile`**（boolean，默认：`true`）— 启用后，插件通过 `before_prompt_build` Hook 将观察时间线注入每个智能体的系统提示词。智能体无需写入 MEMORY.md 即可获得跨会话上下文。设为 `false` 可完全禁用上下文注入（观察记录仍会记录）。

- **`syncMemoryFileExclude`**（string[]，默认：`[]`）— 排除自动上下文注入的智能体 ID。适用于自行管理记忆的智能体。被排除的智能体仍会记录观察。

- **`workerPort`**（number，默认：`37777`）— claude-mem worker 服务监听的端口。仅在你配置了 worker 使用其他端口时才需修改。

---

## 第 4 步：重启网关并验证

重启 OpenClaw 网关以加载新插件配置。重启后，检查网关日志中是否出现：

```
[claude-mem] OpenClaw plugin loaded — v1.0.0 (worker: 127.0.0.1:37777)
```

看到这行说明插件已加载。也可在任意 OpenClaw 聊天中运行 `/claude_mem_status` 验证：

```
Claude-Mem Worker Status
Status: ok
Port: 37777
Active sessions: 0
Observation feed: disconnected
```

观察流显示 `disconnected` 是因为尚未配置。接下来处理。

## 第 5 步：验证观察记录正在写入

让智能体执行一些工作。插件通过以下 OpenClaw 事件自动记录观察：

1. **`before_agent_start`** — 智能体启动时初始化 claude-mem 会话
2. **`before_prompt_build`** — 将观察时间线注入智能体的系统提示词（缓存 60 秒）
3. **`after_tool_call`** — 将每次工具调用（Read、Write、Bash 等）记录为观察
4. **`agent_end`** — 总结会话并标记完成

全部自动完成，无需额外配置。

要验证是否生效，访问 worker 的查看器 UI http://localhost:37777，查看智能体运行后出现的观察记录。

也可访问 worker 的查看器 UI http://localhost:37777 实时查看观察记录。

## 第 6 步：设置观察流（推送到频道）

观察流连接 claude-mem worker 的 SSE（Server-Sent Events）流，将每条新观察实时转发到消息频道。智能体学到东西，你就能在 Telegram/Discord/Slack 等中看到。

### 你会看到什么

每当 claude-mem 从智能体的工具使用中创建新观察，频道中会出现类似消息：

```
🧠 Claude-Mem Observation
**Implemented retry logic for API client**
Added exponential backoff with configurable max retries to handle transient failures
```

### 选择频道

需要两项信息：
- **频道类型** — 必须匹配 OpenClaw 网关上已运行的频道插件
- **目标 ID** — 消息发送到的聊天/频道/用户 ID

#### Telegram

频道类型：`telegram`

查找聊天 ID：
1. 在 Telegram 上给 @userinfobot 发消息 — https://t.me/userinfobot
2. 它会回复你的数字聊天 ID（如 `123456789`）
3. 群聊 ID 为负数（如 `-1001234567890`）

```json
"observationFeed": {
  "enabled": true,
  "channel": "telegram",
  "to": "123456789"
}
```

#### Discord

频道类型：`discord`

查找频道 ID：
1. 在 Discord 中启用开发者模式：设置 → 高级 → 开发者模式
2. 右键目标频道 → 复制频道 ID

```json
"observationFeed": {
  "enabled": true,
  "channel": "discord",
  "to": "1234567890123456789"
}
```

#### Slack

频道类型：`slack`

查找频道 ID（不是频道名称）：
1. 在 Slack 中打开频道
2. 点击顶部频道名称
3. 滚动到频道详情底部 — ID 形如 `C01ABC2DEFG`

```json
"observationFeed": {
  "enabled": true,
  "channel": "slack",
  "to": "C01ABC2DEFG"
}
```

#### Signal

频道类型：`signal`

使用 OpenClaw 网关 Signal 插件中配置的电话号码或群组 ID。

```json
"observationFeed": {
  "enabled": true,
  "channel": "signal",
  "to": "+1234567890"
}
```

#### WhatsApp

频道类型：`whatsapp`

使用 OpenClaw 网关 WhatsApp 插件中配置的电话号码或群组 JID。

```json
"observationFeed": {
  "enabled": true,
  "channel": "whatsapp",
  "to": "+1234567890"
}
```

#### LINE

频道类型：`line`

使用 LINE 开发者控制台中的用户 ID 或群组 ID。

```json
"observationFeed": {
  "enabled": true,
  "channel": "line",
  "to": "U1234567890abcdef"
}
```

### 添加到配置

完整插件配置现在应如下所示（以 Telegram 为例）：

```json
{
  "plugins": {
    "claude-mem": {
      "enabled": true,
      "config": {
        "project": "my-project",
        "syncMemoryFile": true,
        "workerPort": 37777,
        "observationFeed": {
          "enabled": true,
          "channel": "telegram",
          "to": "123456789"
        }
      }
    }
  }
}
```

### 重启并验证

重启网关。按顺序检查日志中是否出现以下三行：

```
[claude-mem] Observation feed starting — channel: telegram, target: 123456789
[claude-mem] Connecting to SSE stream at http://localhost:37777/stream
[claude-mem] Connected to SSE stream
```

然后在任意 OpenClaw 聊天中运行 `/claude_mem_feed`：

```
Claude-Mem Observation Feed
Enabled: yes
Channel: telegram
Target: 123456789
Connection: connected
```

如果 `Connection` 显示 `connected`，就完成了。让智能体执行一些工作，观察记录会流式发送到你的频道。

## 命令参考

插件注册两个命令：

### /claude_mem_status

报告 worker 健康状况和当前会话状态。

```
/claude_mem_status
```

输出：
```
Claude-Mem Worker Status
Status: ok
Port: 37777
Active sessions: 2
Observation feed: connected
```

### /claude_mem_feed

显示观察流状态。接受可选的 `on`/`off` 参数。

```
/claude_mem_feed          — 显示状态
/claude_mem_feed on       — 请求启用（需更新配置以持久化）
/claude_mem_feed off      — 请求禁用（需更新配置以持久化）
```

## 工作原理

```
OpenClaw Gateway
  │
  ├── before_agent_start ───→ 初始化会话
  ├── before_prompt_build ──→ 注入上下文到系统提示词
  ├── after_tool_call ──────→ 记录观察
  ├── agent_end ────────────→ 总结 + 完成会话
  └── gateway_start ────────→ 重置会话跟踪 + 上下文缓存
                    │
                    ▼
         Claude-Mem Worker (localhost:37777)
           ├── POST /api/sessions/init
           ├── POST /api/sessions/observations
           ├── POST /api/sessions/summarize
           ├── POST /api/sessions/complete
           ├── GET  /api/context/inject ──→ 系统提示词上下文
           └── GET  /stream ─────────────→ SSE → 消息频道
```

### 系统提示词上下文注入

插件通过 `before_prompt_build` Hook 将观察时间线注入每个智能体的系统提示词。内容来自 worker 的 `GET /api/context/inject` 端点。上下文按项目缓存 60 秒，避免每次 LLM 轮次重新获取。网关重启时清除缓存。

这样 MEMORY.md 仍由智能体控制，用于精选的长期记忆，而观察时间线通过系统提示词传递。

### 观察记录

每次工具使用（Read、Write、Bash 等）都作为观察发送到 claude-mem worker。worker 的 AI 智能体将其处理为结构化观察，包含标题、副标题、事实、概念和叙述。以 `memory_` 为前缀的工具会被跳过，避免递归记录。

### 会话生命周期

- **`before_agent_start`** — 在 worker 中创建会话。
- **`before_prompt_build`** — 获取观察时间线并作为 `appendSystemContext` 返回。缓存 60 秒。
- **`after_tool_call`** — 从调用的参数和结果记录观察（即发即忘）；失败调用的响应以其错误文本开头。工具响应截断至 1000 字符。
- **`agent_end`** — 发送最后一条助手消息进行总结，然后完成会话。两者均为即发即忘。
- **`gateway_start`** — 清除所有会话跟踪（会话 ID、上下文缓存），让智能体重新开始。

### 观察流

后台服务连接 worker 的 SSE 流，将 `new_observation` 事件转发到配置的消息频道。连接以指数退避自动重连（1 秒 → 最长 30 秒）。

## 故障排查

| 问题 | 检查项 |
|---------|---------------|
| Worker 健康检查失败 | bun 是否安装？（`bun --version`）。37777 端口是否被占用？（`lsof -i :37777`）。尝试直接运行：`bun plugin/scripts/worker-service.cjs start` |
| Worker 从 Claude Code 安装启动但无响应 | 检查 `cd ~/.claude/plugins/marketplaces/thedotmack && npm run worker:status`。可能需要 `npm run worker:restart`。 |
| Worker 从克隆仓库启动但无响应 | 检查 `cd /path/to/claude-mem && npm run worker:status`。确认已先运行 `npm install && npm run build`。 |
| 智能体系统提示词中无上下文 | 检查 `syncMemoryFile` 未设为 `false`。检查智能体 ID 不在 `syncMemoryFileExclude` 中。确认 worker 正在运行且有观察记录。 |
| 观察记录未写入 | 检查网关日志中的 `[claude-mem]` 消息。worker 必须运行且可在 localhost:37777 访问。 |
| 观察流显示 `disconnected` | worker 的 `/stream` 端点不可达。检查 `workerPort` 是否匹配实际 worker 端口。 |
| 观察流显示 `reconnecting` | 连接断开。插件会自动重连——最多等待 30 秒。 |
| 日志中出现 `Unknown channel type` | 频道插件（如 telegram）未在网关上加载。确认频道已配置并运行。 |
| 日志中出现 `Observation feed disabled` | 在配置中将 `observationFeed.enabled` 设为 `true`。 |
| 日志中出现 `Observation feed misconfigured` | `observationFeed.channel` 和 `observationFeed.to` 均为必填。 |
| 显示 `connected` 但频道无消息 | 观察流只发送处理后的观察，不发送原始工具使用。有 1-2 秒延迟。确认 worker 确实在处理观察（检查 http://localhost:37777）。 |

## 完整配置参考

```json
{
  "plugins": {
    "claude-mem": {
      "enabled": true,
      "config": {
        "project": "openclaw",
        "syncMemoryFile": true,
        "workerPort": 37777,
        "observationFeed": {
          "enabled": false,
          "channel": "telegram",
          "to": "123456789"
        }
      }
    }
  }
}
```

| 字段 | 类型 | 默认值 | 描述 |
|-------|------|---------|-------------|
| `project` | string | `"openclaw"` | 在数据库中限定观察记录范围的项目名称 |
| `syncMemoryFile` | boolean | `true` | 将观察上下文注入智能体系统提示词 |
| `syncMemoryFileExclude` | string[] | `[]` | 排除上下文注入的智能体 ID |
| `workerPort` | number | `37777` | Claude-mem worker 服务端口 |
| `observationFeed.enabled` | boolean | `false` | 将观察流推送到消息频道 |
| `observationFeed.channel` | string | — | 频道类型：`telegram`、`discord`、`slack`、`signal`、`whatsapp`、`line` |
| `observationFeed.to` | string | — | 目标聊天/频道/用户 ID |