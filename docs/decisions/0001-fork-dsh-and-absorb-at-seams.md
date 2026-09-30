---
title: git fork DeepSeek Harness，吸收只发生在 seam 上
type: decision
status: accepted
date: 2026-09-30
---

# 0001. git fork DeepSeek Harness，吸收只发生在 seam 上

## Status

accepted

## Context

boom-harness 要成为一个自有 harness，做法对齐天枢：一个仓库拥有运行时，外层是自己的产品。底座选 DeepSeek Harness，是因为它没有可以单独抠走的内核——CLI、Web、桌面、Python SDK 和 vendor 里的 Cordis 是同一列发布火车。`639ed01539` 上已跟踪的 `package.json` 是 352 份：337 个包名是 `@deepseek-ai/*`，2 份 snapshot 没有 `name`，其余 13 份是夹具、技能模板和 `dsh-python-runtime-closure`。`docs/architecture.zh.md` 没有写这个数字。本地 `deepseek-harness` 的 `master` 等于 `upstream/master` 的 `639ed01539`（`0.2.0-rc.2`），没有私人提交。

同时要从 pi、grok-build、ZCode、QwenPaw 吸收少数行为。这四个仓库语言、进程模型和许可证都不同。dsh 处于开发者预览，上游仍在合并。把四个产品树并进来，或在第一周改名这 337 个 `@deepseek-ai/*` 包，都会让 `git merge upstream/master` 无法进行。

品牌规范允许写「基于 DeepSeek Harness 构建」，产品名不能叫 DeepSeek Harness。MIT 版权声明必须留在副本里。pi 是 MIT；grok-build、ZCode、QwenPaw 是 Apache-2.0。把 Apache 源码贴进 MIT 树，贴进去的部分仍是 Apache，还要带 NOTICE。

## Decision

我们将把 `639ed01539` 的已跟踪源码连同 git 历史 fork 为 boom-harness。`upstream` 指向 `https://github.com/deepseek-ai/deepseek-harness.git`。现有 `boomytc/deepseek-harness` 继续当干净镜像。

新行为放在 `packages/boom/<name>`，包名暂时用 `@deepseek-ai/dsh-boom-<name>`，等将来的 scope 改名脚本一起改。`pnpm-workspace.yaml` 的 `packages/*/*` 已经覆盖这个目录。

随附 profile 不是包，而是 `packages/boot/app-boot/src/profile.ts` 里的 `PROFILE_TEMPLATES`。`web` 的 bundles 只有 `@deepseek-ai/dsh-base` 与 `@deepseek-ai/dsh-web-app`。boom 的挂载是在这张表里增加 `boom`，bundles 为上述两个再加 `@deepseek-ai/dsh-boom-bundle`。`web`、`headless`、`sdk`、`sdk-minimal`、`acp` 的模板不改。`desktop` 是 Electron 独占的 `$DSH_HOME/profiles/desktop`，不在这张表里；bootstrap 不改它。

`dsh --profile web --dump-config` 的配置树保持上游组合，用来对照。写在核心循环里的恢复策略对所有 profile 生效，包括 `web`：读工具可以按 `safe` 重放，写工具保持 `never`。同一条会话不因换 profile 而换成另一种崩溃结算。进度回调、权限规则、目录、记忆和选项映射仍只挂在 `boom` 模板上。

只有 seam 本身封闭、插件听不到事件时，才改上游文件。这种改动在对应 design 里点名，并保持能单独 revert。

第一天不改 `dsh` 命令、`~/.dsh`、`DSH_HOME` 和这 337 个既有包名。fork 之后先让未改名的树 `pnpm install`、`pnpm run build`、`pnpm dsh web --no-open` 通过，再写行为。

四个参照仓库留在原地。吸收是在 dsh seam 上重写想法。

## Consequences

- 上游合并的冲突面主要在被点名的核心文件，而不是整棵 `packages/`。
- 默认启动仍是 `dsh web`。`boom` 模板要显式 `--profile boom` 才挂上 boom 包。把默认 profile 或 Desktop 的 bundle 列表改成 boom，是以后的决定。
- `web` 的配置树与上游相同。`web` 的崩溃结算会随核心里的 `replay` 声明一起变，这是有意的。
- `@deepseek-ai/dsh-boom-*` 这个前缀是过渡名。对外发布前必须改掉，否则会占用 DeepSeek 的 scope。
- 本目录 `docs/` 先于 fork 存在。执行 bootstrap 时先把 `docs/` 移走，清空目录再 clone，然后把 `docs/` 放回。
