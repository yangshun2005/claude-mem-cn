# Claude-Mem 的 Cursor 规则

## 自动上下文注入

`context-inject.sh` Hook **自动创建并更新**以下位置的规则文件：

```
.cursor/rules/claude-mem-context.mdc
```

该文件：
- 设置了 `alwaysApply: true`，因此会包含在每个聊天会话中
- 包含过去会话的近期上下文
- 每次提交提示词时自动刷新

**你无需手动创建任何规则文件！**

## 可选：附加指令

如果你想添加关于 claude-mem 的自定义指令（超出自动注入的上下文），请创建一个单独的规则文件：

### `.cursor/rules/claude-mem-instructions.mdc`

```markdown
---
alwaysApply: true
description: "使用 claude-mem 记忆系统的指令"
---

# 记忆系统用法

你可以访问 claude-mem，一个持久化记忆系统。除了上述自动注入的上下文外，你还可以使用 MCP 工具搜索更详细的信息：

## 可用的 MCP 工具

1. **search** - 查找相关的过往观察记录
   ```
   search(query="authentication bug", project="my-project", limit=10)
   ```

2. **timeline** - 获取特定观察记录前后的上下文
   ```
   timeline(anchor=<observation_id>, depth_before=3, depth_after=3)
   ```

3. **get_observations** - 获取特定 ID 的完整详情
   ```
   get_observations(ids=[123, 456])
   ```

## 何时搜索记忆

- 当用户询问之前的工作或决策时
- 当在本项目中遇到不熟悉的代码模式时
- 当调试可能之前已处理过的问题时
- 当被要求继续或基于之前的工作进行开发时

## 三层工作流

遵循此模式以节省 token：
1. **先搜索** - 获取紧凑索引（约 50-100 token/条结果）
2. **时间线** - 获取感兴趣结果前后的时间顺序上下文
3. **获取详情** - 仅针对相关观察记录（约 500-1000 token/条结果）

未经筛选前，切勿获取完整详情。
```

## 文件位置

| 文件 | 用途 | 创建者 |
|------|---------|------------|
| `.cursor/rules/claude-mem-context.mdc` | 自动注入的上下文 | Hook（自动） |
| `.cursor/rules/claude-mem-instructions.mdc` | MCP 工具指令 | 你（可选） |

## Git 忽略

如果你不想提交自动生成的上下文文件：

```gitignore
# .gitignore
.cursor/rules/claude-mem-context.mdc
```

指令文件可以提交，以便与团队共享。