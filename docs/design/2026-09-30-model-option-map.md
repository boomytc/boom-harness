---
title: 模型选项映射的编译与绑定
type: design
status: draft
date: 2026-09-30
related:
  - specs/2026-09-30-model-option-map.md
---

# 模型选项映射的编译与绑定

> 模型加载时把声明编译成有序路径表。`bind` 对着已冻结的 `LlmCallConfig` 产生一次 JSON merge patch。

## 背景与范围

规格见 [../specs/2026-09-30-model-option-map.md](../specs/2026-09-30-model-option-map.md)。不改 HTTP 栈，不搬 ZCode 的供应商目录。

## 目标与非目标

- 目标：加一个网关字段时只加一条声明。
- 非目标：在请求路径上解释一套模板语言。

## 设计

`packages/boom/llm-option-map` 导出两个纯函数：

```ts
compile(declarations: Declaration[]): CompiledMap
bind(compiled: CompiledMap, config: LlmCallConfig): JsonMergePatch
```

`Declaration` 是 `{ match, overlay }`。`overlay` 把 `reasoningEffort` 或 `maxTokens` 映到一条键路径、可选取值表，以及这个选项族要删掉的其它键。`temperature` 和 `stop` 不在这张表里。`bind` 只读 `LlmCallConfig` 的 `reasoningEffort` 与 `maxTokens`。配置里没有的标量不产生键。

键路径是键的列表，`bind` 把它变成 RFC 7396 要的嵌套对象。`['thinking', 'type']` 是 `{ thinking: { type } }`，不是字面键 `thinking.type`。应用补丁前先删除 `replaces` 里的键，再做 merge。只 merge 不删除，就无法满足「不出现另一个推理字段」。

`compile` 拒绝未知标量和越界取值，不静默丢掉。重复路径时后者覆盖前者，并留下诊断。模型名表从前往后合并，后者赢，不按通配符长短自动选择。overlay 只含上下文窗口、模态和本映射的声明，不含账号与 URL。没有任何 glob 命中时，与没有 `CompiledMap` 一样返回原对象。

### 挂载点

`llm-deepseek` 在 `serialize()` 的返回值上调用 `bind`，再 merge。不要走 `prepareRequestExtensions`：那个扩展点只许追加字段，和基座同名会抛 `REQUEST_EXTENSION`，也不能删键。

`llm-pi-ai` 没有 dsh 持有的请求体。它把 `maxTokens` 和 `reasoning` 交给 `streamSimple`，HTTP 体在 pi-ai 内部生成。不在这里插入 `bind`，也不外包一层 fetch。`max_completion_tokens` / `max_tokens` 的夹具是 `bind` 应用到夹具对象上的结果。

`boom` profile 提供声明表。`web` profile 不加载该表时，适配器发现没有 `CompiledMap` 就保持今天的体。无映射则返回原对象。

声明表示例（不是要提交的供应商目录，match 是虚构的）：

```yaml
- match: example-gw-*
  overlay:
    reasoningEffort:
      path: [reasoning_effort]
      values: { high: high, low: low }
      replaces: [thinking]
    maxTokens:
      path: [max_tokens]
```

DeepSeek 的真实夹具必须跟 `serialize()` 一致：`off` 才是 `thinking.type: disabled`；`low` / `high` / `max` 都是 `thinking.type: enabled`，档位在 `output_config.effort`。

## 备选方案

- 在每个适配器里写 `if (provider === ...)`：加模型就要改适配器。这是现状，规格要离开它。
- 运行时用 JSONPath 字符串解释声明：请求路径上的错误会变成难以测试的动态行为。编译期拒绝更合适。
- 包一层 AI SDK：多一个 HTTP 栈，和 `llm-pi-ai` 重叠。不选。
- 把 `high` 映成 `thinking.type: enabled`、`low` 映成 `disabled` 来演示 `deepseek-*`：和 `serialize()` 的对照基线不一致。不选。

## 横切面

- 前缀缓存：映射不改变 `LlmCallConfig` 的标量，因此不额外制造 header 快照。请求体字段变化本来就会改变提供方缓存键，这是声明的目的。
- 合并：`llm-deepseek` 的调用点要小，并用「无映射则原样返回」保住上游测试。
- 安全：声明来自 profile 配置，不来自模型输出。
