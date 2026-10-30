# OpenClaw Claude-Mem 插件 — 测试指南

## 快速开始（Docker）

测试插件最快的方式是使用预构建的 Docker E2E 环境：

```bash
cd openclaw

# 自动化测试（构建、在真实 OpenClaw 上安装插件、验证一切）
./test-e2e.sh

# 交互式 shell（用于手动探索）
./test-e2e.sh --interactive

# 仅构建镜像
./test-e2e.sh --build-only
```

---

## 测试层级

### 1. 单元测试（最快）

```bash
cd openclaw
npm test    # 编译 TypeScript，运行 17 个测试
```

测试插件注册、服务生命周期、命令处理、SSE 集成以及全部 6 种通道类型。

### 2. 冒烟测试

```bash
node test-sse-consumer.js
```

快速检查插件能否正确加载并注册其服务和命令。

### 3. 容器单元测试（全新安装）

```bash
./test-container.sh          # 在干净 Docker 中运行单元测试
./test-container.sh --full   # 使用 mock worker 运行集成测试
```

### 4. 真实 OpenClaw 上的 E2E（Docker）

```bash
./test-e2e.sh
```

这是最全面的测试。它会：
1. 使用官方 `ghcr.io/openclaw/openclaw:main` Docker 镜像
2. 通过 `openclaw plugins install` 安装插件（与真实用户操作一致）
3. 通过 `openclaw plugins enable` 启用插件
4. 在端口 37777 上启动 mock claude-mem worker
5. 使用插件配置启动 OpenClaw 网关
6. 验证插件加载、连接 SSE 并处理事件

**全部 16 项检查必须通过。**

---

## 人工 E2E 测试（交互式 Docker）

如需手动逐步测试，使用交互式 Docker 模式：

```bash
./test-e2e.sh --interactive
```

这会让你进入一个已完整配置、预装插件的 OpenClaw 容器。

### 容器内分步操作

#### 1. 验证插件已安装

```bash
node openclaw.mjs plugins list
node openclaw.mjs plugins info claude-mem
node openclaw.mjs plugins doctor
```

**预期：**
- `claude-mem` 在插件列表中显示为 "enabled" 或 "loaded"
- Info 显示版本 1.0.0，源位于 `/home/node/.openclaw/extensions/claude-mem/`
- Doctor 报告无问题

#### 2. 检查插件文件

```bash
ls -la /home/node/.openclaw/extensions/claude-mem/
cat /home/node/.openclaw/extensions/claude-mem/openclaw.plugin.json
cat /home/node/.openclaw/extensions/claude-mem/package.json
```

**预期：**
- `dist/index.js` 存在（已编译插件）
- `openclaw.plugin.json` 包含 `"id": "claude-mem"` 和 `"kind": "memory"`
- `package.json` 包含指向 `./dist/index.js` 的 `openclaw.extensions` 字段

#### 3. 启动 mock worker

```bash
node /app/mock-worker.js &
```

验证其正在运行：

```bash
curl -s http://localhost:37777/health
# → {"status":"ok"}

curl -s --max-time 3 http://localhost:37777/stream
# → data: {"type":"connected","message":"Mock worker SSE stream"}
# → data: {"type":"new_observation","observation":{...}}
```

#### 4. 配置并启动网关

```bash
cat > /home/node/.openclaw/openclaw.json << 'EOF'
{
  "gateway": {
    "mode": "local",
    "auth": {
      "mode": "token",
      "token": "e2e-test-token"
    }
  },
  "plugins": {
    "slots": {
      "memory": "claude-mem"
    },
    "entries": {
      "claude-mem": {
        "enabled": true,
        "config": {
          "workerPort": 37777,
          "observationFeed": {
            "enabled": true,
            "channel": "telegram",
            "to": "test-chat-id-12345"
          }
        }
      }
    }
  }
}
EOF

node openclaw.mjs gateway --allow-unconfigured --verbose --token e2e-test-token
```

**网关日志中预期出现：**
- `[claude-mem] OpenClaw plugin loaded — v1.0.0`
- `[claude-mem] Observation feed starting — channel: telegram, target: test-chat-id-12345`
- `[claude-mem] Connecting to SSE stream at http://localhost:37777/stream`
- `[claude-mem] Connected to SSE stream`

#### 5. 运行自动化验证（可选）

在容器内另开一个 shell（或停止网关后）：

```bash
/bin/bash /app/e2e-verify.sh
```

---

## 手动 E2E（真实 OpenClaw + 真实 Worker）

使用真实 claude-mem worker 和真实消息通道进行测试：

### 前置条件

- 已安装并配置 OpenClaw 网关
- Claude-Mem worker 运行在端口 37777
- 插件已构建：`cd openclaw && npm run build`

### 1. 安装插件

```bash
# 构建插件
cd openclaw && npm run build

# 在 OpenClaw 上安装（从 openclaw/ 目录执行）
openclaw plugins install .

# 启用插件
openclaw plugins enable claude-mem
```

### 2. 配置

编辑 `~/.openclaw/openclaw.json` 添加插件配置：

```json
{
  "plugins": {
    "entries": {
      "claude-mem": {
        "enabled": true,
        "config": {
          "workerPort": 37777,
          "observationFeed": {
            "enabled": true,
            "channel": "telegram",
            "to": "YOUR_CHAT_ID"
          }
        }
      }
    }
  }
}
```

**支持的通道：** `telegram`、`discord`、`signal`、`slack`、`whatsapp`、`line`

### 3. 重启网关

```bash
openclaw restart
```

**在日志中查找：**
- `[claude-mem] OpenClaw plugin loaded — v1.0.0`
- `[claude-mem] Connected to SSE stream`

### 4. 触发观察事件

启动启用 claude-mem 的 Claude Code 会话并执行任意操作。worker 将发出 `new_observation` SSE 事件。

### 5. 验证投递

检查目标消息通道是否收到：

```
🧠 Claude-Mem Observation
**Observation Title**
Optional subtitle
```

---

## 故障排查

### `api.log is not a function`
插件针对错误的 API 构建。确保 `src/index.ts` 使用 `api.logger.info()` 而非 `api.log()`。用 `npm run build` 重新构建。

### Worker 未运行
- **症状：** `SSE stream error: fetch failed. Reconnecting in 1s`
- **修复：** 启动 worker：`cd /path/to/claude-mem && npm run build-and-sync`

### 端口不匹配
- **修复：** 确保配置中的 `workerPort` 与 worker 实际端口一致（默认：37777）

### 通道未配置
- **症状：** `Observation feed misconfigured — channel or target missing`
- **修复：** 在配置的 `observationFeed` 中同时添加 `channel` 和 `to`

### 未知通道类型
- **修复：** 使用：`telegram`、`discord`、`signal`、`slack`、`whatsapp` 或 `line`

### Feed 已禁用
- **症状：** `Observation feed disabled`
- **修复：** 设置 `observationFeed.enabled: true`

### 消息未送达
1. 验证 bot/集成已在目标通道中配置
2. 检查目标 ID（`to`）是否正确
3. 在日志中查找 `Failed to send to <channel>`
4. 通过 OpenClaw 内置工具测试该通道

### 内存槽冲突
- **症状：** `plugin disabled (memory slot set to "memory-core")`
- **修复：** 在插件配置中添加 `"slots": { "memory": "claude-mem" }`