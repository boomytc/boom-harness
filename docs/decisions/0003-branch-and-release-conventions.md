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

fork 之后一棵树里同时存在三种流动：上游 `upstream/master` 的合并、行为队列的逐条实现（[plan 06](../plan/2026-09-30-06-follow-on.md) 要求每条 seam 单独开分支）、发布物 `@deepseek-ai/dsh-boom-bundle` 跟随上游版本火车的版本号。若分支命名和合流方向不先定死，会出现「哪个分支是真源」「上游合并落进哪条线」的争议。另外 plan 00 会带来一条本地 `master`，它和产品分支 `main` 并存，必须说清各自地位。

## Decision

分支命名与生命周期：

| 分支 | 生命周期 | 进入 | 离开 |
|---|---|---|---|
| `main` | 永久 | 只收 `dev-x.x.x` 合并、上游合并、`release-*` 回灌的修复 | — |
| `dev-<major>.<minor>.<patch>` | 一个版本线的开发期 | 从 `main` 切出 | 该版本发布、转入修复线后 |
| `release-<major.minor>.x` | 惰性启用，发布线存活期 | 从 `main` 上的 `v*` tag 切出 | 发布线废弃后删除 |
| 本地 `master` | 永久镜像 | 只随 `upstream/master` 移动 | 永不在其上提交 |
| tag `v*` | 永久 | 发布点打在 `main`（或对应 release 分支） | — |

- `dev-` 是**面向目标版本的开发线**前缀，命名 `dev-<major>.<minor>.<patch>`：第一位大版本、第二位小版本、第三位修复版本，数字可为多位（如 `dev-1.10.23`）。`dev-0.2.0` 面向 0.2.0；该版本发布后，修复进 `dev-0.2.1`，新功能进 `dev-0.3.0`。同一时刻只维护当前活跃的版本线。
- `release-<major.minor>.x` **在第一个真实发布线需要与 `main` 分叉时才创建**，不提前养分支。命名取 `major.minor` 不取 patch，一条线收同一 minor 的修复。
- 发布点先在 `main`（或 release 分支）打 `v*` tag，`release-*` 从 tag 对应的提交切出。发布线只收 cherry-pick 修复，修复必须同时回灌 `main`。
- 上游合并落点：`git fetch upstream && git merge upstream/master` **只进 `main`**（实现中的 `dev-x.x.x` 先合并进 `main` 再做上游合并）。冲突保留清单按 [UPSTREAM.md](../plan/2026-09-30-00-fork-bootstrap.md) 的约定：`packages/boom/**`、`docs/**`、`PROFILE_TEMPLATES.boom`，以及 `apps/cli` 里对 `@deepseek-ai/dsh-boom-bundle` 的 `workspace:*` 依赖，以及根 `AGENTS.md` 尾部的「boom-harness 本地约定」段。
- 本地 `master` 永远等于 `upstream/master`，只作上游镜像与合并基点，任何私人提交不得落在上面。

## Consequences

- 「当前真源」是活跃的 `dev-x.x.x` 开发线；`main` 只收版本发布点、上游合并与回灌修复，保持可发布。
- 长期分支只有 `main`、`master` 与当前活跃的 `dev-x.x.x`；旧版本线发布后冻结或删除。
- 若日后上游把默认分支从 `master` 改名，只需改 UPSTREAM.md 的合并命令，本约定不受影响。
- plan 00 写 UPSTREAM.md 时必须引用本决策；plan 06 各条落在面向其目标版本的 `dev-x.x.x` 线上。
- 本决策对 agent 会话的生效入口是 [agent-contract.md](../agent-contract.md)，由根 `AGENTS.md` 尾部追加段指向；上游 `AGENTS.md` 正文不改写。
