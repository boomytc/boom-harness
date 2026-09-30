---
title: 实现压缩回放目录
type: plan
status: draft
date: 2026-09-30
related:
  - specs/2026-09-30-context-recall-index.md
  - design/2026-09-30-context-recall-index.md
---

# 实现压缩回放目录

> 压缩成功后从已有 `compaction/summary` 组装序号目录，运行时上下文展示它，`recall_session` 只读当前日志。

## 目标

- 压缩后的下一次请求能看见被遮蔽区间的序号。
- 回放不增加第二条用户历史，也不创建数据库。

## 非目标

- 修改摘要算法或 `deriveMessages()` 对摘要消息的渲染。
- 新增会话事件类型。

## 背景与依据

[../design/2026-09-30-context-recall-index.md](../design/2026-09-30-context-recall-index.md)。

## 任务分解

- [ ] 新增 `packages/boom/recall-index`。
- [ ] 从无 `error` 的 `compaction/end` 找到配对的 `compaction/summary`，用 `shadowedSeqs` 的最小值和最大值作 `seqFrom` / `seqTo`。不读 `compaction/end` 的载荷当区间，不在 `session/event` 里再 `append`。
- [ ] 用 `ctx.systemPrompt.context` 注册 `boom.recall-index`，`order` 为 200。空文本不输出。上限 4096 字节，超出时折叠最早区间。标题在写入快照前去掉 `{{` 与 `}}`。
- [ ] 注册 `recall_session`，只读当前会话。投影表层 `user/message`、`assistant/message`、`tool/result`。单次 32 KiB，超出时返回下一偏移。
- [ ] 测试：压缩夹具的下一次运行时上下文含起止序号，且被遮蔽原文不在目录文本里。`deriveMessages` 对摘要消息的渲染不变。
- [ ] 测试：两次目录相同时系统前缀字节相同；目录变化时不重写首个系统节点。
- [ ] 测试：工具结果等于该区间的表层投影，日志中的用户消息条数不因此增加。
- [ ] `web` profile 不挂本包时，现有 compaction 测试通过。

## 风险与依赖

- 前置：bootstrap。
- 运行时上下文会插值。标题里留下 `{{` 会让每步组装失败。
