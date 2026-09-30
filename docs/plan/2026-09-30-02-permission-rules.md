---
title: 实现权限规则
type: plan
status: draft
date: 2026-09-30
related:
  - specs/2026-09-30-permission-rules.md
  - design/2026-09-30-permission-rules.md
---

# 实现权限规则

> `boom` profile 在审批之前应用持久规则。`web` profile 不加载该包。

## 目标

- 复合 `bash` 不会把只覆盖前半段的放行扩大到 `git push`。
- 进程重启后持久规则仍在。
- 记住的规则不出现在会话工作区目录里。

## 非目标

- 技能正文扫描、沙箱后端替换、托管策略的 UI 编辑器。托管规则先只来自插件 Config。

## 背景与依据

[../design/2026-09-30-permission-rules.md](../design/2026-09-30-permission-rules.md)。

## 任务分解

- [ ] 新增 `packages/boom/permission-rules`。`defineDomain` 的 `name` 为 `boom_permission_rules`，`version` 为 1，`tables` 用 zod 的 `domainTable`。用 `ctx.storageDomain.open` 打开。不设置 `invalidRecords: 'backup-and-skip'`。存储范围只有 `session` 与 `persistent`。托管规则只来自插件 Config。
- [ ] 实现判定顺序与 `bash` 分段。普通段按空白前缀匹配。`git push`、`rm`、`chmod` 和 `| sh` / `| bash` 的放行必须覆盖整段，拒绝仍按前缀。管道到 shell 保持为一段。拆不开时，可能命中拒绝模式才拒绝，否则询问。
- [ ] 挂到 `tools/pre-execute`。全段放行返回 `{kind:'allow'}`。有未命中的段返回 `{kind:'ask'}`，不调用 `next()`。放行之后 `confine` 仍执行。
- [ ] 扩展 `allowed-once` 旁边的 `allow-persist` 与 `deny-persist`，并让 `serviceAsk`、`OUTCOMES`、`APPROVAL_OUTCOMES`、`ui-approval` 和 ACP 的选项映射接受它们。请求仍用 `callId`，不附工具参数。记住按钮只在本插件挂上时出现。
- [ ] 持久结果按 `callId` 写入 domain，并追加已登记或标了 `ignorable` 的 `approval/rule`。提供 `revoke`。
- [ ] `/名字` 预注入和 `skill` 工具执行都按技能名拒绝。没有规则时照常注入。
- [ ] domain 打不开或记录校验失败时，需要授权的调用拒绝。
- [ ] 测试：有 `git push` 的拒绝规则时，`git status && git push` 不调用 execute。只有 `git status` 的放行、且策略为 `ask` 时，`git status && git diff` 进入询问，不直接拒绝。
- [ ] 测试：重启后 allow 仍命中；revoke 后再次询问。
- [ ] 测试：托管 deny 加会话 allow 时仍然拒绝。
- [ ] 确认 `web` profile 的 dump 不含本包，现有 approval 测试不加载 boom domain 时仍通过。

## 风险与依赖

- 前置：bootstrap。与第 1 条无代码依赖，默认同序进行是为了少同时改 `tools/pre-execute`。
- shell 分段不是完整解析器 → 可能命中拒绝模式才拒绝，否则询问，并在测试里固定这个选择。
