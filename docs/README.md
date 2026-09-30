# 文档

本目录是 boom-harness 的实现方案。源码树尚未从 DeepSeek Harness fork 进来；这些文档先于 fork 存在，执行 [plan/2026-09-30-00-fork-bootstrap.md](plan/2026-09-30-00-fork-bootstrap.md) 时必须整目录保留。

文档用于准备参考并吸收 `/Users/boom/workspace/` 中 harness 相关仓库的实现，最终落地 boom-harness：`deepseek-harness` 是运行时底座，`pi`、`grok-build`、`ZCode`、`QwenPaw` 是行为源码参照；`Tianshu-harness` 的产品组织方式与 `LightAgent` 的局部行为参照，边界见决策 [0001](decisions/0001-fork-dsh-and-absorb-at-seams.md) 与 [0002](decisions/0002-out-of-scope-sources.md)。每条吸收先核对本地源码与 dsh seam，再按行为队列的 spec → design → plan 重写并验收。分支和编码契约服务于这条实施路径。

## 归类

| 目录 | 回答 | 不回答 |
|---|---|---|
| [decisions/](decisions/README.md) | 已经选定的做法，以及被排除的做法 | 任务勾选 |
| [specs/](specs/README.md) | 做什么、为什么、怎样算做完 | 文件级改法和被否决的结构 |
| [design/](design/README.md) | 落在哪个 seam、数据怎么流、为什么不选另一种结构 | 用户故事编号的重复展开 |
| [plan/](plan/README.md) | 下一步改哪些路径、按什么顺序、用什么命令验收 | 新的产品决定 |

给编码 agent 的硬契约入口是 [agent-contract.md](agent-contract.md)，只引用上表内容，不新增决定。

## 阅读顺序

1. [decisions/0001-fork-dsh-and-absorb-at-seams.md](decisions/0001-fork-dsh-and-absorb-at-seams.md)
2. [decisions/0002-out-of-scope-sources.md](decisions/0002-out-of-scope-sources.md)
3. [decisions/0003-branch-and-release-conventions.md](decisions/0003-branch-and-release-conventions.md)
4. [specs/2026-09-30-fork-baseline.md](specs/2026-09-30-fork-baseline.md) 与 [plan/2026-09-30-00-fork-bootstrap.md](plan/2026-09-30-00-fork-bootstrap.md)
5. 之后按 plan 编号 01 到 05，每条先读同名 spec，再读同名 design，再执行 plan

## 行为队列

| 顺序 | 规格 | 设计 | 计划 |
|---|---|---|---|
| 0 | [fork 基线](specs/2026-09-30-fork-baseline.md) | 见决策 0001 | [bootstrap](plan/2026-09-30-00-fork-bootstrap.md) |
| 1 | [工具副作用恢复](specs/2026-09-30-tool-effect-recovery.md) | [design](design/2026-09-30-tool-effect-recovery.md) | [plan](plan/2026-09-30-01-tool-effect-recovery.md) |
| 2 | [权限规则](specs/2026-09-30-permission-rules.md) | [design](design/2026-09-30-permission-rules.md) | [plan](plan/2026-09-30-02-permission-rules.md) |
| 3 | [压缩回放目录](specs/2026-09-30-context-recall-index.md) | [design](design/2026-09-30-context-recall-index.md) | [plan](plan/2026-09-30-03-context-recall-index.md) |
| 4 | [Markdown 记忆](specs/2026-09-30-markdown-memory.md) | [design](design/2026-09-30-markdown-memory.md) | [plan](plan/2026-09-30-04-markdown-memory.md) |
| 5 | [模型选项映射](specs/2026-09-30-model-option-map.md) | [design](design/2026-09-30-model-option-map.md) | [plan](plan/2026-09-30-05-model-option-map.md) |
| 其后 | [后续 seam](specs/2026-09-30-follow-on-seams.md) | [design](design/2026-09-30-follow-on-seams.md) | [plan](plan/2026-09-30-06-follow-on.md) |
