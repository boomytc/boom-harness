---
title: fork bootstrap
type: plan
status: draft
date: 2026-09-30
related:
  - specs/2026-09-30-fork-baseline.md
  - decisions/0001-fork-dsh-and-absorb-at-seams.md
  - decisions/0003-branch-and-release-conventions.md
---

# fork bootstrap

> 得到一棵等于 `639ed01539` 的可运行树，保留本目录 `docs/`，并加上空的 `boom` profile。

## 目标

- `pnpm install` 与 `pnpm run build` 在 fork 上通过。
- `dsh --profile web` 的配置树不含 boom 包。
- `dsh --profile boom` 能启动，且只比 web 多一个空 bundle。

## 非目标

- 行为规格 01–06。
- 改 bin、改 `DSH_HOME`、改既有包名。

## 背景与依据

[../specs/2026-09-30-fork-baseline.md](../specs/2026-09-30-fork-baseline.md)。这些文档现在在本目录。本目录已经是 git 仓库，`origin` 是 `https://github.com/boomytc/boom-harness.git`，`main` 上已有 fork 前的文档提交。清空会带走 `.git`，clone 之后要把这个 `origin` 加回去。分支落点按 [决策 0003](../decisions/0003-branch-and-release-conventions.md)：`master` 只镜像上游，产品 `main` 从固定分叉点创建，bootstrap 改动落在第一条 `dev-*`。

## 任务分解

- [ ] 把 `docs/` 拷到目录外。当前根 `README.md` 只是指向 docs 的占位，不放回上游 README 之上。
- [ ] 清空本路径，包括 `.git`。在本路径执行 `git clone https://github.com/deepseek-ai/deepseek-harness.git .`。把 clone 出来的 `origin` 改名为 `upstream`，再 `git remote add origin https://github.com/boomytc/boom-harness.git`，然后 `git fetch upstream`。确认 `639ed01539` 存在且是 `upstream/master` 的祖先；默认分支已前移时仍保留固定分叉点。
- [ ] 执行 `git switch -c main 639ed01539`，确认此时 `main` 的 `HEAD` 为该分叉点。再执行 `git branch -f master upstream/master` 与 `git branch --set-upstream-to=upstream/master master`，让本地镜像等于最新已 fetch 的上游。clone 已经创建了 `master` 时也使用这组命令，不再 `git switch -c master`，不把镜像倒退到分叉点。
- [ ] 从产品 `main` 执行 `git switch -c dev-0.2.0`，作为面向首个目标版本 0.2.0 的开发线。以下 bootstrap 文件改动及其提交都落在该线；版本完成验收后才按决策 0003 合入 `main`。分支目标名不改变根与 bundle 当前的 `0.2.0-rc.2` 版本号，也不在 bootstrap 提前打发布 tag。
- [ ] 把 `docs/` 放回，并把 `docs/README.md` 首段的「源码树尚未从 DeepSeek Harness fork 进来」改为已完成 fork 的说明，记录分叉点 `639ed01539` 与基线版本 `0.2.0-rc.2`，保留行为队列及阅读顺序。不改写上游 README 与 `AGENTS.md` 正文；只在根 `AGENTS.md` 尾部追加「boom-harness 本地约定」小节，指向 [../agent-contract.md](../agent-contract.md)。
- [ ] 写 `UPSTREAM.md`：分叉点、基线版本、分支落点与下节的合并顺序、冲突保留清单，并引用决策 0003。维护者提交这次 fork 时，更新后的 `docs/` 与 `UPSTREAM.md` 在 `dev-0.2.0` 的同一次提交里；`master` 不得含私人提交。
- [ ] 确认 `git status` 不含 `node_modules/`、`lib/`、`*.tsbuildinfo`。`dist` 的忽略范围以规格现状为准。
- [ ] `pnpm install`，然后 `pnpm run build`。
- [ ] 新增 `packages/boom/bundle`（`@deepseek-ai/dsh-boom-bundle`）。它落在 `scripts/check-workspace-constraints.ts` 的 release member 上：`version` 等于根上的 `0.2.0-rc.2`，不设 `"private": true`，`publishConfig.access` 为 `public`，`repository` 为 `git+https://github.com/deepseek-ai/deepseek-harness.git` 且 `directory` 为 `packages/boom/bundle`。`@deepseek-ai/dsh-*` 还要 `type: module`、`main` 为 `lib/index.js`、`types` 为 `lib/types/index.d.ts`、对应的 `exports`，以及 cordis 的 peer 与 dev 范围相同。`files` 必须与 `expectedDshPackageFiles` 算出的列表一致，其中包含这份 patch。`cordis.patch.yml` 的内容是 `[]`。零字节或只有注释的文件不是顶层 YAML 数组，`loadOverlayPatches` 会抛错。`apps/cli` 的依赖写成 `"@deepseek-ai/dsh-boom-bundle": "workspace:*"`。不要新建 `packages/boom/profile`。
- [ ] 改完依赖后再 `pnpm install`，然后 `pnpm constraints`。`pnpm run build` 不跑这道门。不重装时，安装锚点解析不到该包，dump 可能在 stderr 跳过它且退出码仍为 0。
- [ ] 在 `packages/boot/app-boot/src/profile.ts` 的 `PROFILE_TEMPLATES` 增加 `boom`，bundles 依次为 `@deepseek-ai/dsh-base`、`@deepseek-ai/dsh-web-app`、`@deepseek-ai/dsh-boom-bundle`。不改 `web`、`headless`、`sdk`、`sdk-minimal`、`acp`。不改 `desktop`。
- [ ] `DSH_HOME` 指向一个新建的空目录。空字符串和纯空白会被 `resolveDshHome` 当成未设置，仍然落到 `~/.dsh`。用这个目录执行 `pnpm dsh web --no-open`，看到 URL 后停掉。再用同一目录执行 `pnpm dsh boom --no-open`，看到 URL 后停掉。
- [ ] 用同一新建的 `DSH_HOME` 执行 `--dump-config`。`web` 的 bundles 是 `@deepseek-ai/dsh-base` 与 `@deepseek-ai/dsh-web-app`。`boom` 在 `dsh-web-app` 之后有 boom bundle。`headless`、`sdk`、`sdk-minimal`、`acp` 与 `PROFILE_TEMPLATES` 相同。`sdk-minimal` 只有 `@deepseek-ai/dsh-sdk-minimal`。

