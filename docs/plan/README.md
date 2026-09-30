# plan

执行顺序。勾选表示树已经到达那一步，不表示规格被改写。

| 顺序 | 计划 | 前置 |
|---|---|---|
| 0 | [fork bootstrap](2026-09-30-00-fork-bootstrap.md) | 无 |
| 1 | [工具副作用恢复](2026-09-30-01-tool-effect-recovery.md) | 0 |
| 2 | [权限规则](2026-09-30-02-permission-rules.md) | 0 |
| 3 | [压缩回放目录](2026-09-30-03-context-recall-index.md) | 0 |
| 4 | [Markdown 记忆](2026-09-30-04-markdown-memory.md) | 0；索引段名在写人设之前占用 |
| 5 | [模型选项映射](2026-09-30-05-model-option-map.md) | 0 |
| 6 | [后续 seam](2026-09-30-06-follow-on.md) | 1–5 中它点名依赖的那几条 |

1 到 5 都只依赖 bootstrap，可以在人够的时候并行。默认仍按编号做：第 1 条改变崩溃语义，后面的测试都要在新语义上写。
