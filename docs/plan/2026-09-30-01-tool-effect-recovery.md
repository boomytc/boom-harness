---
title: 实现工具副作用恢复
type: plan
status: draft
date: 2026-09-30
related:
  - specs/2026-09-30-tool-effect-recovery.md
  - design/2026-09-30-tool-effect-recovery.md
---

# 实现工具副作用恢复

> `resume` 在关闭中断轮次之前，先按 `replay` 与暂存结局结算工具。

## 目标

- 规格 US-1 至 US-3 各有一个可重复的测试。
- 未挂 boom 进度插件、且工具保持默认 `never` 时，旧日志的 closer 文案仍然可被现有测试接受。

## 非目标

- 子代理去重、助手流 WAL、会话树。

## 背景与依据

[../design/2026-09-30-tool-effect-recovery.md](../design/2026-09-30-tool-effect-recovery.md)。

## 任务分解

- [ ] 在 `packages/core/tools/src/schema.ts` 与 `ToolDefinition` 增加 `replay`，缺省 `never`，并加入「不出现在 `schemas()`」的测试（对照 `isConcurrencySafe` 的测试）。
- [ ] 给 `read`、`grep`、`glob`、`web_search`、`web_fetch` 声明 `safe`。确认 `bash`、`write`、`edit`、`str_replace_editor`、`read_image` 的定义里没有 `safe`。
- [ ] 写入 `tool/call` 之前把 `replay` 收成精确的 `'safe'` 或 `never`。补会话格式测试：旧日志没有该字段时按 `never`。重放参数使用 `parseArguments` 的同一规则。
- [ ] 把 `tool/progress` 与 `tool/outcome-staged` 登记进核心 `SessionEventMap` 并重跑 persistence catalog，不标 `ignorable`。`tool/outcome-staged` 在 `normalizeDispatchResult` 返回时由核心写入，字段是 `content`、`isError`、`error`。`packages/boom/tool-replay` 只提供进度回调和 8 KiB 上限。
- [ ] `ToolCallRecovery.results()` 保持纯函数。`resume` 先追加本次 `tool/result`，再对追加后的日志调用 `interruptedTurnClosers`。有暂存结局则提升为表层结果且不调用 `execute`。`safe` 且无结局则对已有 `callId` 调用工具体 `execute` 一次，不走 `startCall`。其余保持 `TOOL_OUTCOME_UNKNOWN` / `TOOL_NOT_STARTED`。有进度时附上最后一条。
- [ ] 不在这里续跑 `llm/retry`。
- [ ] 测试：`never` 工具在 `tool/call` 后杀进程，`resume` 后 execute 计数不变。
- [ ] 测试：`safe` 工具同样杀进程，同一次 `resume` 恰好一次 execute，并有一条 `tool/result`。下一次 `resume` 在已有表层结果时不再执行。
- [ ] 测试：已有 `tool/outcome-staged` 时 execute 计数不变，`resume` 提升出的表层结果等于暂存的 `content`、`isError`、`error`。
- [ ] 测试：未配对的 `llm/retry` 仍按现有 closer 收口。续跑重试不在本计划。
- [ ] `pnpm` 下对该包的现有 session repair 与 agent-loop resume 测试全部通过。读工具被标成 `safe` 之后，更新那些假定「崩溃后一律不执行」的夹具。

## 风险与依赖

- 前置：bootstrap。
- 重放 `safe` 工具若不是只读，会重复副作用 → 缺省 `never`，并在 review 时核对每个 `safe` 声明。
- 与上游 `repair.ts` 冲突 → 把进度缓冲留在 boom 包，核心只保留策略与结算顺序。
