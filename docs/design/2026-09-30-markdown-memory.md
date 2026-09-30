---
title: Markdown 记忆的目录与索引段
type: design
status: draft
date: 2026-09-30
related:
  - specs/2026-09-30-markdown-memory.md
---

# Markdown 记忆的目录与索引段

> 记忆是 `$DSH_HOME/memory` 下的 Markdown。系统提示只注入索引。整理任务是第二步，失败时收件箱保持原样。

## 背景与范围

规格见 [../specs/2026-09-30-markdown-memory.md](../specs/2026-09-30-markdown-memory.md)。第一刀交付空索引、收件箱追加和按仓库分目录。整理任务（US-4）在同一包的第二个提交，不阻塞前三个故事。

## 目标与非目标

- 目标：人可以用编辑器打开这些文件；代理只能经插件 API 追加观察，经只读工具阅读正文。
- 非目标：用 embedding 检索，或把记忆放进 git 工作区。

## 设计

### 布局

```text
$DSH_HOME/memory/global/INDEX.md
$DSH_HOME/memory/global/inbox/YYYY-MM-DD.md
$DSH_HOME/memory/global/topics/<slug>.md
$DSH_HOME/memory/workspaces/<key>/INDEX.md
$DSH_HOME/memory/workspaces/<key>/inbox/YYYY-MM-DD.md
$DSH_HOME/memory/workspaces/<key>/topics/<slug>.md
```

`<key>`：`git rev-parse --git-common-dir` 的规范化路径的 sha256 前 16 个十六进制字符。命令失败时用工作区根的同一哈希。实现放在 `packages/boom/memory-files`，通过 `ctx.subprocess` 调用 git，不链接 libgit。

### 提示词

`ctx.systemPrompt.section` 名称为 `boom.memory-index`，`order` 为 `-100`，`interpolate: false`。`-100` 小于部署人设前缀的 `0`，也不占用 `getSectionOrder` 的第一方名字。以后的 boom 人设段必须用更大的 order。内容来自两份 `INDEX.md`。两份都没有可列条目时渲染固定句子：「记忆索引为空。事实写入记忆收件箱，不写入这段提示。」已有收件箱文件时，索引列出其相对路径，仍不改 topics。字节上限 4096，超出则截断并加「索引已截断」。

人设、产品说明不得写入具体记忆。fork 基线规格的非目标已把个人知识挡在系统提示之外。

### 写入与阅读

回合结束（含中止）时，本插件把该轮登记的观察追加到当天 inbox。登记 API 是进程内的 `ctx` 服务，只有 boom 包和以后的显式工具可以调用。不提供通用「写 DSH_HOME」工具。

阅读工具 `memory_read` 只打开 `.md`。相对路径解析（含符号链接）后必须仍在全局或该工作区记忆根内。只拒绝字符串 `..` 不够。模型按索引里的路径打开正文。

### 整理

US-4 用 `dsh-jobs` 跑这次模型调用，不新造调度器。jobs 是进程内登记，没有租约或收件箱快照。租约、快照归 `memory-files`：开始时把 inbox 复制为快照并写租约文件（含过期时间）。模型调用使用快照，不持有写入锁。成功时先写临时目录，再用一次目录改名换上 topics、INDEX 和已消费 inbox。失败或租约过期则删除租约和快照，不改 topics，收件箱保持领取前。同一工作区同时只允许一个租约。

## 备选方案

- 放进 `ctx.storageDomain`：人不能直接改长文。不选。
- 放进仓库的 `MEMORY.md`：会被提交，也会被代理的 `write` 改掉。不选。
- 第一版就上整理模型：收件箱还没有稳定的追加语义时，整理会吞掉失败的观察。不选。

## 横切面

- 安全：在 follow-on 规格的「沙箱补强」落地前，`danger-full-access` 的 shell 仍可能写到 `$DSH_HOME`。本设计把正常工具路径收紧。文档不把当前沙箱说成已经挡住。
- 前缀缓存：索引段字节不变则不改写。观察追加但不改变索引时，不碰这段。
- 合并：全新的 `packages/boom/memory-files`，不改 `agent-instructions`。
