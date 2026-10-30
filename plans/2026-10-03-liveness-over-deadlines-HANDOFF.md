# 交接 — 活性优先于截止时间 + Supabase 云同步

写于 2026-10-04 ~00:45 UTC，上下文清理前。计划：`plans/2026-10-03-liveness-over-deadlines.md`
（其中 Phase 12 是修订后的切换顺序 — 按它执行）。

## 当前状态

**claude-mem** — worktree `~/.superset/worktrees/claude-mem 🧠/numerous-meadowlark`，分支
`feat/liveness-over-deadlines`，**PR #4368**（交接时 head 为 `b3ced5fe6`）。
- Phase 1–7（never-pay-twice、SSE/idle fetch、provider streaming、/api/ready、hook spool、context cache、
  corpus SSE）、Phase 11 客户端（SyncClient 中的 Supabase Realtime）、sync-api 的 FORWARD_ORIGIN 代理模式、
  CI 全量测试修复以及两轮 Greptile 修复均已提交并推送。
- `b3ced5fe6` 上 CI 为绿色（8 通过，3 跳过）。
- **未完成：** 2 个 Greptile 讨论串，均位于 `src/services/sync/SyncClient.ts:721`：
  - `PRRT_kwDOPng1J86otZRM` — 即使拉取过程中收到 `advance`，catch-up 仍会标记为已追上。修复：记录加入以来
    通告的最大 head_seq；caught-up 需要一次加入后的拉取，且 `more=false` 且 cursor ≥ 该 head；
    被 single-flight 跳过的 advance 需安排一次后续拉取。
  - `PRRT_kwDOPng1J86otZRO` — 达到上限的周期之后的后续拉取绕过了最小请求间隔。修复：让后续拉取走
    最小间隔调度器（若没有则新增 `MIN_PULL_GAP_MS`）。
  - 交接时有子代理正在实现这两项。检查 worktree 中的 `git status`：若这些改动未提交地存在，验证
    （`bun test tests/worker/sync`、`npm run typecheck`、全量 `bun test tests`），提交、推送、回复并解决两个讨论串。
    若不存在，重做。
- 已知非回归问题：`field-deadline-wire` 首个测试仅在本地 Bun 1.3.9 失败（CI 1.4.2 通过）。
- 之后持续跟进，直到 CI 绿色且 0 个未解决讨论串；然后合并。

**claude-mem-pro** — worktree `~/Scripts/claude-mem-pro/.claude/worktrees/supabase-sync`，分支
`feat/supabase-cloud-sync`（已推送；最新 `ae998e7`；**尚未开 PR**）。迁移 0064–0070、Edge
Functions `cmem-sync` + `embed`、SupabaseContentStore、summary-landed 路由、回填脚本。

**生产 Supabase**（项目 `ziczmqtpmaxbornfghye`，"CMEM.ai"）：
- 0064–0070 已应用；`cmem-sync` + `embed` 已部署；ES256 密钥已作为 **standby** 导入（kid `1236177f…`）；
  函数 secrets + Vault secrets 已设置。
- 冒烟账号 `cmem-sync-smoke+20261003@cmem.ai`（用户 `f8361ee9-da08-4b47-a61e-76b65c0f3588`，凭据在 Pro
  `.scratch/prod-smoke-account.env`）针对生产通过完整 e2e，含 Realtime。
- 所有者账号 `453c8562-f3fa-48a8-a47a-f7035bde44e3` 已回填（165k 行）。
- 磁盘 12 GB（已用 8）。完整回填需要约 65 GB。
- Pro `.scratch/` 中的辅助脚本：`mgmt.sh`（Management API，token 取自 keychain）、`prodsql.sh`（需先
  `vercel env pull` 到 `.scratch/prod.env`；用后删除）。Vercel 的 secrets 在 `.scratch/prod-secrets.env`。
- 尚未切换任何面向用户的内容：旧客户端仍通过 Fly `sync.cmem.ai` → Neon → tpuf 同步；Pro 仍
  读取 tpuf（main 分支）。

## 等待中
1. SyncClient 修复子代理（见上）。
2. **Supabase 磁盘冷却于 2026-10-04 03:22 UTC 结束**（磁盘变更每 4 小时允许一次）。

## 接下来，按顺序
1. 完成 PR #4368 → 合并。
2. ≥03:22 UTC：`POST /v1/projects/ziczmqtpmaxbornfghye/config/disk`
   `{"attributes":{"type":"gp3","size_gb":200,"iops":3000,"throughput_mibps":125}}`（约 $24/月；无法缩小）。
   通过 GET `/config/disk` 确认。然后 `nohup` 运行完整回填
   （`scripts/backfill-cmem-content-from-tpuf.ts --apply --all`，约 9.3M 行，约 10 小时，幂等，跳过 embedding
   队列）→ `.scratch/backfill-all.log`；监控磁盘（空闲 < 25 GB 则暂停）。实时写入导致的少量按用户 MISMATCH
   属预期；标记 > 1% 的用户。
3. 回填后：设置 Vercel 环境变量 `CMEM_SUMMARY_LANDED_SECRET`、`CMEM_EMBED_SECRET`；打开并验证 Pro PR 构建；
   然后**紧接着**：在 `services/sync-api` 中执行
   `fly secrets set FORWARD_ORIGIN=https://ziczmqtpmaxbornfghye.supabase.co/functions/v1/cmem-sync`
   + `fly deploy`（回滚：unset），并合并 Pro PR（Vercel 部署）。
   验证 `curl https://sync.cmem.ai/health` → `mode:"forward"`，以及 Pro 仪表盘中所有者账号的计数。
4. claude-mem：在 `migratedCloudSyncHubUrl`（`SettingsDefaultsManager.ts`）中将 `https://sync.cmem.ai`
   映射到函数 URL，然后 `/version-bump`（npm 发布是人工步骤）。
5. 7 天无问题后：将 Fly 缩容到 0，删除 Neon，删除 tpuf 命名空间 + 旧版擦除步骤，删除
   `services/sync-api/` 和 `workers/sync-hub/`。