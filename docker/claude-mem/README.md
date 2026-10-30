# claude-mem Docker harness

一个最小化容器，用于端到端测试 claude-mem，且不污染宿主机。这不是开发环境——只够启动 `claude`，加载本地构建的插件，并把观测数据捕获到一个可事后检查的一次性 SQLite 数据库中。

## 文件

| 文件 | 用途 |
|------|---------|
| `Dockerfile` | 镜像定义（node:20 + Bun + uv + Claude Code CLI + 本地 `plugin/`） |
| `build.sh` | 先运行 `npm run build`，再运行 `docker build`。标签默认为 `claude-mem:basic`。 |
| `entrypoint.sh` | 在容器内运行。若挂载了 OAuth 凭据，则将其写入 `$HOME/.claude/`，然后 `exec "$@"`。 |
| `run.sh` | 宿主机侧启动器。提取凭据（Keychain → 文件 → 环境变量），挂载持久化数据目录，进入交互式 shell。 |

## 快速开始

```bash
# 从仓库根目录运行：
docker/claude-mem/build.sh
docker/claude-mem/run.sh
```

`run.sh` 会进入容器内的 `bash`，`claude` 已在 `PATH` 中，插件预置于 `/opt/claude-mem`。启动方式：

```bash
claude --plugin-dir /opt/claude-mem
```

退出后，SQLite 数据库保留在宿主机的 `./.docker-claude-mem-data/claude-mem.db`——检查方式：

```bash
sqlite3 .docker-claude-mem-data/claude-mem.db 'select count(*) from observations'
```

## 镜像内容

布局参照 [anthropics/claude-code 的 devcontainer](https://github.com/anthropics/claude-code/blob/main/.devcontainer/Dockerfile)：
`FROM node:20`，非 root 的 `node` 用户，全局 `npm install -g @anthropic-ai/claude-code`。
省略了防火墙/zsh/fzf/delta/git-hist 工具，因为本镜像用于运行 claude-mem，而非编辑代码。

在此基础上：

- **Bun**（`/usr/local/bun`）—— claude-mem 的 worker 服务运行时
- **uv**（`/usr/local/bin/uv`）—— 按 `CLAUDE.md` 为 Chroma 提供 Python
- **`plugin/`** 复制到 `/opt/claude-mem` —— 本地构建的插件目录树
- **`/home/node/.claude`** 和 **`/home/node/.claude-mem`** —— 预创建的挂载点

层顺序是刻意安排的：插件文件在 `npm install` 层**之后**复制，这样迭代插件不会破坏 CLI 安装缓存。

## 固定版本

所有关键项都是 `--build-arg`——为可复现性固定版本，省略则用最新版：

```bash
docker build \
  -f docker/claude-mem/Dockerfile \
  --build-arg BUN_VERSION=1.3.12 \
  --build-arg UV_VERSION=0.11.7 \
  --build-arg CLAUDE_CODE_VERSION=1.2.3 \
  -t claude-mem:basic .
```

| 参数 | 默认值 | 说明 |
|-----|---------|-------|
| `BUN_VERSION` | `1.3.12` | 通过官方 `bun.sh/install` 脚本安装，标签 `bun-v${BUN_VERSION}`。 |
| `UV_VERSION` | `0.11.7` | 通过带版本的 `astral.sh/uv/${UV_VERSION}/install.sh` 安装。 |
| `CLAUDE_CODE_VERSION` | `latest` | npm 标签或精确版本。CI 中固定，本地可浮动。 |

## 认证

`run.sh` 按以下顺序选用第一个可用的认证源：

1. **`ANTHROPIC_API_KEY`** 环境变量 —— 直接挂载进容器。
2. **macOS Keychain** —— `security find-generic-password -s 'Claude Code-credentials'`。
3. **`~/.claude/.credentials.json`** —— 旧版磁盘形式，部分较旧的 CLI 安装和迁移过的机器上仍存在。

若使用凭据文件，会写入 `mktemp` 文件并 `chmod 600`，以只读方式挂载到 `/auth/.credentials.json`，容器 entrypoint 在 exec 前将其复制到 `$HOME/.claude/.credentials.json`。`run.sh` 返回时 `EXIT` trap 会删除临时文件——`docker run` 刻意**不**用 `exec`，以便 trap 有机会触发。

若未找到任何认证源，`run.sh` 会报错退出，提示使用 `claude login` 或 `ANTHROPIC_API_KEY`。

## 手动调用（不使用 `run.sh`）

```bash
docker run --rm -it \
  -v $(mktemp -d):/home/node/.claude-mem \
  -e CLAUDE_MEM_CREDENTIALS_FILE=/auth/.credentials.json \
  -v /path/to/creds.json:/auth/.credentials.json:ro \
  claude-mem:basic
```

或使用 API key 认证：

```bash
docker run --rm -it \
  -v $(mktemp -d):/home/node/.claude-mem \
  -e ANTHROPIC_API_KEY \
  claude-mem:basic
```

## 环境变量

| 变量 | 位置 | 用途 |
|-----|-------|---------|
| `TAG` | `build.sh`、`run.sh` | 覆盖镜像标签（默认 `claude-mem:basic`）。 |
| `HOST_MEM_DIR` | `run.sh` | 覆盖持久化 `.claude-mem` 卷的宿主机路径（默认 `$REPO_ROOT/.docker-claude-mem-data`）。 |
| `ANTHROPIC_API_KEY` | `run.sh`、entrypoint | API key 认证。跳过 OAuth 凭据提取。 |
| `CLAUDE_MEM_CREDENTIALS_FILE` | entrypoint | 挂载的 OAuth 凭据 JSON 路径（容器内）。启动时复制到 `$HOME/.claude/.credentials.json`。 |

## 透传参数

`run.sh` 之后的任何内容都会作为命令转发给容器：

```bash
docker/claude-mem/run.sh claude --plugin-dir /opt/claude-mem --print "what did we learn yesterday?"
```

## 清理

```bash
rm -rf .docker-claude-mem-data   # 清除持久化 DB + Chroma 存储
docker rmi claude-mem:basic       # 删除镜像
```