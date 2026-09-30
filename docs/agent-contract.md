---
title: 编码 agent 工作契约
type: contract
status: accepted
date: 2026-09-30
related:
  - decisions/0003-branch-and-release-conventions.md
  - decisions/0001-fork-dsh-and-absorb-at-seams.md
  - decisions/0002-out-of-scope-sources.md
---

# 编码 agent 工作契约

本文件是给本仓库所有编码 agent 会话的硬契约入口。只引用各决策的操作面，不新增决定；细节以链接为准。fork 之后根 `AGENTS.md` 尾部的「boom-harness 本地约定」小节指向这里，上游正文不改写。

## 动工前

- 先读 [docs/README.md](README.md) 的行为队列；每条按同名 spec → design → plan 的顺序执行。
- 新行为优先放进 `packages/boom/`，落在 dsh 已有 seam 上。bootstrap 阶段不改既有包名；只有 seam 对插件封闭时才改上游文件，且必须由对应 design 点名并能单独 revert。工具恢复与后续 US-7 点名的 `agent-loop.resume` 属于此范围（[决策 0001](decisions/0001-fork-dsh-and-absorb-at-seams.md)、[工具恢复 design](design/2026-09-30-tool-effect-recovery.md)、[后续 seam design](design/2026-09-30-follow-on-seams.md#已调度的模型重试)）。
- 参考本地 `/Users/boom/workspace/` 中的 harness 相关源码。行为队列已选定的部分按对应 spec → design → plan 在 dsh seam 上重写；其余参照与排除范围按 [决策 0002](decisions/0002-out-of-scope-sources.md)，不搬整份实现或第二套运行时。要推翻已有决定，先新开 decision，再动代码。

## 分支与发布

- `main`：可发布基线，只收完成版本验收的 `dev-*` 合并、验证通过的上游合并与 `release-*` 回灌修复；进行中的开发留在 `dev-*`。
- `dev-<major>.<minor>.<patch>`：面向目标版本的开发线。第一位大版本、第二位小版本、第三位修复版本，数字可为多位（如 `dev-1.10.23`）。发布后修复进 `dev-x.x.(n+1)`，新功能进下一小版本线。
- `release-<major.minor>.x`：惰性启用，从 `v*` tag 切出，只收 cherry-pick 修复并回灌 `main`。
- 本地 `master`：最新已 fetch 的 `upstream/master` 纯镜像，永不在其上提交。上游默认分支改名时，镜像名、跟踪目标及相关文档一起更新。
- 发布点打 `v*` tag。完整约定见 [决策 0003](decisions/0003-branch-and-release-conventions.md)。

## 提交与推送

- 提交信息中文、结构化（变更内容 + 为什么）；一次提交一个原子单元，不夹带无关文件。
- 未经要求不 `git push`；覆盖推送只用 `git push --force-with-lease`。
- 改提交描述用 `git commit --amend`（未推送时）。

## 上游合并

- 切到 `main`，fetch 后同步镜像，再合并 `upstream/master`；验证通过后把 `main` 合入当前 `dev-x.x.x`。不得把进行中的开发提前合入 `main`。命令顺序见 [plan 00 的 UPSTREAM.md 约定](plan/2026-09-30-00-fork-bootstrap.md#upstreammd-的合并与冲突记录)。
- 冲突保留清单见同一约定：fork 专属路径和条目保留；已实现的核心与适配器 hunk 按对应 design 合并，包括工具恢复、审批词汇与后续 US-7 的 `resume`，不整文件覆盖上游。
- 上游的 `README.md`、`AGENTS.md` 正文、`CLAUDE.md` 一律不改写。

## 文档归属

- 已选定的做法进 [docs/decisions/](decisions/README.md)，编号单调递增不复用；spec/design/plan 只引用，不重复展开决定。
- 改动若超出 design 点名范围，先补 decision，再写代码（[plan 06](plan/2026-09-30-06-follow-on.md) 的门）。
