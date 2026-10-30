<!-- 镜像自 GitHub Issue #3982。Issue 是公开跟踪器；本文档是设计。
     若两者不一致，设计以本文档为准，状态以 Issue 为准（oh-my-issues 健康检查）。 -->

> **跟踪器：** https://github.com/thedotmack/claude-mem/issues/3982

## 缺陷
FTS5 unicode61 无法对 CJK 分词 → 即使存在匹配行，短语搜索也始终返回 0。Chroma 为空时回退到 FTS 也无法修复匹配。

## 子 Issue
- #3801
- #3859

## 相关 PR
- #3810
- #3173
- #3285
- #3430

## 修复顺序
按脚本检测启用 CJK 感知分词器 / trigram / LIKE 回退；worker SQLite 与服务端搜索采用同一契约。

## 测试矩阵
- CJK 查询 × worker SQLite FTS × 服务端搜索
- Chroma 为空时的回退路径

## 不在范围内
- Chroma 启动/固定（plan-22 #3610）
- 项目标识（plan-20）