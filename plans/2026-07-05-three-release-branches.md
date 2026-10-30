# 计划：三条长期发布分支（stable / core-dev / community-edge）

负责人：独立维护者（thedotmack）。仓库：github.com/thedotmack/claude-mem。
目标：将三个计划 PR 转为三条永久发布线，在一处记录该策略，并为非 stable 线提供克隆/运行说明。

## 三条线（事实来源）

| 线              | 分支             | 面向对象                                     | 发布到 npm？      |
|-----------------|------------------|----------------------------------------------|-------------------|
| Stable          | `main`           | 所有人。`npx claude-mem` 安装。              | 是（唯一发布线）  |
| Core Dev        | `core-dev`       | 维护者 + 需要根因修复的测试者                | 否——从源码运行   |
| Community Edge  | `community-edge` | 最前沿，集成社区 PR                          | 否——从源码运行   |

流程（最简单、低仪式）：三条线均从 `main` 出发。`main` **向前**合并到 `core-dev` 和 `community-edge` 以保持同步。已验证的修复通过常规 PR **回推**到 `main`。无门禁、无强制审查者——由独立维护者自行决定。

## 事实（2026-07-05 核实）

- PR #3141 `plan/stable-working-build` @ `f6c7f51` — 基于 `main`。STABLE 落地。
- PR #3143 `plan/root-cause-holistic-fixes` @ `c81acee` — 转为 `core-dev`。
- PR #3142 `plan/community-bleeding-edge` @ `a20d1ac` — 转为 `community-edge`。
- origin 上尚无 `core-dev` / `community-edge` 分支。
- 文档：`docs/public/*.mdx`，导航在 `docs/public/docs.json`（第 79 行的 "Configuration & Development" 分组）。
- README 贡献章节：README.md 约第 374 行。
- 非 stable 运行路径 = 克隆 + `git checkout <branch>` + `npm install` + `npm run build-and-sync`。

## 执行前需决策（Phase 1）

两个源 PR #3143 和 #3142 的存在是为了将其内容合并到 `main`。但我们**不**将它们合并到 `main`——其内容改为成为永久的 `core-dev` / `community-edge` 分支。因此分支推送后，这两个 PR 即冗余。

**建议：** 从 PR head 创建分支，然后**关闭 #3143 和 #3142**，附评论："已提升为长期分支 `core-dev` / `community-edge`；现为永久发布线，而非合并到 main。" 保留 #3141 开启（或合并）作为 stable 落地。

Phase 1 执行关闭前需确认。

---

## Phase 1 — 创建并发布分支

1. 从 PR #3143 head 创建 `core-dev`：
   `git branch core-dev c81acee39a771848bd30ee26e3c52ec26d713cd1`
   `git push origin core-dev`
2. 从 PR #3142 head 创建 `community-edge`：
   `git branch community-edge a20d1ac44144d4749acfd68faceda2794f3187f0`
   `git push origin community-edge`
3. （按上述决策）关闭 #3143 和 #3142，附重定向评论。

**验证：** `git ls-remote --heads origin | grep -E 'core-dev|community-edge'`
显示两个 ref 位于预期 SHA。

## Phase 2 — 编写分支策略文档

创建 `docs/public/branches.mdx`。内容（照此结构，勿自创）：

- Frontmatter：`title: "Release Branches"`，description 一句话。
- 章节 "The three lines"：上表（stable/core-dev/community-edge、面向对象、是否发布到 npm）。
- 章节 "How changes flow"：`main` 向前合并到两条前沿线以保持同步；已验证的修复通过 PR 回推到 `main`。一张图/一行，无政策仪式。
- 章节 "Which one should I use?"：常规使用选 Stable；core-dev 用于提前测试根因可靠性修复；community-edge 用于最新社区集成（最不稳定）。
- 章节 "Run a non-stable line locally"：
  ```bash
  git clone https://github.com/thedotmack/claude-mem.git
  cd claude-mem
  git checkout core-dev          # or: community-edge
  npm install
  npm run build-and-sync         # builds + syncs to local marketplace + restarts worker
  ```
  注意：仅 `main` 发布到 npm，因此 `npx claude-mem@latest` 始终 = stable。
  回到 stable：`git checkout main && npm run build-and-sync`。
- 章节 "Releasing"（维护者）：发布（`npm run release`、标签、publish）仅在 `main` 上进行。前沿线从源码运行，永不发布。

**验证：** 文件存在、frontmatter 有效、无自创 npm 脚本（仅真实脚本：`build-and-sync`、`release`、`release:patch|minor|major`）。

## Phase 3 — 接入导航 + README

1. `docs/public/docs.json`：将 `"branches"` 加入 "Configuration & Development" 分组的 pages 数组（在 `"development"` 之后）。保持 JSON 有效。
2. `README.md` 贡献章节（约第 374 行）：加一行指向 Release Branches 文档——"claude-mem 从三条分支发布：`main`（stable）、`core-dev`、`community-edge`。见 [Release Branches](https://docs.claude-mem.ai/branches)。"

**验证：** `node -e "JSON.parse(require('fs').readFileSync('docs/public/docs.json','utf8'))"`
干净退出。README 渲染出链接。

## Phase 4 — 最终验证

- 两条分支均以正确 SHA 推送（`git ls-remote`）。
- `docs.json` 可解析。
- `branches.mdx` 存在且在导航中。
- README 已更新。
- （若已决定）#3143 和 #3142 已关闭并附重定向评论；#3141 状态不变。
- 勿编辑 CHANGELOG（自动生成）。