---
title: 实现 Markdown 记忆
type: plan
status: draft
date: 2026-09-30
related:
  - specs/2026-09-30-markdown-memory.md
  - design/2026-09-30-markdown-memory.md
---

# 实现 Markdown 记忆

> 先占住 `boom.memory-index` 段并追加收件箱。整理任务作为同一计划的后半，不阻塞前半验收。

## 目标

- 空记忆时提示词有固定空索引句，没有具体事实。
- 两个 worktree 共用一份工作区记忆。
- 整理失败时主题文件不变。

## 非目标

- 嵌入检索、把记忆提交进用户仓库、通用写 `$DSH_HOME` 的工具。

## 背景与依据

[../design/2026-09-30-markdown-memory.md](../design/2026-09-30-markdown-memory.md)。

## 任务分解

- [ ] 新增 `packages/boom/memory-files`，创建 `$DSH_HOME/memory` 布局。
- [ ] 用 `git rev-parse --git-common-dir` 计算 workspace key；失败时用工作区根哈希。
- [ ] 注册 `boom.memory-index` 段，`order` 为 `-100`，`interpolate: false`。两份 INDEX 都没有可列条目时使用规格里的固定句子。已有收件箱文件时列出相对路径。上限 4096，超出带截断标记。
- [ ] 这段排在以后的 boom 人设段之前。人设文件不得包含具体记忆事实。
- [ ] 回合结束和中止时把已登记观察追加到当天 inbox。无观察则不改 INDEX。
- [ ] `memory_read` 只打开记忆根内的 `.md`。解析含符号链接后的真实路径必须仍在该根内。
- [ ] 测试：同一 common dir 的两个目录得到同一 key；无关目录得到另一 key。
- [ ] 测试：只有 inbox 变化时 topics 的哈希不变。
- [ ] 后半：用 `dsh-jobs` 跑整理的模型调用。租约和快照是 `memory-files` 自己的文件。失败删除租约和快照且不改 topics。成功时用一次临时目录改名换上 topics、INDEX 和已消费 inbox。
- [ ] 测试：模型调用抛错后 topics 字节与作业前一致。

## 风险与依赖

- 前置：bootstrap。索引段应早于任何人设文案提交。
- 测试不要把「shell 仍可能写记忆目录」说成已关闭。这个缺口留到 follow-on 的沙箱补强。
