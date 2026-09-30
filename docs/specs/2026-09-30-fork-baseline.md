---
title: fork 基线与产品挂载
type: spec
status: draft
date: 2026-09-30
related:
  - decisions/0001-fork-dsh-and-absorb-at-seams.md
---

# fork 基线与产品挂载

> 先得到一棵能构建、能打开、且 `web` profile 与上游一致的树，再挂 boom 自己的 profile。

## 动机

行为规格都假设 Cordis、会话日志和工具管线已经按 `0.2.0-rc.2` 工作。在未改名的树跑通之前改 `agent-loop` 或包名，失败无法归因。

## 现状

- 分叉点：`https://github.com/deepseek-ai/deepseek-harness` 的 `639ed01539`，版本 `0.2.0-rc.2`。
- 工作区产物不在 git 里：`node_modules/`、`lib/`、`*.tsbuildinfo` 是全局忽略。`dist` 只忽略 `/dist/`、`apps/web/dist/` 和 `dist-exe/`。
- 命令是 `apps/cli` 的 `dsh`。数据目录默认 `~/.dsh`（`DSH_HOME`）。
- 品牌规范：说明文字可以写「基于 DeepSeek Harness」；产品名不使用 DeepSeek Harness；保留 `LICENSE` 的 DeepSeek 版权声明。

## 用户故事

### US-1：未改名的树能跑（P0）

作为维护者，我希望 fork 后的源码在本机完成安装和构建，以便后面的差异有对照。

**独立验证**：不挂任何 `packages/boom` 插件。

**验收场景**：

- Given 干净的 `639ed01539` 工作区，When 执行 `pnpm install` 与 `pnpm run build`，Then 两者退出码为 0。
- Given 构建完成，When 执行 `pnpm dsh web --no-open`，Then 进程监听本机端口并打印 URL。
- Given 同一个构建，且 `DSH_HOME` 是新建的空目录，When 执行 `dsh --profile web --dump-config`，Then 打印的 bundles 仍是 `@deepseek-ai/dsh-base` 与 `@deepseek-ai/dsh-web-app`，没有 `@deepseek-ai/dsh-boom-*`。profile 目录已经存在时，dump 不会按 `PROFILE_TEMPLATES` 重写 bundles。空字符串 `DSH_HOME` 会被当成未设置。

### US-2：boom profile 是加法（P0）

作为维护者，我希望 boom 插件挂在单独 profile 上，以便 `web` 仍是上游对照。

**独立验证**：只增加 profile 与空 bundle，不改变工具行为。

**验收场景**：

- Given 空的 boom bundle，When 使用 `--profile web`，Then 配置树不出现 `@deepseek-ai/dsh-boom-*`。
- Given `PROFILE_TEMPLATES.boom` 已加入，When 执行 `dsh --profile boom --dump-config`，Then bundles 依次为 `@deepseek-ai/dsh-base`、`@deepseek-ai/dsh-web-app`、`@deepseek-ai/dsh-boom-bundle`。
- Given 同一个构建，When dump `headless`、`sdk`、`sdk-minimal` 或 `acp`，Then 它们的 bundles 与上游模板相同。

## 功能需求

- FR-001：仓库 MUST 记录分叉点提交、版本号和 `upstream` remote。
- FR-002：`docs/` MUST 在 fork 提交中保留。
- FR-003：默认数据目录 MUST 仍是 `DSH_HOME`，缺省 `~/.dsh`。
- FR-004：既有包名与 `dsh` bin MUST 在本规格内保持不变。
- FR-005：新插件包 MUST 位于 `packages/boom/<name>`，名称 MUST 为 `@deepseek-ai/dsh-boom-<name>`。boom bundle MUST 声明 `dsh.bundle.patch`，并且 MUST 成为 `apps/cli` 的依赖，这样安装目录才能解析到它。
- FR-006：`boom` MUST 写进 `PROFILE_TEMPLATES`。MUST NOT 另做 `packages/boom/profile` 来冒充 profile。自定义的 `$DSH_HOME/profiles/<name>` 不能代替随附模板，否则换一台机器就没有这条组合。
- FR-007：产品说明的对外名称 `[NEEDS CLARIFICATION: 名称未定。文档暂称 boom-harness，bin 仍为 dsh]`。名称确定之前，bootstrap MUST NOT 改写上游 README 的产品介绍。

## 成功标准

- SC-001：`web` profile 的 dump 与上游同一提交的 dump 无 boom 条目。
- SC-002：`boom` profile 能完成与 US-1 相同的启动。

## 非目标

- 双命令名、数据目录迁移、scope 改名、桌面安装包重签名。
- 修改 Electron 独占的 `desktop` profile。桌面端要挂 boom bundle 时另开一步，不在本规格里改 `$DSH_HOME/profiles/desktop`。
- 把个人知识写进系统提示词。记忆规格拥有那个槽位。
