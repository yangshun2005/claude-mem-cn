# 更新日志

本项目的所有重要变更都会记录在此文件中。

格式基于 [Keep a Changelog](https://keepachangelog.com/en/1.0.0/)。

## [15.7.0] - 2026-10-30

## My Work：刷新停留在你的 agent 上，以及一个 Feedback 按钮（#5190）

当 My Work 的自动刷新从 Claude Code 得到 claude-mem 拒绝的答复时（最常见的是超过 1,200 字符的评审），整个刷新会转到下一个已登录的 agent。在安装了 Codex 的机器上，这意味着在所有者 ChatGPT 账户上隐式运行 codex exec --ephemeral，即使设置了 CLAUDE_MEM_PROVIDER=claude（#5189：四天内约 230 次运行）。

- 被拒绝的答复现在会在给出它的 agent 上结束该次刷新。下一次计划运行会重试同一个 agent，并沿用现有的失败退避。完全无法运行的 agent（未登录、崩溃）仍会将运行交给下一个。
- My Work 头部新增 Feedback 按钮，位于 Star 旁边。它会打开一个简短的"My Work feedback" Issue 表单。

## Note-taker 始终使用最新的 Haiku（#5195）

note-taker 的默认模型现在是 Claude Code 的 haiku 别名，始终解析为最新的 Haiku，而不是固定的 Haiku 4.5 ID。仍带有出厂固定 ID 的已保存设置会在升级时迁移到该别名。你自己输入的模型保持不变。仓库的 CLAUDE.md 现在将此列为策略：绝不将带日期的模型 ID 固定为默认值。

## My Work 与 worker

- My Work 的活动驱动后台重建已合并，因此打开的 My Work 页面不再在每次数据库提交时重建（#5133，缓解 #5124）。
- Work View 在构建前等待数据库完成启动（#5157）。
- 查看器的处理状态在流中断后重新同步（#5139）。
- 早于 CLAUDE_MEM_LOG_RETENTION_DAYS（默认 30，0 表示永久保留）的每日 worker 日志会被清理。今天和昨天的文件始终保留（#5186）。

## 项目

- 父 Git 仓库内的命名环境可以声明显式继承，因此不相关的父级记忆不再泄漏到其上下文中（#5022）。
- Oh My Pi 将瞬态项目记忆保留在模型上下文中（#5183）。
- Claude Duet：捕获与召回（#5188）。

## 媒体（默认关闭）

- 图像感知观察的第一阶段：有界媒体契约，以及带大小、像素和截止时间限制的惰性 Sharp 0.35.5 转换器。CLAUDE_MEM_MEDIA_CAPTURE_ENABLED 和 CLAUDE_MEM_MEDIA_INFERENCE_ENABLED 均默认为 false（#4341）。

## 安全

- 可执行文件发现从不查找当前目录（#5148）。
- 安装程序 URL 使用 https，Bun 从所有者控制的目录运行（#5149）。
- worker 拒绝跨源浏览器对会话端点和 POST /api/memory/ingest 的写入（#5162、#5161）。
- Postgres API 密钥以加盐 scrypt 哈希存储（#5160），并遵循所有 Postgres sslmode（#5152）。
- 原子写入使用独占创建的随机临时名称（#5151）。
- 私有标签在截断前被剥离，Hermes 的最后一轮受门控（#5150）。
- 脱敏现在覆盖 GitLab、Hugging Face 和 npm 令牌（#5174）、URL 凭据中的密码（#5173），以及本地脱敏开启时的入站同步文本（#5171）。

## 修复

- Hook stdin 在完整时即被框定，无需等待 EOF（#5126）。
- 已关闭会话的 EPIPE 不再记录为致命错误（#5177）。
- Windows Codex 沙箱拒绝会得到解释（#5140）。
- OpenAI 兼容 provider 从带文本代码的 200 错误信封中读取 HTTP 状态（#5191）。
- SessionStart 在没有 TeamBrain 时也会将记忆使用块计入其预算（#5181）。
- CLAUDE_MEM_DATA_DIR 中的前导 ~ 会被展开（#5179）。
- 日志尾部只读取完整行（#5166）。
- 过滤后的 Chroma 回退收窄到 SQLite 行 ID（#5154）。

完整更新日志：https://github.com/thedotmack/claude-mem/compare/v15.6.0...v15.7.0

## [15.6.0] - 2026-10-29

## My Work：查看器状态的单一存储（#5187）

My Work 过去将状态保存在三个地方：手写的模块存储、`App` 中的组件状态，以及 React contexts。现在它运行在单个 Zustand 存储上，分为四个切片：work、view、rules 和 focus。

**你会注意到的变化**
- **重新加载和前进/后退可用。** 页面、焦点区域和打开的请求抽屉都在 URL 中（`?view=`、`?focus=`、`?request=`）。
- **你的列表视图、过滤和排序在访问之间被记住。** 主题仍跟随系统或 `?theme=`。如果浏览器存储被阻止，偏好仅在当前访问期间有效。
- **实时更新更稳定。** 已停止的源无法发布。较旧的读取绝不会覆盖较新的实时模型。当实时更新超过保存时，你会看到当前模型，且你的保存被保留。心跳只重新渲染状态指示器。
- **抽屉按 ID 跟随其请求**，并在请求离开信息流时关闭。如果首次读取失败，你会得到 Retry 按钮。选中的 Closed 过滤器即使没有匹配项也保持可见。
- **My Rules：** 一次保存一个，保存运行时控件被禁用。每个 Approve/Reject 回执都绑定到 worker 保存的版本，一旦规则此后发生变化，Undo 会被拒绝。
- **My Focus：** 所有屏幕共享一个保存历史。

**Worker**
- 规则保存返回 `savedRuleId`。
- "Load in Claude Code" 将变更发布到其他打开的标签页。
- 信息流签名包含 `rulesFile`。

**完整更新日志：** https://github.com/thedotmack/claude-mem/compare/v15.5.0...v15.6.0

## [15.5.0] - 2026-10-29

## 功能

- **My Work / My Focus：** My Work 和 My Focus 中的 Aceternity 组件（#5136）
- **Observer：** 可选的 OpenAI 兼容批处理（#5117）
- **Work view：** 摘要 provider 选择现在显式（#5104）
- **遥测：** 在 PostHog 中统计 My Work 查看器打开和页面切换（#5107）

## 修复

### Work view / My Work
- 通过部分预算将构建故事保持在限额内（#5128）
- 阻止失败的刷新淹没 agent 并无限重试（#5108）
- 在刷新修正中保留被拒绝的答复（#5113）
- 渲染记录的组标题和评审摘要（#5114）

### 脱敏
- 脱敏裸 `token`、`secret` 和 `api_key` 键下的值（#5130）
- 脱敏 Authorization 头对象的 bearer 令牌（#5129）
- 在结构化载荷中配对 `MYSQL_PWD` 和 `dbPassword` 键（#5111）
- 脱敏 Slack 应用级（`xapp-`）和刷新（`xoxe-`）令牌（#5119）

### Antigravity / MCP
- 以真实工具名和稳定 ID 记录 agy MCP 调用（#5170）
- 解析 Antigravity 项目上下文（#5105）
- 在 Windows 8.3 短路径查找中使用 `%I` 而非 `%~I`（#5099）

### Worker、安装和 agent
- 保护 `POST /api/logs/clear` 免受外部 localhost 源（#5123）
- 端口回收在 Linux 上从 `/proc` 读取端口所有者、命令行和名称（#5101）
- 安装或修复完成时清除 `last-install-error.json`（#5102）
- 当已安装的 claude-mem 没有备份时 `backup-setup` 停止（#5103）
- 通过解析后的可执行文件启动 Codex（#5112）
- Pi：为会话启动上下文请求提供 hooks 的 API 预算（#5118）

### 测试
- 对提交的插件包做语法检查，而非构建后的（#5137）

**完整更新日志：** https://github.com/thedotmack/claude-mem/compare/v15.4.0...v15.5.0

## [15.4.0] - 2026-10-28

## My Work：焦点区域，以及自行归档的请求

### 焦点区域是 My Work 的顶层（#5131）
- **MY FOCUS 栏。** 你的焦点区域按优先级顺序列出，每个都带有 needs-you 计数。每个请求恰好属于一个区域，或属于 **Other**。在手机上，区域显示为 chip 行。
- **区域头部。** 每个区域页面显示其目标语句、合并进度、你自己的引述、你尚未开始的请求，以及指向 Past focus 的链接。My Goals 页面已退役；其内容现在位于此处。
- **Suggested。** 新区域会自行出现在此处。你可以对每个执行 Keep、Rename、Merge 或 Dismiss，并支持 Undo。你的决定跨计算机同步，刷新绝不会覆盖它们。
- **Owner-goals 对其挖掘内容排序。** 它按顺序应用三项测试，因此每个语句恰好成为一个焦点区域、请求或规则。优先级和未完成请求不再相互重复。
- **Focus 步骤在自动刷新中取代 Align。** 它放置每个请求，并从 Other 或从包含多个独立请求的请求中提出新区域。
- **请求卡片。** 每张卡片显示其区域，请求详情有 **Move to…**。"What came out of it" 现在称为 **Deliverables**。

### 判定器将新对话归档到其请求下（#5110）
- **在任何分组 agent 运行之前，** 每个新对话都会被归档：代码链接（相同的工作状态列表或 `request_id`）立即加入。在 CMEM Pro 上，或使用你自己的 OpenRouter 密钥时，Jev 还会检查该对话是否延续你的某个未完成请求。检查分数达到 0.8 或以上时加入。在 0.5 到 0.8 之间时，卡片上会出现 **Join / Not this** 建议。否则它会开启自己的请求。
- **焦点区域也是如此。** 在相同的安装上，Jev 在 Focus 步骤运行前为每个请求建议一个焦点区域。目前该选择只是给 Focus agent 的提示：自动放置要等到有足够多你自己的操作来校准截止值。
- **没有 Jev 时，** 一切照旧。

### 请求跨项目根保留其对话（#5106）
- 决策中指明其项目根的请求会保留来自其他根的对话，后续刷新仍可向其添加。

### 修复
- sync-api 查询计划测试应用自己的 schema，因此不再因测试顺序而失败。

**CMEM Pro：** Jev 决策网关（`/api/inference/v1/decisions`）已为 Pro 订阅者上线。

## [15.3.0] - 2026-10-26

## 变更内容

### My Work 卡片上的 Deliverables（#5109）

让 agent 生成演示文稿、报告、HTML 页面或任何其他文件，在 My Work 中该请求卡片正面、**Deliverables** 下即可一键访问。

- **Stop hook 将本轮为你生成的内容复制**到 `~/.claude-mem/deliverables/<project>/`，按内容寻址（`report-3f9a1c2b.md`），因此副本比其生成所在的 worktree、临时文件夹或临时目录更持久。agent 通过 `work_state_write` 附加的文件（`work_product`、`deliverable`，或任何命名了供人打开文件的字段）都计入，在源树之外写入的演示文稿、文档、页面、图像、视频和电子表格也计入。源文件绝不会被推断复制。SubagentStop 从其自身转录捕获子 agent 生成的内容。
- **worker 记录并提供每个副本**，位于 `GET /api/deliverables/:id/:filename`，遵循与记录文件相同的 loopback、同源和沙箱规则；markdown 和文本内联打开。当副本在磁盘上且文件此后未被写入时，它代替文件生成位置的记录。
- **隐私和限制是明确的。** 提示完全为 `<private>` 的一轮不留下副本也不留下记录。对于另一主机上的 worker 不复制任何内容，因为它无法打开副本。每次 Stop 50 个文件，每文件 512 MiB，达到时报告，绝不静默。`CLAUDE_MEM_DELIVERABLES_ENABLED=false` 关闭复制。
- SessionStart 工作状态规则现在要求 agent 在完成前将你要求的内容写为文件并作为 `work_product` 附加。

### 其他新增

- **自动本地数据库快照**，带崩溃安全恢复：`npx claude-mem backup [run|status|list]` 和 `npx claude-mem restore`。默认关闭；`CLAUDE_MEM_BACKUP_ENABLED=true` 配合 `CLAUDE_MEM_BACKUP_INTERVAL_HOURS` 和 `CLAUDE_MEM_BACKUP_RETAIN_COUNT`。（#3751）
- **Notion 导出。** 每个新观察在存储后立即 upsert 到 Notion 数据库。通过 `CLAUDE_MEM_NOTION_ENABLED`、令牌和数据库 ID 选择加入；绝不阻塞捕获。（#4799）
- **Codex 子 agent 以记忆时间线启动。** 新的 `SubagentStart` hook 为全新子 agent 提供与 SessionStart 给会话相同的上下文；继承父级历史的 fork 被跳过。Codex 会请求一次信任新 hook。（#4760）
- **Runcrate 预设**，用于 openai-compatible provider。（#5067）

### 修复

- **Antigravity CLI：** claude-mem 不再注册 `PreToolUse`，它曾自动批准每个 agy 工具调用并将每个调用排队两次（#5085）。agy 提示现在每轮记录一次，因此 agy 工作会出现在 My Work 中，在 agy 中读取的文件获得其观察历史（#5092）。
- **脱敏：** 临时 AWS 凭据被完整脱敏：ASIA 访问密钥 ID、会话令牌，以及 `aws sts` JSON 输出中的 SecretAccessKey（#5066、#5073、#5080）。
- **转录：** watcher 检查点保持到缓冲工作被存储（#5072）；转录监视配置路径遵循 `CLAUDE_MEM_DATA_DIR`（#5070）。
- **Observer 解析器：** 读取模型最终确定的类型，绝不读取其周围的标记（#5077）。
- **Windows：** 当 `bun.exe` 位于非 ASCII 路径时，启动器通过 8.3 短路径编译（#5089）。
- **SessionStart** 终端预览为其页脚留出空间（#5084）；安装帮助说明哪些 provider 需要 cmem.ai 登录（#5086）。
- **发布工具：** 每个发布的插件清单携带相同版本，由构建同步并在 CI 中检查（#5083、#5082、#5075）。

**完整更新日志**：https://github.com/thedotmack/claude-mem/compare/v15.2.1...v15.3.0

## [15.2.1] - 2026-10-25

## 变更内容

- **npm 发布现在使用可信发布（OIDC）。** "Publish to npm" GitHub Actions 工作流使用为本仓库工作流签发的短期 OIDC 令牌向 npm 认证，而非长期 `NPM_TOKEN` 密钥（自 v13.26.1 起每个标签都失败）。可信发布还携带 npm 来源证明，将包链接到构建它的提交和工作流。（#5063）

插件或 worker 无运行时行为变更；此版本仅为通过新发布路径发布。

**完整更新日志**：https://github.com/thedotmack/claude-mem/compare/v15.2.0...v15.2.1

## [15.2.0] - 2026-10-25

## 亮点

### 浏览器中的安装漏斗（#5031）
全新安装现在打开浏览器设置流程，在设置完成前找到你最近的工作并将其转化为 Work、Goals 和 Rules。现有安装照旧打开 My Work。

1. **安装** — `npx claude-mem install` 安装检测到的 hooks，启动 worker 并打开浏览器。
2. **扫描** — 读取最近 30 天支持的本地 agent 历史。配置的排除项目和 agent 编写的提示被排除，存储的提示去重，私有标签清理和配置的脱敏在显示或提取任何内容前运行。
3. **揭示** — 将过滤后的所有者提示（gzip）上传到 CMEM Cloud 进行托管意图提取。匿名安装获得私有、持久的安装凭据，支持重试和登录复用。返回的目标、规则和请求组填充本地 My Work 快照。上传和提取错误会显示并可重试。预勾选的产品改进框仅更改 `improveProduct`。
4. **存储选择** — At Home 或 On The Go。On The Go 使用现有的配对、登录和结账流程；交付的记忆凭据在结账标记为已付费前激活，仅同步账户保留其 provider。
5. **完成设置** — 安装任何其他选定的本地工具，并可导入现有记忆。设置变更重启 worker 时，进行中的提取或导入被保留。
6. **下一步** — 搜索已保存的记忆和提示，将结果保留为笔记，然后交给 My Work。终端成功退出。

持续目标和进度刷新使用已安装、已登录的 agent：Claude Code 仅获得有界文件工具和记忆召回；Codex 以严格只读的工作区/运行时