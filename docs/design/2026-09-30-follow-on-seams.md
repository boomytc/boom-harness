---
title: 后续 seam 的落点
type: design
status: draft
date: 2026-09-30
related:
  - specs/2026-09-30-follow-on-seams.md
---

# 后续 seam 的落点

> 每条都是现有服务上的附加层。默认行为与未挂载时相同。

## 背景与范围

规格见 [../specs/2026-09-30-follow-on-seams.md](../specs/2026-09-30-follow-on-seams.md)。本文件只固定落点，避免以后做成第二套运行时。实施顺序以 [../plan/2026-09-30-06-follow-on.md](../plan/2026-09-30-06-follow-on.md) 为准。

## 目标与非目标

- 目标：八条都能独立合并、独立关掉。
- 非目标：在这一篇里展开每一条的事件载荷。开工那一条时再补一篇单独 design。

## 设计

### 子代理 worktree

`ctx.subagents` 的隔离选项增加 `shared | worktree`，默认 `shared`。只在服务上加枚举不够：今天 spawn 继承父工作目录。`subagent-spawn-in-process`、`subagent-acp`、`subagent-dsh-sdk`、`subagent-codex`、`subagent-claude-code` 的 cwd 都来自父会话 `header.cwd`。`worktree` 要改的是这些提供方的 cwd 来源。它调用 `git worktree add`，目录在会话临时区，子代理的 cwd 指向该目录。返回父会话的是摘要，不是直接写父 checkout。`apply` 在父工作区执行 `git merge --no-ff`；冲突则 `merge --abort` 并返回说明。

### hunk 归属

继续用 `dsh-workspace-changes` 的回合快照。`tools/pre-execute` 只有路径。`workspace-changes` 的摘要在 Host 内存里，重启就没了。`user-on-agent-file` 要拿代理写完后的副本和轮次结束副本对比，分不出这两份就不要标这个值。列表写入 `workspace/attribution`。这个事件按仓库里的会话事件声明进 `SessionEventMap`，或在登记前标 `ignorable: true`。不要写到临时对象目录。`ctx.workspaceChanges` 的现有摘要仍只含路径和行数。

### 模型可见的工具副本

现有 `PostToolDecision.content` 会进入随后提交的结果，`appendToolResult` 把这份 `content` 写进 `tool/result`。模型和日志今天是同一份。模型副本是另一条投影，不写入 `tool/result`。现有 `content` / `value` 仍是日志结果。审计和回放读原始结果。`hooks-claude-code` 目前不支持 `updatedToolOutput`。被记录后未应用的是 `updatedInput`。本条才把 `updatedToolOutput` 接到那条不入日志的投影。

### 沙箱补强

文件 confine 仍是 `SandboxPolicy` 的按次调用。没有可用 runner 时已经是 `SANDBOX_UNAVAILABLE`，不另造失败码。Windows 的 `enforcement: 'partial'` 今天仍会执行；本条不把它改成新的拒绝。补上：拒绝包含改名后的目标；拒绝写入 `$DSH_HOME` 下的 memory、权限规则存储和钩子配置。子进程网络允许列表放在 confine 旁边，不放进 `SandboxPolicy`。模型 API 走宿主 fetch，允许列表在那条路径上，也不写成 `SandboxPolicy` 字段。不在 Web/Desktop 宿主进程启动时启用 Landlock 或 Seatbelt。

### 停止门

清单是会话 cwd 下的一个 Markdown 或 JSON 文件。路径由会跨轮继续的 goal 或会话配置给出。ralph 在 worker 报告完成时就返回，不走父会话的 `agent/turn-stopping`，不挂这条路径。宿主监听 `agent/turn-stopping`，读自己算出的未通过条目，再 `agent.steer`。不改循环内部。`steer` 的消息 `source.kind` 不是 `user`，压缩标题和「用户原文」投影跳过这个 source。事件写入会话日志，崩溃后续跑仍能看见这次催促。不另写一条 `source.kind === 'user'` 的消息。用户中止与清单清空都可以结束。

### 架构棘轮

脚本只遍历 `packages/boom/**`。检查公开入口、禁止深导入 `apps/` 与其它包的 `src/`、禁止 boom 包依赖 `apps/web` 的内部模块。只对这里的新增违规失败。不把行数上限当作规则。

### 已调度的模型重试

改 `agent-loop` 的 `resume`。等待是进程内的 `cancellableDelay`，崩溃后续跑时定时器已经没了，日志里有 `delayMs`。`interruptedTurnClosers` 会同时补上未完成工具结果、`step/end` 和 `turn/end`。未配对的 `llm/retry` 先按工具恢复规格结算工具，然后不追加这段 closer 里的 `step/end` 与 `turn/end`。按日志里的 `delayMs` 重新等待；延迟已过则追加原来的 `llm/retry-started` 并重跑该步骤。这一条会改变所有 profile 的恢复，所以不放进工具恢复那一次核心 diff。

### 技能正文扫描

`renderSkillContent` 把正文同时送进 `skill` 工具结果和 `/名字` 预注入。扫描放在这两个调用方之前，或放进该函数本身。与某条已提交夹具全文相同才跳过注入，并写非表层记录。其余正文放行。不搬 QwenPaw 的规则库。

## 备选方案

- 用 grok 的 fast-worktree 实现隔离：依赖私有扩展和特定文件系统。不选。
- 用归属替换 `workspace-changes`：Web 改动卡片已经读现有摘要。不选。
- 把停止门做成新的 agent loop：`goal` 与 `ralph` 已经占据「何时继续」。不选。
- 续跑文本不进日志：崩溃后只能靠门再算一遍。不选。

## 横切面

- 合并：worktree 改 `subagent` 的隔离选项与提供方 cwd；停止门只监听 `agent/turn-stopping` 再 `steer`，不改循环内部；模型重试续跑（US-7）才改 `agent-loop.resume`。工具结果双份碰到 `hooks-claude-code` 和 `appendToolResult` 旁边的投影。沙箱补强扩展文件 confine，网络允许列表不进 `SandboxPolicy`。hunk、技能扫描和棘轮可以停在 boom 包或脚本里。
- 安全：沙箱补强是权限规则和记忆文件的实际边界。它排在后面，是因为前面的规格用 API 限制了正常工具；全开沙箱的 shell 仍要等这一条。
