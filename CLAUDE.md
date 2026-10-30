# Claude-Mem：AI 开发说明

Claude-mem 是一个 Claude Code 插件，提供跨会话的持久记忆。它捕获工具使用情况，使用 Claude Agent SDK 压缩观察结果，并将相关上下文注入未来的会话。

## 构建

```bash
npm run build-and-sync        # 构建、同步到市场、重启 worker
```

## 文件位置

- **源码**：`<project-root>/src/`
- **构建后的插件**：`<project-root>/plugin/`
- **已安装的插件**：`~/.claude/plugins/marketplaces/thedotmack/`
- **数据库**：`~/.claude-mem/claude-mem.db`
- **Chroma**：`~/.claude-mem/chroma/`

## 要求

- **Bun**（所有平台 - 缺失时自动安装）
- **uv**（所有平台 - 缺失时自动安装，为 Chroma 提供 Python）
- Node.js

## 文档

**公开文档**：https://docs.claude-mem.ai（Mintlify）
**源码**：`docs/public/` - MDX 文件，编辑 `docs.json` 以调整导航
**部署**：推送到 main 后从 GitHub 自动部署

## 重要

无需编辑变更日志，它会自动生成。

## 托管

不使用按量计费的 Cloudflare 计算。新的服务端工作放在 Fly 或 Vercel 上。Cloudflare Workers、Durable Objects、Queues、Workflows、crons 或 Workers AI 需要先经负责人批准的成本对比。免费的 Cloudflare 服务（DNS、R2、Email Routing）可以使用。

## 模型策略

切勿将带日期的 Claude 模型 ID（如 `claude-haiku-4-5-20251001`）作为默认值固定在安装程序或文档中。使用 Claude Code 的别名（`haiku`、`sonnet`、`opus`），它们始终解析为该层级的最新模型。Anthropic 的最新模型价格更低或同价性能更强，因此固定 ID 会在新模型发布后悄悄让每次安装变得更慢、更贵。用户仍可主动输入完整 ID 来固定某个版本。当发现旧默认值被固定时，添加设置迁移，将已发布的 ID 改为别名（参见 `src/shared/SettingsDefaultsManager.ts` 中的 `PINNED_SHIPPED_HAIKU_MODELS`）。