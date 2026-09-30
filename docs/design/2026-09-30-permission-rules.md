---
title: 权限规则的存储与执行前判定
type: design
status: draft
date: 2026-09-30
related:
  - specs/2026-09-30-permission-rules.md
---

# 权限规则的存储与执行前判定

> 规则放在 `DSH_HOME` 的 storage domain。`tools/pre-execute` 返回允许、拒绝或询问。询问才进入今天的审批。

## 背景与范围

规格见 [../specs/2026-09-30-permission-rules.md](../specs/2026-09-30-permission-rules.md)。沙箱后端不动。`user-approval` 不监听 `tools/pre-execute`。`confine` 在 `bash-sandbox` 的执行体内。

## 目标与非目标

- 目标：进程重启后仍能记住允许和拒绝。规则文件在 `DSH_HOME` 一侧，不在工作区仓库里。
- 非目标：在本设计里做技能正文扫描，或把规则存进 `.claude/settings.json`。

## 设计

### 存储

`packages/boom/permission-rules` 用 `defineDomain` 声明领域，再用 `ctx.storageDomain.open` 打开。`ctx.storageDomain` 是宿主打开接口，后端由 `storage-domain` 的 `backend` / `routes` 选择。领域名是 `boom_permission_rules`：`UNIT_NAME_RE` 是 `/^[a-z][a-z0-9_]*$/`，带点的名字在加载时就会抛。`tables` 使用 zod，例如 `tables: { rules: domainTable(ruleSchema) }`。不设置 `invalidRecords: 'backup-and-skip'`，那个选项会把坏记录挪走并当成没有这些规则。缺省是整次 `open` 拒绝。打不开或校验失败时，插件拒绝进入「无规则」状态，并把需要授权的调用判为拒绝。领域记录留在宿主侧，不写成会话事件。

```ts
type Rule = {
  id: string
  tool: string          // 工具名，或字面量 'skill'
  target: string        // 路径 glob，或 shell 段的有界模式
  effect: 'allow' | 'deny'
  scope: 'session' | 'persistent'
  workspaceKey: string
}
```

存储里只有 `session` 与 `persistent`。托管规则来自插件 Config，不进可撤销的那一组。`revoke` 碰不到它。会话规则的键带 `sessionId`。人类要直接看文件时，组合选用 `storage-json`。

### 判定

挂在 `tools/pre-execute`。这条瀑布的终点默认是 `{kind:'allow'}`，不会调用 `ctx.approval`。`ask`/`never` 只在有人返回 `{kind:'ask'}` 之后，由 `serviceAsk` 进入。`never` 在 `approval.request` 里、应答者之前就返回 `rejected`。

1. 托管 deny
2. 持久 deny
3. 会话 deny
4. 持久 allow
5. 会话 allow

整次结果取各段里更严的一个：拒绝严于询问，询问严于放行。deny 返回 `{kind:'deny'}`。全部段都放行才返回 `{kind:'allow'}`，这只跳过 `serviceAsk`。未命中的段是询问，监听器返回 `{kind:'ask'}`，不调用 `next()`。`next()` 会落到默认放行，`git status && git diff` 就不会询问。

`confine` 不在这个监听器里。持久 allow 之后，`bash-sandbox` 仍按现有 `SandboxPolicy` 做本次 confine。

普通段按空白前缀匹配：`git push` 能套上 `git push origin main`，`git status` 套不上 `git diff`。`git push`、`rm`、`chmod`，以及向 shell 管道下载执行的段，放行必须覆盖整段；拒绝仍按前缀。没有完整放行、也没有拒绝时询问，不直接拒绝。

`bash` 在判定前拆段。拆分只认 `&&`、`||`、`;`、`|`、换行，不做完整 shell 语法。引号内的分隔符不拆。`| sh` 与 `| bash` 保持为一段，避免 `curl` 和 `sh` 各自吃到前缀放行。拆不了时：有任一 deny 模式可能命中则拒绝，否则询问。

### 审批词汇

`serviceAsk` 今天只把 `allowed-once` 当成放行，其它结果进 `assertNever`。`allow-persist` 必须在这里放行，`deny-persist` 必须在这里拒绝。同时改 `OUTCOMES`、不变式里的 `APPROVAL_OUTCOMES`、`ui-approval` 的应答类型，以及 `packages/acp/acp/src/index.ts` 里把非 `allow-once` 收成 `rejected` 的那一处。保留 `allowed-once`。

请求仍由 `serviceAsk` 组装，继续带 `callId`，不附工具参数。`displayReason` 只用于展示。本插件按这次 `callId` 写下规则，因为 `approval/decided` 里没有目标。记住按钮只在本插件挂上时出现，`web` 不挂本包时界面不多出这个动作。

选择持久结果时写入 domain，并追加非表层 `approval/rule`（创建、命中或撤销）。这个事件要进 `SessionEventMap`，或在登记前标 `ignorable: true`。模型仍只看见消费方最终的工具结果。

### 技能

`tool-skill` 有两条正文入口：`agent/pre-step` 对 `/名字` 的预注入，以及 `skill` 工具的 `execute`。两处都只按技能名拒绝。deny 则跳过正文。没有规则时照常注入，不走工具那种「未命中即询问」。正文扫描不在本包。

### 撤销

插件提供宿主侧 API `revoke(id)`，由设置页或 CLI 调用。下一次 `pre-execute` 读到的 domain 状态不再含该规则。

## 备选方案

- 把 YAML 放进仓库：代理可以改自己的授权。不选。
- 只在 `permission-presets` 里加档位：档位不是「这个目标以后都允许」。不选。
- 让 `auto-review` 模型决定记住什么：规格要求记住的是用户动作。不选。
- 持久 allow 时跳过 `confine`：会把记住的放行变成关沙箱。不选。

## 横切面

- 安全：失败关闭。托管 deny 不接受会话 allow。规则放行不改变沙箱模式。
- 前缀缓存：规则不进系统提示词。命中只产生非表层日志和最终那一条工具结果。
- 合并：审批结果联合、`serviceAsk`、`ui-approval` 和 ACP 的选项映射是点名的上游改动。判定逻辑在 `packages/boom/permission-rules`。`web` 不挂本包。
