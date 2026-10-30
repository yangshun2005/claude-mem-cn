# 🪟 Windows 用户——我们认为已修复了最大的问题。帮忙测试一下？

## 🐛 之前哪里坏了

- 🧟 **残留进程。** claude-mem 会在后台启动一个小的搜索辅助程序。关闭时，Windows 只停止了*顶层*程序——下面的辅助进程像僵尸一样继续运行。
- 🔒 **端口卡死。** 这些僵尸进程占着 claude-mem 重新启动所需的端口。于是它无法重启，于是每个提示都被阻塞。有人**在一次会话中撞上了 834 次。**
- 💾 **磁盘被吃光。** 僵尸进程还会永久留下垃圾文件。一位用户测到 **144 GB**。😳
- 🤫 **搜索悄悄死掉。** 辅助程序可能抓到机器上*错误的* Python，然后……就不工作了。没有报错，没有警告。记忆搜索静默停止。

## 🔧 我们修了什么

- 👨‍👩‍👧‍👦 关闭时现在会结束**整个进程家族**，而不只是父进程。
- 🧹 不再在下载中途杀进程，垃圾文件不再堆积。
- 🐍 错误的 Python 再也混不进来。
- 🎯 确保绝不会误关错误的程序（这一项经过 **7 轮**审查才搞定 😅）。
- 🤖 **加入了真正的 Windows 测试。** 我们的测试以前在 Windows 上从未真正启动过搜索辅助程序——这正是这些 bug 一直漏过去的原因。现在每次变更都会跑。

## ✅ 在 Windows 上实际验证过

不是“应该能用”——这些在真实 Windows 机器上跑过并通过：

- ✅ 搜索辅助程序启动、保存一条记忆、再找回来
- ✅ 关闭后**零**残留进程
- ✅ 即使 Python 环境很乱也能正常工作

---

## 📦 想试试吗？（约 10 分钟）

**你需要：** Node.js 和 Git。☕ 拿杯咖啡，构建要几分钟。

打开 **PowerShell**，**逐条**粘贴：

```powershell
git clone https://github.com/thedotmack/claude-mem.git
cd claude-mem
git checkout windows-megafix
npm install
npm run build-and-sync
node dist\npx-cli\index.js doctor
```

（所有 Windows 修复都合到了一个分支——PR **#3661**。）

最后一条会打印健康检查。🩺

## 👀 “成功了”是什么样

- ✅ `doctor` 显示绿色/OK 行——尤其是 **Bun**、**uv** 和 **Worker daemon**
- ✅ Claude Code 正常启动，没有提示被阻塞
- ✅ 记忆搜索确实返回结果
- ✅ 停止 claude-mem，然后运行 `Get-Process uv,python -ErrorAction SilentlyContinue`——应该**什么都不剩**。一条命令验证整个修复。🎉

## 🆘 如果出问题

给我们留言，附上：

1. 📋 `node dist\npx-cli\index.js doctor` 的完整输出
2. 🪟 你的 Windows 版本
3. 📜 `%USERPROFILE%\.claude-mem\logs\` 里最新的日志文件

说真的——一份失败的反馈**和**成功的反馈**同样有用**。这正是测试的意义。🙏

## ⏪ 想退出？很简单。

```powershell
npx claude-mem@latest install
```

直接回到正常发布版本。无需撤销，无需清理。👍

---

### 📚 好奇的话，这里是相关 Pull Request

- **#3661** — 你刚安装的汇总分支（`windows-megafix`），合并了以下所有内容

- **#3644** — 最大的那个（残留进程、端口卡死、磁盘垃圾、搜索静默死亡）
- **#3647** — 代码搜索在 Windows 上静默返回空
- **#3648** — 设置里用了 `~\` 时数据被存到错误文件夹
- **#3649** — 缺少 Git Bash 时现在会给出*清晰提示*，而不是莫名其妙的崩溃
- **#3657** — 修复在 Windows 上从源码构建（之前需要一个仅限 Mac/Linux 的工具 🙃）

⚠️ 这些还没合并——你测试的是预览版。这正是我们现在需要的。💛