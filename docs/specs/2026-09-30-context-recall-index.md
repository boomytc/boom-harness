---
title: 压缩后的序号回放目录
type: spec
status: draft
date: 2026-09-30
related:
  - design/2026-09-30-context-recall-index.md
---

# 压缩后的序号回放目录

> 摘要继续代替窗口里的旧回合。压缩完成之后，模型能看见一张按会话序号展开的目录，并按序号取回原文。还没压缩时不注入空目录。

## 动机

`dsh-compaction` 把被选中的历史换成一条摘要 user 消息。被遮蔽的事件留在日志里，回放能重建同一份压缩后的对话（`packages/compaction/compaction/README.zh.md`）。模型默认只看见摘要。`session_search` 不搜当前会话。同一可选包里的 `session_event_search` / `session_event_read` 仍能读当前会话里更早的事件，但不提供被遮蔽区间的目录。

## 现状

- 成功的压缩在日志中是 `compaction/start`、`compaction/summary`、一条表层摘要 `user/message`、`compaction/end`。
- `deriveMessages()` 渲染摘要和保留下来的近期节点。
- 前缀缓存依赖稳定的系统提示词。压缩替换会使被遮蔽历史的缓存失效；目录若写进冻结的系统前缀，会在每次压缩后再失效一次。

## 用户故事

### US-1：目录跟着压缩走（P1）

作为模型，我希望压缩之后仍知道旧回合的序号范围，以便知道该向哪一段要原文。

**验收场景**：

- Given 一次成功压缩遮蔽了序号 A 到 B，When 组装下一次请求，Then 模型可见上下文包含一条目录，其中有 A–B 的条目，且被遮蔽原文不整段出现。
- Given 尚未压缩，When 组装请求，Then 不注入空目录。

### US-2：按序号取回原文（P1）

**验收场景**：

- Given 目录中有一段序号，When 模型调用回放工具并给出这段闭区间，Then 返回该区间内的表层 `user/message`、`assistant/message` 与 `tool/result` 投影。单次最多 32 KiB，超出则给出下一偏移。不返回 `system/message` 与 `developer/message`。不另写一份历史库。
- Given 区间超出日志或落在未来序号，When 调用工具，Then 返回可理解的错误，不抛未捕获异常。

### US-3：目录不改写冻结前缀（P1）

**验收场景**：

- Given 同一会话连续两轮且目录内容没变，When 比较两次请求的系统前缀，Then 前缀字节一致。
- Given 目录因新的压缩而变化，When 组装请求，Then 变化出现在运行时上下文快照里，不改系统提示词字节，也不重写首个系统节点。

## 功能需求

- FR-001：目录条目 MUST 包含起止 `SessionSeq`、一行标题、以及来源（回合或既有摘要）。
- FR-002：注入文本 MUST 有字节上限。超出时 MUST 折叠更早的条目并保留序号，MUST NOT 静默丢弃「还有更早历史」这一事实。
- FR-003：回放工具 MUST 只读当前会话日志，MUST 按序号闭区间返回表层 `user/message`、`assistant/message`、`tool/result` 的文本投影。单次 MUST 不超过 32 KiB；超出时 MUST 给出下一偏移。MUST NOT 返回 `system/message` 或 `developer/message`。
- FR-004：回放结果 MUST 是普通工具结果，MUST NOT 把取回的原文写回为新的未压缩历史。
- FR-005：摘要 user 消息 MUST 继续存在。目录不是摘要的替代。

## 成功标准

- SC-001：压缩集成测试里，下一次请求的运行时上下文快照含有被遮蔽区间的序号。`deriveMessages` 对摘要 user 消息的渲染不变。
- SC-002：回放工具的结果文本等于该区间表层事件的投影，且会话日志不因此增加第二条用户历史。

## 非目标

- QwenPaw 的 `history.db`、视觉压缩、`recall_history_python`。
- 关掉摘要，或把压缩改成 pi 的会话树。
- 跨会话搜索。那仍是 `session_search` 的范围。