## UPSTREAM.md 的合并与冲突记录

后续上游合并在干净工作区依次执行以下命令；这是 fork 后的操作约定，不是 bootstrap 时把最新上游并入固定分叉点的步骤。

```sh
git switch main
git fetch upstream
git branch -f master upstream/master
git merge upstream/master
```

解决冲突并完成构建及受影响行为的验收后，再切到当前活跃的 `dev-<major>.<minor>.<patch>` 执行 `git merge main`。进行中的开发不先合入 `main`。镜像的跟踪目标保持 `upstream/master`；上游默认分支改名时，按决策 0003 同步镜像名、跟踪目标与文档。

冲突保留清单分为 fork 专属内容和上游文件中的行为 hunk：

- fork 专属内容：`packages/boom/**`、`docs/**`、`UPSTREAM.md`、`PROFILE_TEMPLATES.boom`、`apps/cli` 里对 `@deepseek-ai/dsh-boom-bundle` 的 `workspace:*` 依赖，以及根 `AGENTS.md` 尾部的「boom-harness 本地约定」段。
- 工具恢复 hunk：按 [工具恢复 design](../design/2026-09-30-tool-effect-recovery.md) 保留 `replay` 声明与日志载荷、核心事件登记、暂存结局写入点，以及 `packages/core/agent-loop/src/index.ts` 的 `resume` 在 closer 之前的结算顺序。
- 审批词汇 hunk：按 [权限规则 design](../design/2026-09-30-permission-rules.md#审批词汇) 保留结果联合、`serviceAsk`、`OUTCOMES`、`APPROVAL_OUTCOMES`、`ui-approval` 与 ACP 选项映射对持久允许、拒绝的处理。
- 其余已实现的上游 hunk：按各条对应 design 合并，包括 [模型映射 design](../design/2026-09-30-model-option-map.md) 的 `llm-deepseek.serialize()` 返回值绑定点，以及 [后续 seam design](../design/2026-09-30-follow-on-seams.md#已调度的模型重试) US-7 在 `resume` 中续跑重试、跳过轮次 closer 的改动。后续每条扩成单独 design 并落地时，把其文件、符号和 design 链接补进 UPSTREAM.md。

只保留已实现且由 design 点名的行为 hunk，逐段与上游合并，不整文件取本地版本。工具恢复与 US-7 虽然都改 `resume`，仍是两条独立改动，按各自 design 保留。

## 风险与依赖

- 构建因本机 Node 版本失败 → 使用仓库 `engines` 要求的 Node（`^22.19.0 || >=24`）与 `pnpm@11.7.0`。
