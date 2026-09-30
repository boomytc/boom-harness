---
title: 模型选项到网关 JSON 的映射
type: spec
status: draft
date: 2026-09-30
related:
  - design/2026-09-30-model-option-map.md
---

# 模型选项到网关 JSON 的映射

> 推理档和最大输出在 `LlmCallConfig` 里仍是标量。模型创建时编译路径表；发送时 `bind` 才把这两个标量写成 JSON merge patch。

## 动机

`LlmCallConfig` 冻结 `reasoningEffort`、`maxTokens` 等标量（`packages/llm/llm/src/call-config.ts`）。注释写明这些字段与 `GenerateOptions` 同名，提供方专用的请求形状还没有归属。pi-ai 的 `reasoningEfforts` 与 `compat` 只覆盖已知拼写和有限开关。各网关实际要的是 `thinking.type`、`reasoning_effort`、`max_tokens`、`max_completion_tokens` 或 `max_output_tokens`。在每个适配器里手写分支，会在加模型时漏掉。

## 现状

- 调用配置在请求之间保持稳定，变化会记入请求 header，以免静默漂移。
- 适配器负责把配置变成 HTTP 体。本规格不引入第二套 HTTP 客户端。

## 用户故事

### US-1：创建模型时编译，发送时只填值（P1）

作为适配器，我希望模型目录在加载时就知道推理档写到哪个字段，以便每次请求只做值替换。

**验收场景**：

- Given 模型声明推理档写入 `reasoning_effort`，且本次配置为 `high`，When 组装请求体，Then 该字段为 `high`，且不出现另一个推理字段。
- Given 模型声明最大输出写入 `max_completion_tokens`，When `maxTokens` 为 1024，Then 体里该字段为 1024，且没有 `max_tokens`。
- Given 两个补丁写同一路径，When 编译，Then 后者覆盖前者，顺序与声明顺序一致。

### US-2：按模型名叠加选项声明（P1）

**验收场景**：

- Given 有序规则先写 `gpt-*` 再写 `gpt-4.1*`，When 为 `gpt-4.1` 编译映射，Then 后出现的命中规则覆盖前者的 `reasoningEffort` / `maxTokens` 选项声明（键路径、取值表或范围约束、同族删键）；上下文窗口与模态保持模型原有值。
- Given boom 表里没有任何 glob 命中，或根本没有 `CompiledMap`，When 解析，Then 返回原对象，与今天的适配器默认值相同。

## 功能需求

- FR-001：映射 MUST 在模型对象创建时编译成有序路径表。JSON merge patch MUST 只在 `bind` 时、对着已冻结的 `LlmCallConfig` 产生。请求路径 MUST NOT 再解释规则语言。本映射只覆盖 `reasoningEffort` 与 `maxTokens`。
- FR-002：绑定的值 MUST 来自已经冻结的 `LlmCallConfig`。映射 MUST NOT 改写 provider、model 或触发新的 header 快照，除非补丁所依据的标量本身变了。
- FR-003：未知选项或越界取值 MUST 在编译或绑定时报错，MUST NOT 静默省略。
- FR-004：本规格 MUST NOT 改变 `sdk-minimal` 与未挂 boom 插件的 `web` profile 的请求体。

## 成功标准

- SC-001：DeepSeek 夹具是 `serialize()` 返回值经 `bind` 之后的对象。`off` 写成 `thinking.type: disabled` 且没有 `output_config`；`low` / `high` / `max` 写成 `thinking.type: enabled` 加上 `output_config.effort`。另外两条是 `bind` 产出的 merge patch 应用到夹具对象上的结果，分别覆盖 `max_completion_tokens` 与 `max_tokens`。`llm-pi-ai` 只把标量交给 `streamSimple`，不在那里假装有一份 dsh 持有的 HTTP 体。
- SC-002：编译是纯函数：同一声明与同一配置得到同一 JSON。

## 非目标

- 官方网关改写、账号、套餐、可见性与排序字段。
- 替换 `llm-pi-ai` 或新包一层 AI SDK `fetch`。
- 虚拟模型（选择身份与物理路由分离）。那是参照项，未排期。
- 覆盖模型的上下文窗口、模态或其它能力元数据。
