# [plan-08] OpenCode 集成事件契约正确性 — 让 OpenCode 插件真正捕获

## 缺陷

claude-mem 的 OpenCode 插件是按 OpenCode Hook API 中**不存在**的事件名编写的（如 `session.created`、`message.updated`、`session.compacted`、`file.edited`、`session.deleted`）。由于真实事件从不触发，会话从未初始化，也没有任何观察被捕获——而 `claude-mem install` 仍报告成功。结果是静默死循环：OpenCode 用户以为记忆在记录，实际什么都没记录。第二个缺陷加剧了问题：OpenCode 搜索客户端解析 `data.items`，而 worker 返回的是 Claude 风格的 `data.content` 块，因此即使手动搜索也返回“No results”，且它从错误的来源解析 worker 端口。

架构上的修复是将插件绑定到 OpenCode 的**真实**事件契约和 worker 的**实际**响应结构，然后添加契约测试，使未来 OpenCode API 变更时 CI 失败，而不是静默禁用捕获。

## 子项

- #2435 — 插件订阅了不存在的 OpenCode 事件名 → 零会话/零观察被记录
- #2406 — `claude_mem_search` 始终返回“No results”（`data.items` 与 `data.content`）；worker 端口解析使用了错误的来源
- #2419 — 同一缺口的特性描述：OpenCode 插件缺少 `tool.execute.after` 观察捕获
- #2462 — 重复的用户报告：OpenCode 安装报告成功但不捕获任何记忆

## 修复顺序

1. **重新绑定到真实事件：** 按 OpenCode 的实际 Hook（`tool.execute.after`、`chat.message`、`experimental.session.compacting` 等）重写插件；在正确的事件上添加会话初始化 + 观察 POST（#2435、#2419）。
2. **修复搜索客户端：** 解析 worker 的 `data.content` 块结构；从权威设置来源解析 worker 端口，并遵循环境变量覆盖（#2406）。
3. **契约测试：** 一个测试，断言插件只订阅 OpenCode 实际发出的事件名，且搜索客户端解析 worker 的真实响应结构。这是回归防护。
4. **安装诚实性：** OpenCode 安装必须在报告成功前验证捕获已生效（一次往返），使未来的契约断裂在安装时暴露（与 plan-04 关联）。

## 测试矩阵

| 层面 | 输入 | 必需行为 |
|---|---|---|
| OpenCode 中的工具执行 | 一次工具调用 | `tool.execute.after` 触发 → 观察被 POST → 行出现 |
| 会话生命周期 | 打开/压缩/关闭 | 会话初始化 + 压缩通过真实事件处理 |
| `claude_mem_search` | 一个有已知结果的查询 | 解析 `data.content`；返回行（而非“No results”） |
| Worker 端口解析 | 通过环境变量/设置指定非默认端口 | 客户端指向正确端口 |

该矩阵在 CI 中运行。OpenCode 捕获回归必须在用户提交 Issue 之前使 CI 失败。

## 范围之外

- Claude Code Hook IO 规范 → plan-01。
- Worker 写入路径 / 持久化正确性 → plan-09。
- Worker 自身搜索 SQL 的来源范围限定 → plan-09。