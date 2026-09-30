---
title: 压缩回放目录的注入位置
type: design
status: draft
date: 2026-09-30
related:
  - specs/2026-09-30-context-recall-index.md
---

# 压缩回放目录的注入位置

> 从已经写在日志里的成功压缩组装序号目录，放进运行时上下文，用一个只读工具按 `SessionSeq` 读当前日志。

## 背景与范围

规格见 [../specs/2026-09-30-context-recall-index.md](../specs/2026-09-30-context-recall-index.md)。不改摘要算法，不改 `deriveMessages()` 对摘要 user 消息的渲染，不改 `packages/compaction`。

## 目标与非目标

- 目标：目录变化不重写冻结的系统前缀；原文只存在于会话日志。
- 非目标：第二份 SQLite 历史，或把取回的原文再插入为未压缩用户消息。

## 设计

`packages/boom/recall-index`：

1. 不新增 `compaction/recall-index`。`session/event` 回调里再 `append` 会因禁止重入被抛掉，异常被吞掉，压缩本身仍成功。组装时只读日志里已有的、配对成功的压缩：`compaction/end` 没有 `error`，区间取同一次 `compaction/summary` 的 `shadowedSeqs`。`compaction/end` 的载荷只有 `compactionId`、可选 `sourceCommandId`、`turn`、可选 `error`。`shadowedRange.start/end` 必须等于这份列表的首尾，它是表层位置跨度。目录条目用 `shadowedSeqs` 的最小值和最大值作为 `seqFrom` / `seqTo`。闭区间里可以夹着非表层序号；回放只投影表层事件。失败的 `end` 没有 summary，不产生条目。
2. 每条是 `{ seqFrom, seqTo, title, source }`。`source` 是 `turn` 或 `summary`。`source` 为 `summary` 时标题不复制摘要正文，起止序号仍是这次 `shadowedSeqs` 的最小和最大。标题取该区间第一条用户消息的首行，超长则截断。写入快照前去掉标题里的 `{{` 和 `}}`。运行时上下文没有 `interpolate: false`，未知的 `{{name}}` 会让组装抛错。
3. 用 `ctx.systemPrompt.context({ name: 'boom.recall-index', order: 200, text })` 注册。`getContextOrder` 只接受 `SANDBOX_POLICY`（110）、`APPROVAL_POLICY`（115）、`SUBAGENT_DELEGATION`（120）。200 排在这三项之后。没有成功压缩时返回空字符串，空字符串不进入快照。字节上限 4096；放不下时合并最早的相邻条目，并保留一行「更早序号 S–T 已折叠」。
4. 这段是运行时上下文，不是系统提示词段。文本变化后，现有循环会把快照追加为 `source.kind === 'runtime-context'` 的 user 消息。这不重写首个系统节点，也与路由是否声明 `systemPromptUpdate: 'in-history'` 无关。段文本不变时，前缀保持不动。
5. 注册工具 `recall_session`，参数为闭区间 `from`、`to`。实现读当前会话日志，投影该区间的表层 `user/message`、`assistant/message`、`tool/result`。不返回 `system/message` 和 `developer/message`。单次返回上限 32 KiB，超出时说明下一偏移。

`web` profile 不挂本插件，因此上游压缩行为不变。

## 备选方案

- 监听 `compaction/end` 再追加一条目录事件：重入被丢掉，未知类型还会让日志打不开。不选。
- 把目录写进摘要 user 消息末尾：每次压缩都改写那条表层消息的契约。不选。
- 扩展 `tool-session-query` 而不注入目录：模型不知道有哪些序号。规格 US-1 不成立。不选。
- 另建 `history.db`：与「日志是原文」重复。不选。

## 横切面

- 前缀缓存：只允许运行时上下文快照变化。禁止在压缩时重写 persona 或工具 schema。
- 合并：不改 `packages/compaction`，不改 `deriveMessages`。
- 隐私：回放工具只服务当前会话。不读取其它 session id。
