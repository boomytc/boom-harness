---
title: 工具副作用恢复的日志与 resume
type: design
status: draft
date: 2026-09-30
related:
  - specs/2026-09-30-tool-effect-recovery.md
---

# 工具副作用恢复的日志与 resume

> 在现有 `tool/call` 与合成 closer 之间插入「策略 + 进度 + 暂存结局」。只有 `resume` 会按这些记录重放。

## 背景与范围

规格见 [../specs/2026-09-30-tool-effect-recovery.md](../specs/2026-09-30-tool-effect-recovery.md)。改动集中在工具定义、会话事件和 `agent-loop` 的 `resume`。不新增存储引擎。

## 目标与非目标

- 目标：崩溃后的结算只依赖会话日志里已经写下的事件。
- 非目标：用内存检查点或第二份 WAL 证明工具已经跑完。

## 设计

### 工具声明

`ToolDefinition` 增加可选字面量 `replay?: 'safe' | 'never'`，缺省 `never`，不进入 `schemas()`。写入 `tool/call` 之前解析：只有精确的 `'safe'` 才重放；缺省、其它值和抛错都是 `never`。这和 `executionMode` 只把精确 `true` 当成并行是同一条规则。`replay` 不是按参数调用的函数。

`tool/call` 的持久载荷增加 `replay`。恢复只读这个载荷，不回读当前工具注册表。旧日志没有该字段时按 `never`。重放用与 `parseArguments` 相同的规则得到参数：空串是 `{}`，非法 JSON 保留原字符串。`appendToolCall` 存的是未解析字符串。

内建 `read`、`grep`、`glob`、`web_search`、`web_fetch` 在各自 `defineTool` 处写 `'safe'`。`bash`、`write`、`edit`、`str_replace_editor`、`read_image` 不写。

### 非表层事件

两份新事件都进核心 `SessionEventMap`，重跑 persistence catalog，不标 `ignorable`。未知且非 `ignorable` 的类型会让整份日志被拒绝。新增普通事件类型不升 `SESSION_FORMAT_VERSION`。`deriveMessages()` 忽略没有 `surfaceOp` 的事件。

- `tool/progress`：`callId`、单调的进度序号、截断后的文本。单条上限 8 KiB。日志只追加。恢复只读该 `callId` 的最后一条。发出进度的回调可以放在 `packages/boom/tool-replay`，只挂在 `boom` 模板上。没有这条事件时，合成结果里没有进度正文。
- `tool/outcome-staged`：`callId`、`replay`，以及表层要用的 `content`、`isError`、`error`。核心在 `normalizeDispatchResult` 返回时写入。这一刻还没进 `commitReady` 的 `finalize` / `tools/post-execute`，也还没 `appendToolResult`。写入发生在 `tool/call` 已经追加之后，不在 `session.append` 的监听器里。

两个 profile 都走核心的暂存写入。boom 包只留进度回调。

### resume

只在 `packages/core/agent-loop/src/index.ts` 的 `resume` 里重放。`ToolCallRecovery.results()` 保持纯函数，现场失败和 fork seed 不调用 `execute`。

对未配对的 `tool/call`：

- 已有 `tool/outcome-staged`：用暂存的 `content`、`isError`、`error` 追加表层 `tool/result`，不调用 `execute`，也不补跑 `post-execute`。
- `replay === 'safe'` 且没有暂存结局：对已有 `callId` 调用工具体 `execute` 一次，不走 `tool-calls.ts` 里的 `startCall`（那会再写一条 `tool/call`）。成功或失败都先暂存再写成表层结果。
- 否则：沿用今天的 `TOOL_OUTCOME_UNKNOWN` / `TOOL_NOT_STARTED` 文案。有进度时把最后一条附在文案后面。

先把本次 `tool/result` 追加进日志，再对追加后的日志调用 `interruptedTurnClosers`。待重放只是这次 `resume` 的内存标记。表层结果仍按助手消息中的请求顺序追加。未配对的 `llm/retry` 仍走今天的 closer，不在这里继续退避。没有这些新事件的旧日志仍由今天的 closer 收口。

## 备选方案

- 搬 pi 的操作状态机：会话将不再是一条 `SessionEvent` 日志。与决策 0002 冲突。
- 崩溃后一律重放：`bash` 与写文件会重复副作用。
- 只改文案、不记进度和暂存：模型仍然看不见已经发生的输出。规格的 SC-001 无法满足。
- 把暂存推迟到 `appendToolResult` 之前：完成更早、源序更后的 `bash` 已经执行，崩溃时却还没有结局，恢复只能合成未知。不选。

## 横切面

- 前缀缓存：新事件非表层，不进入消息投影，不改写已发送前缀。
- 合并：核心 diff 是 `replay` 字段、`tool/outcome-staged` 的写入点，以及 `resume` 在 closer 之前的结算。进度回调留在 boom 包。
- 安全：`safe` 必须由工具作者声明。参数来自日志，不来自恢复时的模型输出。
