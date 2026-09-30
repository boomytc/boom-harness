---
title: 分支与发布线约定（dev- / release- 前缀）
type: decision
status: accepted
date: 2026-09-30
related:
  - decisions/0001-fork-dsh-and-absorb-at-seams.md
  - plan/2026-09-30-00-fork-bootstrap.md
  - plan/2026-09-30-06-follow-on.md
---

# 0003. 分支与发布线约定（dev- / release- 前缀）

## Status

accepted

## Context

fork 之后一棵树里同时存在三种流动：上游 `upstream/master` 的合并、行为队列的逐条实现（[plan 06](../plan/2026-09-30-06-follow-on.md) 要求每条 seam 落在面向其目标版本的 `dev-*` 上，保持独立改动）、发布物 `@deepseek-ai/dsh-boom-bundle` 跟随上游版本火车的版本号。若分支命名和合流方向不先定死，会出现「哪个分支是真源」「上游合并落进哪条线」的争议。另外 plan 00 会带来一条本地 `master`，它和产品分支 `main` 并存，必须说清各自地位。

## Decision

分支命名与生命周期：

| 分支 | 生命周期 | 进入 | 离开 |
|---|---|---|---|
| `main` | 永久 | 只收完成版本验收的 `dev-x.x.x` 合并、验证通过的上游合并、`release-*` 回灌的修复 | — |
| `dev-<major>.<minor>.<patch>` | 一个版本线的开发期 | 从 `main` 切出 | 该版本发布、转入修复线后 |
| `release-<major.minor>.x` | 惰性启用，发布线存活期 | 从 `main` 上的 `v*` tag 切出 | 发布线废弃后删除 |
| 本地 `master` | 永久镜像 | 只随 `upstream/master` 移动 | 永不在其上提交 |
| tag `v*` | 永久 | 发布点打在 `main`（或对应 release 分支） | — |

- `dev-` 是**面向目标版本的开发线**前缀，命名 `dev-<major>.<minor>.<patch>`：第一位大版本、第二位小版本、第三位修复版本，数字可为多位（如 `dev-1.10.23`）。`dev-0.2.0` 面向 0.2.0；该版本发布后，修复进 `dev-0.2.1`，新功能进 `dev-0.3.0`。同一时刻只维护当前活跃的版本线。
- `release-<major.minor>.x` **在第一个真实发布线需要与 `main` 分叉时才创建**，不提前养分支。命名取 `major.minor` 不取 patch，一条线收同一 minor 的修复。
- 发布点先在 `main`（或 release 分支）打 `v*` tag，`release-*` 从 tag 对应的提交切出。发布线只收 cherry-pick 修复，修复必须同时回灌 `main`。
- 进行中的 `dev-x.x.x` 不合入 `main`。目标版本完成并通过验收后，才把该开发线合入 `main`，形成版本发布点；需要发布时再打 tag。
- 上游合并落点：先切到 `main`，fetch 后同步本地镜像，再 `git merge upstream/master`。合并验证通过后，把 `main` 合入当前活跃的 `dev-x.x.x`，开发继续留在该线。不得为接上游而把未完成的开发提前合入 `main`。
- 冲突保留清单按 [plan 00 中的 UPSTREAM.md 约定](../plan/2026-09-30-00-fork-bootstrap.md#upstreammd-的合并与冲突记录)：保留 fork 专属路径和条目，并按对应 design 保留已实现的上游文件 hunk，包括工具恢复、审批词汇，以及后续 US-7 的 `resume`。只合并这些 hunk，不整文件覆盖上游改动。
- 本地 `master` 始终跟踪最新已 fetch 的 `upstream/master`，只作上游镜像与合并基点，任何私人提交不得落在上面。plan 00 的 `639ed01539` 固定的是产品 `main` 的起点，不把已前移的镜像倒退到分叉点。

## Consequences

- 当前开发成果在活跃的 `dev-x.x.x`；可发布基线在 `main`。`main` 的进入条件只有表中的三类，合并与回灌后都须通过验收。
- 常驻分支是 `main` 与上游镜像 `master`；开发期保留当前活跃的 `dev-x.x.x`，发布线需要分叉时另保留 `release-*` 修复线。旧开发线发布后冻结或删除，发布线废弃后删除。
- 若日后上游把默认分支从 `master` 改名，本地镜像分支与其跟踪目标也随之改名，并同步修订本决策、agent 契约和 UPSTREAM.md 的名称与合并命令。产品 `main`、`dev-*`、`release-*` 的合流方向不变。
- plan 00 写 UPSTREAM.md 时必须引用本决策；plan 06 各条落在面向其目标版本的 `dev-x.x.x` 线上。
- 本决策对 agent 会话的生效入口是 [agent-contract.md](../agent-contract.md)，由根 `AGENTS.md` 尾部追加段指向；上游 `AGENTS.md` 正文不改写。
