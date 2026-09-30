---
title: 实现模型选项映射
type: plan
status: draft
date: 2026-09-30
related:
  - specs/2026-09-30-model-option-map.md
  - design/2026-09-30-model-option-map.md
---

# 实现模型选项映射

> 编译成有序路径表，在 `bind` 时对已冻结的调用配置产生 merge patch。没有映射、或没有任何 glob 命中时，返回原对象。

## 目标

- DeepSeek 夹具对照 `serialize()` 之后的对象。另外两条夹具是 `bind` 应用到对象上的 `max_completion_tokens` 与 `max_tokens`。
- `web` profile 不加载映射时，现有适配器测试的请求体不变。

## 非目标

- 供应商目录搬迁、虚拟模型、新的 HTTP 客户端。
- 在 `llm-pi-ai` 内部改 HTTP 体。
- 改模型的上下文窗口、模态或其它能力元数据。

## 背景与依据

[../design/2026-09-30-model-option-map.md](../design/2026-09-30-model-option-map.md)。

## 任务分解

- [ ] 新增 `packages/boom/llm-option-map`，实现 `compile` 与 `bind`，以及按模型名匹配的选项声明从前往后合并。`overlay` 只含 `reasoningEffort` 与 `maxTokens` 的声明（键路径、取值表或范围约束、同族删键），不覆盖窗口和模态。键路径变成嵌套对象。`bind` 只读这两个标量。
- [ ] 未知标量、越界取值在 `compile` 或 `bind` 抛错。重复路径以后者为准，并在诊断里保留覆盖记录。应用补丁前按 `replaces` 删除同族旧键，再做 RFC 7396 merge。
- [ ] 在 `llm-deepseek` 的 `serialize()` 返回值上调用 `bind`。不要走 `prepareRequestExtensions`。无 `CompiledMap` 或没有 glob 命中时返回原对象。
- [ ] `boom` profile 放入最小声明表。不提交 ZCode 的 `zcode-builtin.json`。
- [ ] 快照测试：DeepSeek 的 `off` / `low` / `high` / `max` 与 `serialize()` 的两个字段一致，且 `replaces` 删掉的推理字段不出现。另外两条夹具是 patch 应用到对象后的结果。
- [ ] 测试：同一声明与同一 `LlmCallConfig` 得到同一 patch。
- [ ] 验收 US-2：先声明 `gpt-*` 再声明 `gpt-4.1*`，后命中的选项声明覆盖前者，窗口与模态保持原值；无 glob 命中或无 `CompiledMap` 时返回原对象。
- [ ] 不挂映射时跑 `llm-deepseek` 的现有请求测试。

## 风险与依赖

- 前置：bootstrap。
- 把补丁插进 `prepareRequestExtensions` 会在同名字段上抛 `REQUEST_EXTENSION`，并且删不掉旧键。
