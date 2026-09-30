---
title: 前五条之后的 seam
type: spec
status: draft
date: 2026-09-30
related:
  - design/2026-09-30-follow-on-seams.md
  - decisions/0002-out-of-scope-sources.md
---

# 前五条之后的 seam

> 这些行为已经决定要做，但每一条都挂在现有服务旁边，且不排在恢复、规则、目录、记忆、选项映射之前。前六条来自对照结论。US-7 从工具副作用恢复拆出，那条规格的 `resume` 仍用今天的 closer，不续跑 `llm/retry`。US-8 从权限规则拆出，那条规格只拒绝技能名。

## 动机

前五条解决「崩溃后副作用不可信」「授权不能记住」「压缩后看不见原文」「没有个人知识槽位」「网关 JSON 形状不统一」。下面八条依赖前面的契约，或会单独改一种恢复或注入契约。

## 用户故事

### US-1：子代理 worktree（P2）

作为用户，我希望显式要求隔离的子代理在单独的 git worktree 里改文件，并用一次显式合并写回，以便并行修改不落在同一 checkout。

**验收场景**：

- Given 子代理未要求隔离，When 它写文件，Then 路径在父工作区。
- Given 子代理要求 `worktree`，When 它写文件，Then 写入的是新 worktree；父工作区在显式 apply 之前不变。
- Given apply，When 合并冲突，Then 父工作区保持 apply 前的内容，并返回冲突说明。

### US-2：hunk 归属（P2）

**验收场景**：

- Given 一轮里既有代理编辑又有用户在同一文件上的后续编辑，When 查询该轮改动，Then 记录能区分代理写入、用户改在代理碰过的文件上、以及无关脏文件。
- Given Host 重启，When 打开同一会话，Then 归属仍在，不依赖进程内存。

### US-3：工具结果的模型副本（P2）

**验收场景**：

- Given `PostToolUse` 返回替换后的模型可见文本，When 组装下一请求，Then 模型看见替换文本。
- Given 同上，When 读取会话日志里的原始 `tool/result`，Then 仍是工具返回的原文。

### US-4：沙箱补强（P2）

**验收场景**：

- Given 策略拒绝读取某路径，When 调用通过改名或移动绕过，Then 调用失败。
- Given 策略要求 confine 且没有可用 runner，When 执行，Then 沿用现有 `SANDBOX_UNAVAILABLE` 拒绝。今天 Windows 上 `enforcement: 'partial'` 仍会执行并在结果里标明 partial；本故事不把 partial 改写成新的拒绝。
- Given 沙箱内的命令，When 目标是 `$DSH_HOME` 下的 memory、权限规则存储或钩子配置，Then 写入被拒绝。
- Given 需要区分网络，When 子进程访问网络与模型 API 访问网络，Then 两者的允许集合分开配置。模型 API 走宿主 fetch，不写成 `SandboxPolicy` 字段。

### US-5：宿主检查的停止门（P2）

**验收场景**：

- Given 任务清单文件列出未通过的检查，When 模型宣称完成，Then 循环不结束。宿主在 `agent/turn-stopping` 上 `steer` 下一拍。这条消息的 `source.kind` 不是 `user`，并且写入会话日志。
- Given 清单全部通过或用户中止，When 步骤边界到达，Then 循环可以结束。

### US-6：只约束 boom 新包的架构棘轮（P2）

**验收场景**：

- Given 上游包已有的依赖，When 运行棘轮，Then 不因这些既有依赖失败。
- Given `packages/boom` 里新增一条指向 `apps/web` 内部文件的深导入，When 运行棘轮，Then 失败。

### US-7：已调度的模型重试接着跑（P2）

**验收场景**：

- Given 打开的轮次里有 `llm/retry` 且没有相同重试 id 的 `llm/retry-started`，When `resume`，Then 不追加这段 closer 里的 `step/end` 与 `turn/end`。按日志里的 `delayMs` 重新等待，或在延迟已过时追加原来的 `llm/retry-started` 并重跑该步骤。
- Given 已有匹配的 `llm/retry-started`，When `resume`，Then 按工具恢复规格结算工具，不再插入一次新的重试调度。

### US-8：技能正文扫描（P2）

**验收场景**：

- Given 技能正文与某条已提交夹具全文相同，When 该技能即将注入，Then 不注入，并记录拒绝。包含片段或另做规则匹配不算命中。
- Given 技能正文不在夹具的危险集里，When 注入，Then 与没有扫描器时相同。
- Given 夹具语料还没有误报门槛，When 实现本故事，Then 不把它提前塞进权限规格的技能名拒绝。

## 功能需求

- FR-001：US-1 MUST 使用普通 `git worktree`。默认 MUST 为共享工作区。
- FR-002：US-2 MUST 挂在 `workspace-changes` 的摘要旁边，MUST NOT 换成另一套回合记录器。归属 MUST 写入会话日志。
- FR-003：US-3 MUST 在 `tools/post-execute` 上增加一条不写入 `tool/result` 的模型投影。现有 `PostToolDecision.content` 会进入随后提交的结果，不能拿来当这份额外副本。`hooks-claude-code` 目前不支持 `updatedToolOutput`；被记录后未应用的是 `updatedInput`。本规格内才把 `updatedToolOutput` 接到模型投影。日志里的 `tool/result` 保持原文。
- FR-004：文件 confine 仍是 `SandboxPolicy` 的按次 confine。没有可用 runner 时沿用 `SANDBOX_UNAVAILABLE`。MUST NOT 在宿主进程启动时套用 Landlock 或 Seatbelt。子进程网络允许列表与模型 API 允许列表分开，且都不是 `SandboxPolicy` 的字段。拒绝写入的是 `$DSH_HOME` 下的 memory、权限规则存储和钩子配置。
- FR-005：US-5 的清单 MUST 由宿主读取。完成与否 MUST NOT 只采纳模型的自述。监听 `agent/turn-stopping`，再 `steer`。MUST NOT 改循环内部。清单路径来自会跨轮继续的 goal 或会话配置，不来自 ralph。续跑消息的 `source.kind` MUST NOT 是 `user`，并且 MUST 写入会话日志。
- FR-006：棘轮脚本只遍历 `packages/boom/**`。只对这里的新增违规失败。不维护一份上游例外清单。
- FR-007：US-7 MUST 复用已写下的重试 id、轮次、步骤和 `delayMs`，MUST NOT 新开一轮。未配对重试时 MUST NOT 追加 `interruptedTurnClosers` 里的 `step/end` 与 `turn/end`。
- FR-008：US-8 的判定 MUST 只使用随仓库提交的夹具，且 MUST 与某条夹具全文相同才拒绝。没有全文命中的正文 MUST 放行。扫描器 MUST NOT 搬 QwenPaw 的 `ShellEvasion` 规则库。`renderSkillContent` 的两个调用方都要在渲染前经过扫描。

## 成功标准

- SC-001：八条各自有一个失败时能指出本故事的集成测试，且测试不依赖 grok、ZCode 或 QwenPaw 的源码。

## 非目标

- CoW 分片、btrfs、overlay、Grove、NFS、`x.ai/git/worktree/*`。
- 把整份 Agent 上传到 SSH 主机。
- QwenPaw 的 `prd.json` / Mission 模式，或 Coding 模式的系统提示词。
- 把 400 行文件上限当成产品能力。
