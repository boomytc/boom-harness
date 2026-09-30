---
title: 可记住的工具与目标规则
type: spec
status: draft
date: 2026-09-30
related:
  - design/2026-09-30-permission-rules.md
---

# 可记住的工具与目标规则

> 用户的一次放行或拒绝可以变成以后还生效的规则；拒绝压过放行；复合 shell 不能靠后半段把窄授权放大。

## 动机

`user-approval` 写明：结果只有 `allowed-once`，没有记住的规则、撤销或授权存储；会话策略只有 `ask` / `never`（`packages/interaction/user-approval/README.zh.md`）。沙箱模式是文件能力档位（`read-only`、`workspace-write`、`danger-full-access`），不是「这个命令以后都允许」。实验性的 `auto-review` 和 `guard` 的重复调用提醒都不裁定授权。

## 现状

- 执行前瀑布是 `tools/pre-execute`（钩子、权限、沙箱），然后才 `tools/execute`（`docs/tool-execution-pipeline.zh.md`）。
- 审批请求不携带工具参数。应答者看到工具名、原因和可选调用 id。
- 规则如果写在仓库里，代理可以用文件工具改掉自己的授权。

## 用户故事

### US-1：拒绝记住，并压过放行（P1）

作为用户，我希望「永远拒绝 `git push`」在同一工作区的下一场会话仍然有效，以便一次危险操作不会因为旧的前缀放行而通过。

**验收场景**：

- Given 持久规则拒绝工具 `bash`、目标匹配 `git push`，且另有一条放行 `git`，When 模型调用 `bash` 且参数为 `git push origin main`，Then 调用不执行，结果为拒绝，且不弹出审批。
- Given 持久规则拒绝 `git push`，且另有一条放行 `git status`，When 参数为 `git status && git push`，Then 整次拒绝，不执行。
- Given 只有放行 `git status`，没有关于 `git diff` 的规则，且会话策略是 `ask`，When 参数为 `git status && git diff`，Then 不执行，并进入询问，而不是直接拒绝。

### US-2：放行可以记住（P1）

**验收场景**：

- Given 审批界面出现且用户选择记住放行，When 下一场会话对同一工作区发出同一工具与同一目标模式，Then 不再询问，并在日志里留下规则命中记录。
- Given 用户选择只此一次，When 下一场会话，Then 再次询问。

### US-3：托管拒绝不能被会话规则打开（P1）

**验收场景**：

- Given 部署配置里的托管规则拒绝某目标，When 会话规则放行同一目标，Then 仍然拒绝。

### US-4：技能名可以拒绝注入（P1）

**验收场景**：

- Given 持久规则拒绝技能名 `danger-skill`，When 该技能经 `/名字` 预注入或经 `skill` 工具执行，Then 两处正文都不进入模型，并记录拒绝。没有规则时两处都照常注入，不进入询问。

## 功能需求

- FR-001：规则 MUST 持久化在 `DSH_HOME` 一侧的 `ctx.storageDomain`，MUST NOT 放在会话工作区目录内。
- FR-002：判定顺序 MUST 为：托管拒绝、持久拒绝、会话拒绝、持久放行、会话放行，然后才是现有 `ask`/`never` 审批。规则放行只跳过这一次工具审批。`bash-sandbox` 的 `confine` 仍在执行体内运行。
- FR-003：`bash` MUST 在判定前按段切开。分隔符是 `&&`、`||`、`;`、`|` 与换行；引号内的分隔符不切开。向 shell 管道下载执行（`| sh` 与 `| bash`）保持为一段，不按 `|` 切开。每一段单独匹配。调用结果取各段里更严的一个：拒绝严于询问，询问严于放行。一段没有命中任何规则时，该段是询问，再交给会话的 `ask`/`never`。拆不开的命令：任一拒绝模式可能命中则整次拒绝，否则整次询问。
- FR-004：`git push`、`rm`、`chmod`、向 shell 管道下载执行，MUST NOT 吃到只按命令前缀写下的旧放行。这些段的放行必须覆盖整段。没有完整放行、也没有拒绝时，该段是询问，不直接拒绝。拒绝仍按前缀匹配。
- FR-005：审批结果词汇 MUST 增加「记住放行」和「记住拒绝」，并保留 `allowed-once`。
- FR-006：规则命中、创建和撤销 MUST 写入会话日志的非表层事件，模型只看到最终工具结果。
- FR-007：撤销 MUST 让下一次调用不再命中该规则。
- FR-008：损坏的规则存储 MUST 失败关闭（拒绝需要授权的调用），MUST NOT 当成没有规则。

## 成功标准

- SC-001：US-1 的复合命令测试不调用 `bash` 的 `execute`。
- SC-002：重启进程后 US-2 的持久放行仍然命中。
- SC-003：`web` profile 的行为与未挂本插件时一致。

## 非目标

- 模型自动分类器、ACP `yolo`、把 `.claude/settings.json` 当作存储。
- 技能正文扫描。本规格只拒绝技能名。正文扫描是 follow-on 的 US-8，需要误报语料之后再做。
- 替换 Seatbelt、bwrap、Landlock。本规格只决定要不要执行，不决定进程怎么关进内核。
