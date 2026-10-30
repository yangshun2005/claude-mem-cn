# 计划主控

本仓库中每个未关闭的 bug 都恰好归属于一个**计划主控**——它描述的是架构缺陷，而非表面症状。表面症状会被单独提交，通过重定向评论归入其主控，并以*不计划修复*关闭。工作单元是主控；每个集群一个 PR，原子性地关闭其所有子项。

GitHub Issue 是公开跟踪器。此处的文档是设计。两者互相引用；当二者不一致时，设计以文档为准，状态以 Issue 为准。

| 计划 | 主控 | 文档 | 架构缺陷 |
|---|---|---|---|
| plan-12 | [#2785](https://github.com/thedotmack/claude-mem/issues/2785) | [12-provider-and-extensibility-roadmap.md](12-provider-and-extensibility-roadmap.md) | 全新能力而非缺陷——提供方、数据摄取、集成、UX |
| plan-14 | [#3602](https://github.com/thedotmack/claude-mem/issues/3602) | [14-child-process-ownership.md](14-child-process-ownership.md) | worker 派生的每个进程都随 worker 一同消亡 |
| plan-15 | [#3603](https://github.com/thedotmack/claude-mem/issues/3603) | [15-worker-port-and-liveness-authority.md](15-worker-port-and-liveness-authority.md) | 对"是否有 worker 在服务此端口？"给出唯一可信答案 |
| plan-16 | [#3604](https://github.com/thedotmack/claude-mem/issues/3604) | [16-canonical-install-identity-and-bundle-integrity.md](16-canonical-install-identity-and-bundle-integrity.md) | 一个可解析、依赖完整的插件根目录；回收操作不能杀死正在工作的 worker |
| plan-17 | [#3605](https://github.com/thedotmack/claude-mem/issues/3605) | [17-hook-wrapper-contract.md](17-hook-wrapper-contract.md) | Hook 包装器：轻量、低开销、在所有宿主上失败开放 |
| plan-18 | [#3606](https://github.com/thedotmack/claude-mem/issues/3606) | [18-observer-response-pipeline.md](18-observer-response-pipeline.md) | 对每个输出分类，失败时绝不确认批次，对每段历史设界 |
| plan-19 | [#3607](https://github.com/thedotmack/claude-mem/issues/3607) | [19-observer-subprocess-isolation.md](19-observer-subprocess-isolation.md) | 为无头生成器显式指定 env、cwd、配置、工具集和认证来源 |
| plan-20 | [#3608](https://github.com/thedotmack/claude-mem/issues/3608) | [20-project-identity-and-injection-scope.md](20-project-identity-and-injection-scope.md) | 捕获、同步和检索共享一个项目解析器；注入配置能到达查询 |
| plan-21 | [#3609](https://github.com/thedotmack/claude-mem/issues/3609) | [21-sqlite-schema-evolution-and-queue-state.md](21-sqlite-schema-evolution-and-queue-state.md) | 一条由内省守护的 DDL 路径，不可变的会话键，可转换的队列状态 |
| plan-22 | [#3610](https://github.com/thedotmack/claude-mem/issues/3610) | [22-chroma-sidecar-contract.md](22-chroma-sidecar-contract.md) | 宿主安全的 sidecar 派生，单写入者 upsert 同步，诚实的回退——或弃用 sidecar |
| plan-23 | [#3611](https://github.com/thedotmack/claude-mem/issues/3611) | [23-host-integration-contracts.md](23-host-integration-contracts.md) | 每个非 Claude Code 适配器都通过针对宿主实际 Schema 的契约测试验证 |
| plan-24 | [#3618](https://github.com/thedotmack/claude-mem/issues/3618) | [24-server-runtime-generation-and-sync-contract.md](24-server-runtime-generation-and-sync-contract.md) | worker 和 server 共享一条生成流水线；每条写入路径都有会话关联 |
| plan-25 | [#3982](https://github.com/thedotmack/claude-mem/issues/3982) | [25-search-read-path-fts-cjk.md](25-search-read-path-fts-cjk.md) | 当 FTS5 unicode61 返回 0 时进行 CJK 感知匹配 |
| plan-26 | 暂无 | [26-telegram-session-wrapups.md](26-telegram-session-wrapups.md) | 每个会话从 Stop 摘要生成一条 Telegram 总结，按项目路由并记账；观察告警默认关闭 |

早期计划（01–11、13）已发布或已并入上述主控；`02`、`04` 和 `08` 作为历史设计文档保留于此。

## 路由新 bug

将症状与上述主控进行模式匹配，并问：*那里描述的修复是否也能修复此问题？* 如果是，向主控添加一条 Round-N 评论，注明子项、一行症状、1–3 行修复草案以及任何新的测试矩阵单元，然后以*不计划修复*关闭子项，并附：

> 合并至 #\<MASTER\>（plan-XX）。根因和修复顺序在该处与集群其余部分一同跟踪——请关注该 Issue 以了解进展。

抵制新开主控。大多数 bug 都是现有计划的子项。有两类情况合理地不纳入此方案：没有共同根因的真正功能请求（归入 plan-12），以及支持性问题——后者应直接回答而非路由。

## 健康检查

- **墓地主控**——5 条以上 Round-N 评论却无发布 PR。强制出一个 PR 或拆分计划。
- **过宽主控**——子项的修复无法装进一个 PR。拆分为两个更窄的计划。
- **表面聚类主控**——子项共享主题但不共享修复。按根因重新聚类。
- **漂移**——主控正文与文档不一致。从 Issue 重新生成文档的镜像。