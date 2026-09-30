---
title: 参照仓库中不进入 boom-harness 的部分
type: decision
status: accepted
date: 2026-09-30
related:
  - decisions/0001-fork-dsh-and-absorb-at-seams.md
---

# 0002. 参照仓库中不进入 boom-harness 的部分

## Status

accepted

## Context

四路对照各自给出了 absorb / reference-only / skip。若只保留「看起来先进」的清单，后续实现会把第二套运行时带进来。dsh 已经有技能、压缩、headless、定时任务、ACP、多代理、会话日志和文件沙箱。重复实现这些，会和上游同一 seam 永久冲突。

## Decision

我们将只实现 [docs/README.md](../README.md) 行为队列里的规格。下列部分保持参照或不做。

保持参照，出现具体缺口再单独立项：

- pi 的会话树、bound values/lists、操作状态机、Chord、助手流写前日志、codemode、ExtensionAPI。会话仍然是一条线性 `SessionEvent` 日志，组合运行时仍然是 Cordis。工具恢复规格只增加重放策略和暂存结局；没有这些记录的旧日志仍用今天的合成 closer。
- pi 的分支摘要、cache 保温、虚拟模型、子代理提交去重。它们有价值，未排期。已调度模型重试的续跑放在 follow-on，不并进工具恢复。
- grok 的技能、钩子总线、headless、长任务调度、压缩算法、代码图、外部会话目录、提示队列合并。
- ZCode 的执行世界身份、双客户端投递、端口/适配器分层、界面密度。远程整包上传、Electron 组件库、自有账号和网关改写不做。
- QwenPaw 的人设文件套件、频道连接器、插件市场、多工作区管理器、定时任务、文件工作区、本地小模型、Hub。交互上的「主界面只留状态和下一步」只作为以后写 UI 文案时的约束。

明确不做：

- 搬运 `pi-ai`。dsh 已经依赖并 patch `@earendil-works/pi-ai@0.87.1`。
- 搬运 grok 的 leader/Unix socket 进程模型、登录、加密 prompt、TUI、CoW/btrfs/Grove worktree。
- 把 QwenPaw 的 ReMe、嵌入、`history.db` 或视觉压缩加进压缩路径。
- 把 ZCode 的动态工作流 journal 当作前五条。`packages/workflow/workflow/README.zh.md` 写的是「没有日志化或恢复」。仓库里没有 `resumeFromRunId`。要推翻这句必须新开决策。

## Consequences

- 前五条做完，boom 仍不是天枢、grok、ZCode 或 QwenPaw 的功能超集。
- follow-on 里还有八条：worktree、hunk 归属、工具结果双份、沙箱补强、宿主停止门、架构棘轮、已调度模型重试的续跑、技能正文扫描。技能名拒绝在权限规格里；正文扫描不在那一条里提前完成。
- LightAgent 的产品不替换这次 fork。`products/pilotdeck` 的产品根就是整份应用，dsh 没有可以外挂的内核。`products/eval-bench` 的离线打分和 `products/water-agent` 的一轮图都不做步骤边界上的停止门。`products/ops-harness` 的迭代催促与现有 `steer` 是同一条缝，预算耗尽后的强制交卷与停止门相反。`products/os-host` 的四态治理、Seatbelt 单中间件和 Scroll sqlite 不吸收。`products/memento` 用通用 Write 改主题、把长说明和检索正文注入提示词，这两条不吸收。写进现有设计的只有三处：持久放行只跳过询问，`confine` 仍在执行体内；停止门的下一拍用非 `user` 来源的 `steer` 并写入日志；`memory_read` 按解析后的真实路径收在记忆根内，且只打开 `.md`。
