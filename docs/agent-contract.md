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
- 新行为只进 `packages/boom/`，只落在 dsh 已有 seam 上，不改包名和 `agent-loop` 核心（[决策 0001](decisions/0001-fork-dsh-and-absorb-at-seams.md)）。
- 参照仓库（pi、grok-build、ZCode、QwenPaw）只作参照，不搬实现（[决策 0002](decisions/0002-out-of-scope-sources.md)）。要推翻已有决定，先新开 decision，再动代码。

## 分支与发布

- `dev-<major>.<minor>.<patch>`：面向目标版本的开发线。第一位大版本、第二位小版本、第三位修复版本，数字可为多位（如 `dev-1.10.23`）。发布后修复进 `dev-x.x.(n+1)`，新功能进下一小版本线。
- `release-<major.minor>.x`：惰性启用，从 `v*` tag 切出，只收 cherry-pick 修复并回灌 `main`。
- 本地 `master`：`upstream/master` 纯镜像，永不在其上提交。
- 发布点打 `v*` tag。完整约定见 [决策 0003](decisions/0003-branch-and-release-conventions.md)。

## 提交与推送

- 提交信息中文、结构化（变更内容 + 为什么）；一次提交一个原子单元，不夹带无关文件。
- 未经要求不 `git push`；覆盖推送只用 `git push --force-with-lease`。
- 改提交描述用 `git commit --amend`（未推送时）。

## 上游合并

- `git fetch upstream && git merge upstream/master` 只进 `main`；实现中的 `dev-x.x.x` 先合并回 `main` 再做上游合并。
- 冲突保留清单（[UPSTREAM.md](plan/2026-09-30-00-fork-bootstrap.md) 约定）：`packages/boom/**`、`docs/**`、`PROFILE_TEMPLATES.boom`、`apps/cli` 里对 `@deepseek-ai/dsh-boom-bundle` 的 `workspace:*` 依赖，以及根 `AGENTS.md` 尾部的「boom-harness 本地约定」段。
- 上游的 `README.md`、`AGENTS.md` 正文、`CLAUDE.md` 一律不改写。

## 文档归属

- 已选定的做法进 [docs/decisions/](decisions/README.md)，编号单调递增不复用；spec/design/plan 只引用，不重复展开决定。
- 改动若超出 design 点名范围，先补 decision，再写代码（[plan 06](plan/2026-09-30-06-follow-on.md) 的门）。
