# 记忆奖评分卡：17 个黑客松项目，按它们实际如何使用 Claude-Mem 评分

*2026 年 8 月 23 日。1000 美元"构建一个真正会记忆的 agentic 工具"挑战赛收到 17 份提交，每份都被克隆、阅读，并按五个维度评分：集成深度、类别契合度、能否运行、周一早上是否有人会用、README 是否说真话。下面每个分数都附有支撑它的文件和行号。*

## 评审方式

Claude-Mem 是第二个 agent，它观察你的主编码 agent 工作，把结构化观察写入本地 SQLite 数据库，并在下次会话开始时交回一份标题优先的时间线。本次奖项要求黑客在此基础上构建：热启动、时间线驱动的检索、记忆 UI、与其他工具的集成、自定义摄取模式、对观察的实时反应，或节省 token 的速度玩法。决胜标准：真正的开发者周一会不会用它。

我们克隆了全部 17 个仓库，阅读了每个涉及 claude-mem 的文件，并把每项声称的集成追踪到确切的代码行。然后我们提出五个问题，每项按 0 到 5 分评分：

1. **集成深度。** 0 分表示 claude-mem 完全没出现。1 分表示只在文档里被提及。2 分表示浅层读取 SQLite 数据库，或对 worker 发起一次 HTTP 调用。3 分表示在产品核心循环中真正读写观察。4 分表示多个层面：Hook、模式、搜索、时间线、技能。5 分表示 claude-mem 就是架构，没有它产品就不存在。
2. **类别契合度。** 项目落在七个奖项类别中一个或多个的程度。
3. **可运行性。** 0 分是空谈。3 分是配置后能跑。5 分是带测试、可端到端演示。
4. **周一早上。** 决胜项。开发者是否真的会用。
5. **诚实度。** README 是否与代码一致。5 分表示一致。1 分表示宣传描述的是代码没做的事。

在进入项目之前，先说主要发现：**没有人在集成深度上拿到 4 分或 5 分。** 没有一个项目使用 claude-mem 的时间线、`get_observations` 工具、MCP 搜索服务器，或接入它的 observer。每个仓库都以一个压缩提交的形式提交，日期是黑客松当天。最常见的失败不是野心不足，而是管道问题：错误的端口号、错误的路由名、错误的设置键，以及静默返回空结果、宣传稿却称"真实可用"的 worker 调用。

以下是 17 个项目，从第一名开始排名。

---

## 1. Temporal (seanowww/temporal-clean) — 25 分中的 20 分

**它做什么。** Temporal 是 Pull Request 评审的来源追溯层。它把 GitHub PR 和提交事件，以及来自 Slack、Jira、Notion、Claude Code 和 Codex 的证据摄取到一张图中，从 PR 向外做带评分的遍历，渲染出有证据支撑的时间线，GitHub Action 可将其作为 PR 评论发布。宣传语：评审 PR 时，你该看到的不只是 diff，还有导致它的决策、对话和 agent 推理。

Claude-mem 是七个连接器之一。Temporal 不摄取原始 agent 转录，而是读取 claude-mem 已压缩的观察、会话摘要和用户提示，因此 agent 决策落到时间线上时已经过提炼。

**为什么它该赢。** 这是唯一一份真正理解 claude-mem 数据模型、且该模型承重的提交。`lib/claude-mem/client.ts` 中的客户端用与 claude-mem 自身相同的公式推导 worker 端口，按 `hasMore` 标志分页遍历 `/api/observations`、`/api/summaries` 和 `/api/prompts`，并正确解包响应信封。worker 宕机时，它回退到直接以只读方式打开 `~/.claude-mem/claude-mem.db`，用 `PRAGMA table_info` 检查保护每个列名，因此 schema 漂移不会破坏它。

归一化层才见心思。观察类型决定产物重要性：决策得 5 分，变更得 2 分。会话摘要的下一步变成 QA 检查。私有标签包裹的内容在边界处被剥离。Codex 会话通过读取 claude-mem 的 `platform_source` 字段路由到 Temporal 的 Codex 来源。由于 claude-mem 观察不带分支信息，团队写了一个 Claude Code SessionStart 和 PostToolUse Hook，把 git 状态记录到附属文件，再把每条观察解析到其时间戳时生效的分支。观察的 `files_modified` 与 PR 变更文件的重叠会增加检索分数。

