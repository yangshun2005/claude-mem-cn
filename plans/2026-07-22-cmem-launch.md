# cmem.ai 发布计划 — 终点线

*创建于 2026-07-22。这是一份整合计划：把已验证的工作（已开启的 PR、冻结的 Turbopuffer 切换计划、已审计的漏斗规格）编排为可在全新会话中通过 `/do` 执行的阶段。每个阶段自包含。状态已于 2026-07-22 实时核验（PR 状态、部署、数据库健康、测试数量）。*

## 仓库与位置

- **claude-mem**（插件 + 同步中枢）：工作树 `/Users/alexnewman/.superset/worktrees/df8069a7-eb08-4626-9d3d-918d1e12eb9f/freckle-nail`，分支 `feat/phase5-two-lane-sync`，远程 `thedotmack/claude-mem`。中枢 worker 位于 `workers/sync-hub/`。
- **claude-mem-pro**（cmem.ai）：`~/Scripts/claude-mem-pro`（当前检出在 `codex/turbopuffer-only-launch`；`main` 自动部署到 Vercel 生产环境）。远程 `thedotmack/claude-mem-pro`。
- **切换计划**：`PLAN-postgres-to-turbopuffer-cutover.md`（位于 claude-mem-pro 工作树 `.claude/worktrees/tpuf-content-plan`，分支 `docs/tpuf-content-migration-plan`）。

## 仅人工事项（Alex，随时可做 — 不阻塞阶段 1）

- [ ] 在 Supabase 组织（`CMEM.ai` 项目 `ziczmqtpmaxbornfghye`）上绑定银行卡 / 自动扣款，确保计费再也不会暂停控制平面（7 月 20–22 日故障的根因）。
- [ ] 阶段 2 交付构建好的 v13.12.0 发布版本后执行 `npm publish`（按版本升级工作流，npm 发布仅限人工）。
- [ ] 可选：如需 deck 分享链接，导出 `WOWERPOINT_API_BASE` / `WOWERPOINT_UPLOAD_TOKEN` / `WOWERPOINT_VIEWER_BASE`。

## 已知状态（2026-07-22 核验，除非被推翻否则可信）

- PR #3333（claude-mem，双通道同步，6 个阶段）— 开启，代码完成，2,465 个测试通过，39/39 端到端测试通过。中枢测试套件今日重跑 68/68 通过。
- PR #51（claude-mem-pro，`GET /api/pro/sync/verify`）— 开启，通过，分支 `feat/sync-hub-token-verify`。
- sync-hub worker 已于 7 月 19 日部署到 Cloudflare 生产环境；`AUTH_CACHE` KV 已存在；`DISCORD_WEBHOOK_URL` 密钥已设置；`TOKEN_VERIFY_URL` 尚未设置 → 中枢按设计失败关闭。
- v13.11.0 是 npm 最新版本（旧版同步协议）。SyncHub 激活需要 v13.12.0（`connect-info` 根据 `13.12.0` + 灰度开关对 worker 协议 v1 进行门控）。
- 生产数据库在 7 月 20–22 日暂停后已恢复且健康；注册正常；251 条故障期间线索已恢复到 `email_waitlist`。
- Postgres 的 `pro_*` 内容表仍服务所有读取（仪表盘、MCP、时间线、设备、SSE）。Turbopuffer v1 仅是有损搜索镜像。
- 7 月 20 日产生的兄弟分支存在：`origin/codex/phase0-sync-verify`、`origin/codex/phase2-tpuf-product-paths` — 审查以复用；它们最近 3 次预览部署在 Vercel 上失败（构建错误），因此视为 WIP 输入，而非事实。

---

## 阶段 1 — 合并并激活同步中枢

**目标：** 中枢在生产环境上线，用真实令牌完成端到端验证。

