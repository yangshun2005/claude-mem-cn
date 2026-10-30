# 原生 ODS 集成

Claude-Mem 直接连接 **ODS 本身**：其已准入的对话请求、已验证的活动事件，以及最终验证的投递。它不需要 agent 插件，也不改动底层 agent 的配置。

ODS 扩展运行本地 Claude-Mem worker，将记忆存储在 `data/claude-mem`，并使用现有的 ODS 模型网关（`ods/current`）进行观察提取。SQLite 搜索已启用；此轻量部署中 Chroma 和遥测关闭。

## 从源码安装

这是源码集成，不是 ODS 目录中已发布的条目。补丁基于 `Osmantic/ODS@c99f52822f2c37813fd880004f964ca83dfc5a35` 的 ODS 3.0.0 验证。安装器在改动所选 ODS 树之前会检查每个补丁片段。

```bash
# 在 Claude-Mem 检出目录中执行；该目录可以是 ODS 仓库或其 ods/ 文件夹。
node ods/install.mjs --ods-dir /path/to/ODS --check
node ods/install.mjs --ods-dir /path/to/ODS
```

这会准备原生源码改动并安装已禁用的服务定义。现有应用数据和密钥会保留。重复执行命令是安全的；本地修改过的集成会被报告，而不是被替换。

在 ODS 安装的 `.env` 中设置以下值：

```dotenv
ODS_CLAUDE_MEM_ENABLED=true
ODS_CLAUDE_MEM_PORT=37778
ODS_CLAUDE_MEM_SCOPE=conversation
```

通过 ODS 现有的本地安装流程部署修改后的 ODS 树，使更新后的入口模块和环境变量到达宿主服务。源码补丁会将其加入 ODS 自身的部署步骤。仅复制扩展目录**不会**更新已在运行的入口进程。然后启用服务：

```bash
ods enable claude-mem
ods start claude-mem
```

worker 的查看器和 API 位于 `http://127.0.0.1:37778`；其健康端点为 `/api/health`。API 无身份验证，因此其暴露范围受网络位置限制：

- **宿主机：** 发布的端口仅绑定 `127.0.0.1`。ODS 宿主机上的任何本地进程或用户都可以通过它读取、写入和删除记忆，与桌面版 Claude-Mem worker 相同。
- **容器：** worker 不在共享的 `ods-network` 上。它位于私有 Compose 网络，该网络唯一的其他成员是 `claude-mem-gateway`。其他 ODS 服务和扩展无法访问 worker 的 API。
- **网关：** `claude-mem-gateway`（`gateway.mjs`）是 `ods-network` 上唯一的容器。它将 worker 的观察者调用转发到 LiteLLM，并从 worker 应答仪表板的 `GET /api/health` 探测。它不转发其他 worker 请求。

`LITELLM_KEY` 会传递给 ODS 网络内现有的本地模型网关。此集成不选择新模型，也不改变 ODS 的本地/云端路由策略。

## 记忆范围

`conversation` 为默认值。每个 ODS 对话拥有独立的持久记忆，即使底层会话已被替换或压缩。

对于个人单用户 ODS 设备，设置 `ODS_CLAUDE_MEM_SCOPE=installation` 可跨聊天回忆记忆。这会有意在该安装的所有对话间共享一个 `ods` 项目。它不是多用户服务的账户隔离边界。范围仅来自运维配置，绝不来自模型输出。

## 集成记录的内容

| ODS 事件 | Claude-Mem 操作 |
| --- | --- |
| 新准入请求 | 使用原生请求 ID 和 `platformSource: ods` 初始化会话 |
| 模型执行前 | 检索范围内最近五条观察，作为有界的仅模型参考 |
| 已验证的终端活动事件 | 存储 `ODSActivity` 及其来源和状态 |
| 已验证的最终投递 | 存储 `ODSConversation`，单独保留验证信息，并生成摘要 |
| 缓存响应重放 | 不进行新的捕获或模型执行 |
| 对话删除 | 围栏排队的写入，并请求删除确切的 Claude-Mem 会话 |

集成捕获的是 ODS 的**公开活动投影**，而非 agent 运行时内部隐藏的原始工具。活动记录标记为观察，而非独立的成功证明。最终消息使用 ODS 已验证的投递。流式和非流式请求使用相同的原生边界。两种模式都会在存储最终答案前收集最终活动快照。私有块在应用捕获大小限制前被移除；嵌套和未完成的私有块保持私有。完全私有的提示会跳过捕获。

记忆读写有短超时。记忆不可用时聊天仍可用，并发出经过清理的警告。写入失败会停止该轮的写入链；它们不会被报告为已保存，也不会被静默重放。此版本不提供离线发件箱。原生请求和工具使用 ID 支持 worker 现有的重复检测。

已发布的 worker 在观察者活动时拒绝删除会话。此时 ODS 报告 `memory-deletion-unconfirmed`，而不是声称所有副本已擦除。观察者空闲后重试删除。从未被捕获或已删除的会话视为不存在。

## 验证

```bash
node --test ods/test/native.test.mjs
ODS_TEST_REPO=/path/to/ODS node --test ods/test/install.test.mjs
ODS_TEST_ROOT=/path/to/ODS/ods node --test ods/test/ingress.test.mjs
# 最后一条命令要求该树中已安装原生集成。

# 运行实际捆绑的 worker、其 SQLite 存储和本地测试观察者。
# 先安装插件依赖，与普通 Claude-Mem 开发相同。
BUN_BIN=/path/to/bun node ods/test/worker-smoke.mjs
```

测试覆盖捕获/回忆、隐私跳过、有界响应、写入失败、对话隔离、删除、安装回滚、本地编辑保护，以及两种 ODS 响应模式。worker 冒烟测试使用确定性模型输出；它不衡量任何特定本地模型的质量。

## 移除

```bash
ods disable claude-mem
node ods/install.mjs --ods-dir /path/to/ODS --uninstall
```

重新部署 ODS 恢复后的入口源码，并从 `.env` 中移除三个可选设置。安装器仅恢复其拥有的源文件，并验证它们未被更改。`data/claude-mem` 保留以供备份或日后重装。仅当你打算擦除时才单独删除该目录。如果移除因文件系统错误中断，修复错误后重复 `--uninstall`。其回执跟踪移除意图，并仍保护本地编辑。