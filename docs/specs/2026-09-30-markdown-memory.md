---
title: 工作区与全局 Markdown 记忆
type: spec
status: draft
date: 2026-09-30
related:
  - design/2026-09-30-markdown-memory.md
---

# 工作区与全局 Markdown 记忆

> 新事实先记入收件箱，整理后才成为主题；提示词里只有一份有字节预算的索引。人可以直接改这些 Markdown。代理只能把观察追加到收件箱，并按索引读取正文。

## 动机

dsh `0.2.0-rc.2` 没有 `packages/memory`。会话日志是逐字层，`agent-instructions` 注入的是工作区指令，不是可维护的个人知识。若在写 boom 自己的系统提示时把决定写进提示词，以后拆不出来。

## 现状

- `ctx.systemPrompt` 按段拼接；段不变时系统前缀保持稳定（`packages/core/system-prompt/README.zh.md`）。
- `ctx.storageDomain` 适合结构化记录，不适合人直接改的长文。
- 同一 git 仓库的多个 worktree 若按 checkout 路径分记忆，会把同一项目拆成两份。

## 用户故事

### US-1：索引占住提示词槽位（P1）

作为维护者，我希望系统提示里有一份记忆索引而不是散落的事实，以便换模型或压缩都不会把知识 baked 进人设。

**验收场景**：

- Given 两份 `INDEX.md` 都没有可列条目，When 启动 `boom` profile，Then 段名 `boom.memory-index` 的文本是「记忆索引为空。事实写入记忆收件箱，不写入这段提示。」且不含具体事实。
- Given 收件箱已有文件而主题未改，When 组装索引，Then 索引列出这些收件箱的相对路径，不用上面的固定句代替这份列表。
- Given 索引文件存在，When 连续两轮索引字节不变，Then 该段文本不变。

### US-2：观察先入收件箱（P1）

**验收场景**：

- Given 一轮会话结束或被中止，且运行中产生了记忆观察，When 回合收尾，Then 观察追加到当天收件箱，主题文件不变。
- Given 没有新观察，When 回合收尾，Then 不改写索引。

### US-3：同一仓库的 worktree 共用工作区记忆（P1）

**验收场景**：

- Given 两个 worktree 指向同一 git common dir，When 分别读取工作区记忆，Then 路径相同。
- Given 两个互不相关的仓库，When 读取工作区记忆，Then 路径不同。

### US-4：整理可以重放（P2）

**验收场景**：

- Given 收件箱有未整理条目，When 维护任务开始，Then 它先领取一份快照；模型调用失败或租约过期时，主题文件与领取前一致，收件箱仍在。
- Given 整理成功，When 再次读取索引，Then 索引指向主题，并注明收件箱已消费的范围。

## 功能需求

- FR-001：文件 MUST 位于 `$DSH_HOME/memory/`。全局与按仓库区分的工作区 MUST 分开。
- FR-002：工作区键 MUST 来自 git common dir；不是 git 仓库时 MUST 退化为工作区根路径的稳定哈希。
- FR-003：注入模型的 MUST 只有索引段，字节上限默认 4096。正文 MUST 通过本插件的 `memory_read` 打开。相对路径经解析（含符号链接）后 MUST 仍在全局或该工作区记忆根内，且 MUST 只打开 `.md`。拒绝字符串 `..` 不够。MUST NOT 靠通用文件工具充当这条契约。
- FR-004：代理 MUST NOT 通过本插件获得写入任意 `$DSH_HOME` 的工具。写入收件箱 MUST 走插件自己的追加 API。
- FR-005：US-4 的模型调用 MUST 在存储事务之外。提交主题与标记收件箱 MUST 是失败则整体不生效的一步。
- FR-006：空索引段的 `systemPrompt` 段名 MUST 是 `boom.memory-index`，order MUST 小于以后的 boom 人设段，且 MUST 设 `interpolate: false`。两份 INDEX 都没有可列条目时，文本 MUST 是「记忆索引为空。事实写入记忆收件箱，不写入这段提示。」已有收件箱文件时，索引 MUST 列出其相对路径，此时 MUST NOT 改 topics。

## 成功标准

- SC-001：只有收件箱变化时，主题目录的文件哈希不变。
- SC-002：索引超过上限时，注入文本长度不超过上限，且含有截断标记。

## 非目标

- 嵌入、重排、ReMe、`auto_dream`、把会话日志再复制一份。
- 第一版就做跨机器同步。
- 在 US-4 完成前阻塞 US-1 至 US-3。没有整理任务时，收件箱本身就可以被索引和阅读。
