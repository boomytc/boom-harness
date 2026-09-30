---
title: 工具副作用按重放策略恢复
type: spec
status: draft
date: 2026-09-30
related:
  - design/2026-09-30-tool-effect-recovery.md
  - decisions/0001-fork-dsh-and-absorb-at-seams.md
---

# 工具副作用按重放策略恢复

> 进程死在工具执行中时，已经做出的副作用不重跑，只读调用才允许用已记下的参数再执行一次。

## 动机

dsh 会在执行前写下 `tool/call`。崩溃恢复却把未完成的调用收成合成错误，让模型自己判断要不要重试（`packages/session/session-persistence/README.zh.md`：「合成 closer 是唯一崩溃方案」）。`TOOL_OUTCOME_UNKNOWN` 的文案要求模型验证副作用，但已完成的输出不在日志里，模型看不见进度，也可能把写操作再跑一遍。

## 现状

- 代码：`TOOL_NOT_STARTED` 表示请求没有持久的开始记录；`TOOL_OUTCOME_UNKNOWN` 表示有开始记录但没有结果（`packages/core/session/src/repair.ts`）。
- `resume` 把 closer 当作普通批次追加，然后才 `setupAndPublish`。没有「接着这一轮」的路径。
- 工具定义有 `isConcurrencySafe`，没有重放策略（`packages/core/tools/src/index.ts` 的 `ToolDefinition`）。
- 并行调用的完成顺序与写入会话的源顺序不是同一件事；恢复不区分二者。

## 用户故事

### US-1：有副作用的调用不重跑（P1）

作为用户，我希望崩溃前已经开始的写操作不要在恢复时自动再执行，以便磁盘和外部系统不被做两次。

**独立验证**：一个声明 `replay: "never"` 的工具在 `tool/call` 之后、`tool/result` 之前杀掉进程。

**验收场景**：

- Given 日志里有 `tool/call` 且没有已暂存结局，When `resume`，Then 追加一条合成 `tool/result`，错误码仍是 `TOOL_OUTCOME_UNKNOWN`，且该工具的 `execute` 不再被调用。
- Given 同上，且日志里有该 `callId` 的进度记录，When `resume`，Then 合成结果包含最后一条进度的文本，并写明执行已中断。
- Given 日志里已有该 `callId` 的暂存结局，但还没有表层 `tool/result`，When `resume`，Then 表层结果来自暂存结局，`execute` 不再被调用。

### US-2：只读调用可以重放一次（P1）

作为用户，我希望纯读取在崩溃后能补上结果，以便恢复不必假装读过。

**独立验证**：声明 `replay: "safe"` 的工具，参数已写在 `tool/call` 里。

**验收场景**：

- Given 没有暂存结局，When `resume`，Then 用已持久化的参数调用一次 `execute`，成功或失败都写成普通 `tool/result`。
- Given 同一次 `resume` 正在重放，When 观察这个 `callId`，Then 这次 `resume` 只调用一次 `execute`。下一次 `resume` 仍只看有没有暂存结局或表层结果：有则不执行，无则再执行一次。

### US-3：未开始的调用保持可重试（P1）

**验收场景**：

- Given 助手消息里有工具请求，但没有对应 `tool/call`，When `resume`，Then 结果码为 `TOOL_NOT_STARTED`，文案仍表示可以在需要时重试，且不调用 `execute`。

## 功能需求

- FR-001：每个工具 MUST 有重放策略 `safe` 或 `never`。未声明时 MUST 为 `never`。
- FR-002：`replay` MUST NOT 出现在模型可见的工具 schema 里。
- FR-003：内建工具 `read`、`grep`、`glob`、`web_search`、`web_fetch` MUST 声明 `safe`。`bash`、`write`、`edit`、`str_replace_editor`、`read_image` MUST 保持 `never`。`read_image` 会调用 `attachments.saveImage`。没有点名的工具保持缺省 `never`。恢复看日志里的 `tool/call.replay`，不回读当前注册表；缺这个字段按 `never`。
- FR-004：进度记录 MUST 只追加、有字节上限，且 MUST NOT 进入 `deriveMessages()` 的表层。
- FR-005：暂存结局 MUST 在表层 `tool/result` 之前可持久化，并带 `callId`。恢复 MUST 先读暂存结局，再决定重放或合成。
- FR-006：表层 `tool/result` 的顺序 MUST 仍按助手消息里的工具请求顺序。`tool/outcome-staged` MUST 在工具函数规范化返回时写入，因此完成更早的调用可以先有暂存结局。暂存载荷是这一刻的 `content`、`isError`、`error`，还没有经过 `tools/post-execute`。
- FR-007：`safe` 重放 MUST 使用 `tool/call` 里已经持久化的参数，MUST NOT 让模型改写参数后再执行。
- FR-008：读工具上的 `safe` 声明在核心，因此 `web` 与 `boom` 使用同一套结算。未声明的工具在两个 profile 上都是 `never`。

## 成功标准

- SC-001：US-1 的杀进程测试中，工具副作用计数器在 `resume` 后不增加。
- SC-002：US-2 的杀进程测试中，`resume` 后日志里有且仅有一条该 `callId` 的成功或失败 `tool/result`。
- SC-003：`replay` 不出现在 `schemas()` 投影中（与 `isConcurrencySafe` 同一类测试）。

## 非目标

- pi 的 entry tree、lane、13 叶状态机、`pi.pending.*` 地址、mutation line。
- 把助手流改成写前日志。未结算的模型文本仍然整段丢弃。
- 子代理「已接受但未写入子日志」的提示去重。
- 自动重放 `bash`。即使命令看起来像 `git status`，默认仍是 `never`。
- 续跑未配对的 `llm/retry`。该行为在 follow-on，避免这一条同时改两种恢复契约。