任务：
1. 在 claude-mem-pro 中：将 PR #51 合并到 `main`（`gh pr merge 51 -R thedotmack/claude-mem-pro`）。Vercel 自动部署。部署后验证 `GET https://cmem.ai/api/pro/sync/verify` 有响应（401/契约响应，而非 404）。
2. 在 claude-mem 中：合并 PR #3333（`gh pr merge 3333 -R thedotmack/claude-mem`）。
3. 配置中枢 worker（`workers/sync-hub/`）：设置 `TOKEN_VERIFY_URL=https://cmem.ai/api/pro/sync/verify`（按 `wrangler.jsonc` 使用 wrangler secret 或 var — 注意工作树中有未提交的 `wrangler.jsonc` 修改；先协调它）。按 `workers/sync-hub/DEPLOY.md` 确认看门狗 cron 触发器和密钥。确保生产环境中 `DEV_ALLOW_ANY_TOKEN` 为空。重新部署。
4. 用真实账户从 `/api/pro/connect-info` 获取的 `setup_token`，对生产环境运行双设备金丝雀测试（`workers/sync-hub/canary/`）。
5. 演练一次熔断开关（触发 → 验证全舰队 `X-Sync-Mode: poll` + WS 以 503 拒绝 → 解除）。

验证清单：
- [ ] `/api/pro/sync/verify` 在 cmem.ai 上线
- [ ] 中枢认证用真实令牌往返成功；用错误令牌失败关闭
- [ ] 金丝雀：两台设备推送/拉取收敛，WS 通知触发
- [ ] 观察到熔断开关触发/解除
- [ ] 看门狗向 Discord 发送每小时心跳

反模式防护：不要在生产环境设置 `DEV_ALLOW_ANY_TOKEN`；不要通过桩实现 verify 来绕过失败关闭默认值。

## 阶段 2 — 发布 v13.12.0

**目标：** 一个插件包支持中枢协议的发布版本，供 Alex 执行 `npm publish`。

任务：
1. 在 claude-mem 已合并的 `main` 上，运行 `version-bump` 技能流程升级到 `13.12.0`（清单：根 `package.json`、`plugin/plugin.json`、marketplace.json；通过 `npm run build-and-sync` 构建；验证 worker 启动；打标签 + GitHub 发布）。
2. 确认构建出的 `plugin/scripts/worker-service.cjs` 包含 SyncHub 客户端路径（`/v1/sync/ops`、`/v1/sync/changes`），且 `CLAUDE_MEM_CLOUD_SYNC_HUB_URL` 默认为空（除非配置否则同步关闭）。
3. 交给 Alex 执行 `npm publish`。发布后，按 `connect-info` 中的分阶段灰度开关，在 Vercel 上开启 `CMEM_WORKER_PROTOCOL_V1_ONBOARDING_ENABLED` + 金丝雀用户 ID。

验证清单：
- [ ] 发布的包中 `grep -c "/v1/sync/" plugin/scripts/worker-service.cjs` > 0
- [ ] 全新安装冒烟测试：插件启动，无中枢 URL 时同步保持关闭
- [ ] Alex 发布后 `npm view claude-mem dist-tags` 显示 13.12.0

## 阶段 3 — 执行 Turbopuffer 切换

**目标：** Turbopuffer v2 是唯一内容存储；Postgres 仅作控制平面。本阶段执行现有冻结计划 — 开始前完整阅读 `PLAN-postgres-to-turbopuffer-cutover.md`；其契约（规范文档 Schema、稳定 ID 公式、`chronological_key` 分页、`(epoch, seq)` 投影检查点、实体版本/墓碑语义）具有约束力。

子步骤（计划自身的阶段 0–3）：
1. **确认前提**：统计 `pro_observations` / `pro_summaries` / `pro_prompts` 中的行数；将现有数据归类为可丢弃的开发数据并有意清除。添加缺失的 `verify:tpuf-content` 脚本。
2. **构建 v2 存储 + 投影**：`cmem-{userId}-v2` 命名空间、规范文档、由无状态中枢 worker 在 `pushOps` 后调用的投影端点、检查点 + 每用户租约、定时修复任务。解决压缩水位线缺口：投影器必须持有中枢压缩水位线（注册为设备或添加专用投影器水位线），使压缩无法超越它。
3. **切换所有产品路径**：仪表盘 obs/summaries/prompts 路由、MCP 工具（全行读取，不仅是搜索）、时间线、统计、设备计数，并用有界轮询或中枢通知替换 Supabase Realtime SSE 流。审查 `origin/codex/phase2-tpuf-product-paths` 以复用差异（其 Vercel 构建失败 — 修复或重写，不要盲目信任）。
4. **删除旧世界 + 接入门控**：通过经审查的迁移删除 `pro_observations`、`pro_summaries`、`pro_prompts`、`pro_sync_state`；仅在 v13.12.0 在野外取代 v13.11.0 后移除旧版批量同步路由；从 `pro_users` 中剥离过时的 `*_migrated` 列。