47 个测试全部通过。其中 14 个是 claude-mem 专属，覆盖两种传输方式、分支解析、私有标签剥离，以及从 claude-mem 导出端到端组装产物。TypeScript 编译干净。README 的 claude-mem 部分与代码逐行一致。

**为什么它该输。** Claude-mem 是可选的。没有它产品也能完整运行，精修的演示数据中零条 claude-mem 记录，因此演示路径从未走通集成。它是只读的：没有回写、没有自定义模式、没有使用搜索或时间线检索。端口解析中有一个死分支，读取的是 claude-mem 不存储的设置键。而且它只是七个连接器之一，而非架构本身。

**排名与原因。** 第一名。集成深度 3，类别契合 4，可运行 5，周一早上 3，诚实度 5。它完全命中"构建集成"，部分命中"摄取任何内容"，并且是唯一把 claude-mem 的观察结构当作值得理解的东西、而非可计数或截断的 blob 的项目。

---

## 2. TeamBrain (MrFibonacc1/yc-hacks-greptileFast) — 25 分中的 18 分

**它做什么。** TeamBrain 把多位工程师的记忆汇入每个团队共享的大脑。一个零依赖的 Bun 服务器摄取每位工程师的 claude-mem 数据库，把每条记忆归属到作者，并在会话开始时注入团队简报。它还把 Greptile 和 PR 评审对话作为记忆摄取，把反复出现的教训"毕业"为跨团队模式，并运行一个 PR 哨兵，标记重复其他团队已犯错误的 diff。标语：Claude-Mem 让一个工程师的 agent 变聪明；TeamBrain 让整个团队的 agent 变聪明。

**为什么它该赢。** 这是最有野心、类别最密集的提交。它同时命中热启动、记忆 UI、集成，以及带自定义模式的摄取。claude-mem 集成正确使用了 claude-mem 的三种格式。它带恢复游标直接读取 `~/.claude-mem/claude-mem.db`，并拒绝敏感类型的行。它实现了 cmem.ai 云同步中心线协议的客户端，路由和请求头与 claude-mem 的同步中心 worker 一致。它还附带自定义模式 `modes/code--team.json`，正确使用了 claude-mem 的双连字符父级继承；我们对照真实的 `code.json` 验证了覆盖中的每个提示键、类型 ID 和概念 ID。

198 个测试全部通过。我们启动了服务器、创建大脑、运行 SessionStart Hook，并端到端运行了 MCP 服务器。仪表盘有"Pool in Claude-Mem"面板。预检代码甚至把云路径标记为"未验证"，而非声称可用。

**为什么它该输。** 头条导入路径在真实安装上是坏的。`server/src/cmem.ts:430` 的导入器只要存在名为 `memory_items` 的表就优先用它。在标准的 claude-mem v13 安装中，该表存在但为空；数据在 `observations` 里。我们对一个拥有超过十万条观察的真实数据库运行导入，得到 `opsScanned: 0, imported: 0`。测试夹具只构建旧表，所以测试套件抓不到这个问题。项目范围过滤也应用在错误的分支，因此 Hook 中防止把全部历史汇入当前活跃大脑的守卫是空操作。

除 bug 之外：它是一个兄弟记忆系统，摄取 claude-mem 而非在其上构建。计划文档直言"实现我们自己的轻量 Hook 和服务器"。它不使用 worker API、MCP 搜索工具或时间线。自定义模式格式正确，但未安装，附有手动复制的说明。云同步路径只对桩测试过。

**排名与原因。** 第二名。集成深度 3，类别契合 4，可运行 4，周一早上 3，诚实度 4。团队负责人会真心想要这个。奖项应附带一行修复的条件：当 `memory_items` 为空时优先用 `observations`。

---

## 3. RegressionForge (KaushikSiva/regression-forge，外加 demo-ecom-store) — 25 分中的 16 分

**它做什么。** 一个部署后回归门禁。FastAPI 后端驱动 Playwright 走完店面结账，然后通过 API 验证订单、通过 Mailpit 验证确认邮件、验证履约 webhook，以及 SigNoz 中的关联日志。它输出确定性的 PASS、FAIL 或 NEEDS_REVIEW 证书，配有 React 证据室。配套仓库 ForgeCart 是故意可破坏的目标，带有良好、损坏和修复三种契约变体，以及一个在认证失败时阻止 PR 的 GitHub Actions 工作流。

**为什么它该赢。** 这是唯一搭起 claude-mem server-beta 运行时的团队。Dockerfile 克隆固定到特定提交的 claude-mem，Render 清单配置 Postgres、Valkey、带 API key 认证的 Web 服务和 worker。Python 客户端访问 `/v1/projects`、`/v1/search` 和 `/v1/memories`。我们对照 claude-mem 实际的路由处理器验证了载荷形状，完全一致。这些调用位于核心 `execute()` 循环内：浏览器步骤前召回，运行完成后保存。匹配结果进入 Codex 诊断包，并在 UI 的 MemoryRail 中渲染。单元测试 mock HTTP 层，断言确切的 URL、bearer 头和载荷。每个不可用状态都被暴露而非伪造。

**为什么它该输。** 召回查询是硬编码字面量：`"RegressionForge PASS"`。没有按工作流、部署或失败文本限定范围。返回的内容是产品自己的 JSON 摘要经全文索引回显，而非 observer 生成的观察；Chroma 和生成都被禁用，所以没有语义搜索，配置的 Anthropic worker 从不做任何事。README 的架构图显示记忆喂给门禁，但门禁是 `decide(step_results)`，记忆从不触及它。本地 `make demo` 根本没接 claude-mem；记忆路径只存在于手动引导的 Render 部署上，我们无法实时验证。演示商店仓库零 claude-mem 代码。

**排名与原因。** 第三名。集成深度 3，类别契合 3，可运行 3，周一早上 3，诚实度 4。一个真正接线、契约正确但浅层的 claude-mem 用法，把它当作托管的键值和全文存储，附加在一个强大的部署门禁产品上，而该产品用任何数据库都能同样工作。

---

## 4. Rewind (Ashutosh-Brain/greptile-fasthackathon-26) — 25 分中的 16 分

**它做什么。** 一个 Codex 优先的上下文记录。它通过 App Server 观察 OpenAI Codex 任务，把其证据与 git 提交关联，把有证据支撑的记录存为 git note 和 Supabase，让队友检查决策、添加反驳，并在隔离的 worktree 中 fork 修正后的重放。Electron 加 Next.js，带本地 HTTP 伴生服务。

**为什么它该赢。** claude-mem 客户端是准确的。它从设置或环境解析 worker，端口公式正确，访问 `/api/health`、`/api/stats`、`/api/processing-status`、`/api/observations`、`/api/summaries` 和 `/api/prompts`，这些路由全部存在且字段形状符合预期。它把所有内容归一化为带类型的 `memory.json`，写在每个 Codex 轨迹旁边。README 异常坦率：claude-mem 是"可选的"、"独立的增强层，从不覆盖源轨迹"。结果文档公开区分"插件已安装"、"worker 已连接"、"Hook 正在捕获"和"压缩已认证"，并承认压缩从未运行，因为 OAuth 会话过期了。

**为什么它该输。** 没有任何东西读取 `memory.json`。我们 grep 了 UI、上下文记录、Codex 问答、Supabase schema 和 GitHub 评论生成器：零消费者。它是只写的死数据。UI 显示带版本号和计数的状态卡，从不显示观察。影片预检把演示门禁在 worker 显示"已连接"上，尽管其数据不贡献任何场景；这张卡的存在是为了出现在结尾定版镜头里。按团队自己的结果，唯一真实的端到端产物是一条原始捕获的提示；零条压缩观察曾流过。测试完全 mock fetch。

**排名与原因。** 第四名，与 RegressionForge 同分但排名更低，因为 RegressionForge 确实读回记忆。集成深度 2，类别契合 2，可运行 4，周一早上 3，诚实度 5。一个构建良好的 Codex 工具，戴着一枚真实、正确实现、只写的 claude-mem 徽章。

---

## 5. SourceTether (abheeshtroy/sourcetether) — 25 分中的 15 分