验证清单：
- [ ] 端到端：本地写入 → 中枢 → 投影 → Turbopuffer → MCP 工具回答返回该行
- [ ] 超过 10,000 行时分页正确（`chronological_key`，不透明游标）
- [ ] claude-mem-pro 服务代码中 `grep -r "pro_observations\|pro_summaries\|pro_prompts" src/` 无结果
- [ ] 仅在 npm 最新版本 ≥ 13.12.0 后移除旧版路由

反模式防护：不要整体采纳 `codex/turbopuffer-only-launch`（认证融合 tpuf 设计是已记录的失败模式）；绝不用 `created_at_epoch` 作为同步游标（仅用中枢 `seq`）；Durable Object 不得有出站 I/O；认证/会话/计费代码保持不动（Supabase 仍是身份提供方）。

## 阶段 4 — MCP 连接页 + 引导测试提示（MVP 界面）

**目标：** 通往已审计规格的漏斗前门。数据源是 `/api/pro/connect-info`（两个令牌、MCP URL `{base}/api/mcp/mcp`、添加命令、mcp.json 片段的唯一事实来源）。

规格（来自 7 月 12 日产品流程审计）：
- 掩码显示 MCP 链接，带复制成功状态、轮换和撤销操作。
- Claude / ChatGPT / Gemini 的大客户端卡片，保持"按下"状态并显示各客户端说明。
- 真实连接验证：按客户端跟踪最后一次成功的 MCP 请求（见 main 上的 `src/app/api/pro/mcp-activity`），而非自我声明。
- 连接后的引导测试提示 — 一个建议提示，能明显召回用户已同步的记忆（顿悟时刻）— 然后展示信息流/搜索。
- 多个本地安装注册为同一账户下的设备，而非独立身份。

验证清单：
- [ ] 全新账户走查：注册 → 结账（免费模式可）→ 连接页 → 将链接粘贴到至少一个真实客户端 → 验证翻转为已连接 → 测试提示返回真实记忆
- [ ] 轮换在 60 秒缓存窗口内使旧 `mcp_token` 失效；`setup_token` 不受影响
- [ ] 仪表盘不再为未连接用户绕过激活（查看者优先 P0）

## 阶段 5 — 发布门控 + 演练

**目标：** 让发布可存活的那些无聊安全护栏。

任务：
1. 通过 `copy_from_namespace` 进行每日跨区域 Turbopuffer 备份 + 备份时效告警。
2. 从备份进行一次完整恢复/重放演练 — 记录在案。
3. 告警：投影器延迟、被拒操作、Turbopuffer 写入失败 — 发送到 Discord（复用看门狗模式）。
4. 控制平面正常运行时间 + 计费告警（7 月 20–22 日的教训）：一个外部检查命中数据库支撑的端点（而非静态落地页），失败时呼叫；Alex 侧的 Supabase 计费告警。
5. 完整演示演练：结账 → 连接 → 粘贴链接 → 代理记住 → 信息流展示。最后再跑一次同步端到端矩阵（39 个场景）和完整测试套件。

验证清单：
- [ ] 备份存在、新鲜，且成功恢复过一次
- [ ] 每个告警在强制测试中至少触发一次
- [ ] 演示路径录制/演练无故障
- [ ] 所有套件通过（根、中枢、WS、openclaw、端到端矩阵）

---

## /do 执行说明

- 阶段严格有序；1 和 2 较小，3 是大块（其子步骤可拆为独立会话），4 可与 3 重叠（它通过 `connect-info` 和 MCP 端点读取，与后端存储无关），5 最后。
- 生产部署发生在阶段 1–3（合并时 Vercel 自动部署，中枢用 wrangler 部署）。阶段 5 通过前不要宣布任何事。
- 恢复的故障期间线索（251 行，`source` 完好）已在 `email_waitlist` 中，用于宣布邮件。