**它做什么。** 一个源码验证的记忆门禁。它通过结构化 AST 指纹（对解析树做 SHA-256，忽略注释和空白）把手写的原子断言（如"gravity 是初始化为 9.81 的静态属性"）绑定到 TypeScript 声明。释放记忆前，它对照当前源码重新验证指纹。若声明已变，记忆被扣留，并返回精确的重读目标。一个 canvas 月球着陆器演示展示：使用过期地球重力记忆的控制器耗尽燃料，而重新验证过的控制器软着陆。

**为什么它该赢。** 这个想法与记忆陈旧问题真正相关，这是该领域没人解决的真实问题。42 个测试全部通过，TypeScript 干净；我们运行过。唯一一次 claude-mem 调用是对 `/api/observation/{id}` 的 GET，其字段映射我们对照 worker 的路由处理器验证过。README 极其诚实：它说"只获取一条本地 Claude-Mem 观察以建立来源"、"不自动推导断言"，并有明确的局限部分。

**为什么它该输。** 观察内容被故意丢弃。演示工作流复制 ID、时间戳、项目和会话 ID，扔掉文本；存储的断言是硬编码字符串常量。一个测试字面输入"必须永不返回的不可信 Claude-Mem 叙述"。claude-mem 依赖可被任何带 ID 和时间戳的对象替换，什么都不会变。默认端口是 37701，即作者自己 uid 推导的端口，覆盖它的环境变量从未文档化。它绑定一个文件、一个符号、一个断言；无法把任意观察绑定到任意符号，也没有任何东西流回 agent 上下文。

**排名与原因。** 第五名。集成深度 2，类别契合 2，可运行 5，周一早上 1，诚实度 5。一个干净、测试良好、诚实描述的 AST 门禁，把 claude-mem 当作来源徽章而非基底。

---

## 6. Product Git (Luciayuanzhu/product-git) — 25 分中的 14 分

**它做什么。** 产品行为的版本控制。一个 stdio MCP 服务器暴露六个工具，供编码 agent 开始回合、声明影响、提交产品状态和结束。一个 localhost Web 工作台让人接受、拒绝或锁定 agent 对 Product Graph 的声明变更，并保存 Product Versions。下一个 agent 回合收到带锁定逻辑和必需代码 diff 的范围化上下文包。它以 Codex 为主机。

**为什么它该赢。** 产品本身真实且有测试：24 个测试加一个 MCP 冒烟测试通过。claude-mem 调用实时可用；我们对照运行中的 worker 验证过。每个回合开始时，它用包名、框架、用户请求和未决锁定语句构建查询，访问 `/api/search`，把结果作为"Director Note"注入上下文包，显示在预览旁。doctor 命令报告 claude-mem 可用性，connect 命令把 worker URL 转发到 Codex MCP 环境。README 坦率："可选、失败开放、一条咨询性 Director Note。"

**为什么它该输。** Director Note 是原始搜索 markdown 的前 600 个字符，包含竖线、emoji 和表头，因为代码截断而非压缩。`source_refs` 数组始终为空，因为代码查找 worker 响应中不存在的 `results` 键。没有传项目过滤，所以无关项目的观察泄漏进 note；我们亲眼看到。根目录名为 `claude-mem-adapter.js` 的文件是 product-git 自己 API 的遗留浏览器 shim，尽管名字如此，从不触及 claude-mem。根目录还附带一份旧 PRD，描述基于 Hook 的捕获和演示观察，代码中不存在。主机 agent 是 Codex，所以使用 product-git 的 agent 不是 claude-mem 记录其工作的那个。

**排名与原因。** 第六名。集成深度 2，类别契合 2，可运行 4，周一早上 2，诚实度 4。一个真实的 MCP 治理产品，其中 claude-mem 是一次无范围限定的搜索调用，其输出是侧面板中的截断表格。

---

## 7. Relay (thehimalayanleo/relay) — 25 分中的 13 分

**它做什么。** 一种延续协议，把进行中的调查从一对人加 agent 移交给另一对。它把目标、决策、被拒方案和下一步行动的规范 JSON 胶囊封存在 SHA-256 摘要和过期能力 URL 之后，附加可选工作 pod，为 Codex、Claude Code、Cursor 和 OpenCode 渲染特定 harness 的恢复提示，并记录接受回执。Greptile 和 Sail 是主要集成。

**为什么它该赢。** 移交协议