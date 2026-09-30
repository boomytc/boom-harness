---
title: fork bootstrap
type: plan
status: draft
date: 2026-09-30
related:
  - specs/2026-09-30-fork-baseline.md
  - decisions/0001-fork-dsh-and-absorb-at-seams.md
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

[../specs/2026-09-30-fork-baseline.md](../specs/2026-09-30-fork-baseline.md)。这些文档现在在本目录。本目录已经是 git 仓库，`origin` 是 `https://github.com/boomytc/boom-harness.git`，`main` 上有一次与 dsh 无关的提交。清空会带走 `.git`，clone 之后要把这个 `origin` 加回去。

## 任务分解

- [ ] 把 `docs/` 拷到目录外。当前根 `README.md` 只是指向 docs 的占位，不放回上游 README 之上。
- [ ] 清空本路径，包括 `.git`。在本路径执行 `git clone https://github.com/deepseek-ai/deepseek-harness.git .`。确认在 `master` 上且 `HEAD` 为 `639ed01539`。若默认分支已经前移，用 `git switch -c master 639ed01539`，不要裸检出这个 hash。把 clone 出来的 `origin` 改名为 `upstream`，再 `git remote add origin https://github.com/boomytc/boom-harness.git`。
- [ ] 把 `docs/` 放回。不改写上游 README。写 `UPSTREAM.md`：分叉点、版本 `0.2.0-rc.2`、合并命令 `git fetch upstream && git merge upstream/master`。冲突时保留 `packages/boom/**`、`docs/**`、`PROFILE_TEMPLATES.boom`，以及 `apps/cli` 里对 `@deepseek-ai/dsh-boom-bundle` 的 `workspace:*` 依赖。维护者提交这次 fork 时，`docs/` 与 `UPSTREAM.md` 在同一次提交里。
- [ ] 确认 `git status` 不含 `node_modules/`、`lib/`、`*.tsbuildinfo`。`dist` 的忽略范围以规格现状为准。
- [ ] `pnpm install`，然后 `pnpm run build`。
- [ ] 新增 `packages/boom/bundle`（`@deepseek-ai/dsh-boom-bundle`）。它落在 `scripts/check-workspace-constraints.ts` 的 release member 上：`version` 等于根上的 `0.2.0-rc.2`，不设 `"private": true`，`publishConfig.access` 为 `public`，`repository` 为 `git+https://github.com/deepseek-ai/deepseek-harness.git` 且 `directory` 为 `packages/boom/bundle`。`@deepseek-ai/dsh-*` 还要 `type: module`、`main` 为 `lib/index.js`、`types` 为 `lib/types/index.d.ts`、对应的 `exports`，以及 cordis 的 peer 与 dev 范围相同。`files` 必须与 `expectedDshPackageFiles` 算出的列表一致，其中包含这份 patch。`cordis.patch.yml` 的内容是 `[]`。零字节或只有注释的文件不是顶层 YAML 数组，`loadOverlayPatches` 会抛错。`apps/cli` 的依赖写成 `"@deepseek-ai/dsh-boom-bundle": "workspace:*"`。不要新建 `packages/boom/profile`。
- [ ] 改完依赖后再 `pnpm install`，然后 `pnpm constraints`。`pnpm run build` 不跑这道门。不重装时，安装锚点解析不到该包，dump 可能在 stderr 跳过它且退出码仍为 0。
- [ ] 在 `packages/boot/app-boot/src/profile.ts` 的 `PROFILE_TEMPLATES` 增加 `boom`，bundles 依次为 `@deepseek-ai/dsh-base`、`@deepseek-ai/dsh-web-app`、`@deepseek-ai/dsh-boom-bundle`。不改 `web`、`headless`、`sdk`、`sdk-minimal`、`acp`。不改 `desktop`。
- [ ] `DSH_HOME` 指向一个新建的空目录。空字符串和纯空白会被 `resolveDshHome` 当成未设置，仍然落到 `~/.dsh`。用这个目录执行 `pnpm dsh web --no-open`，看到 URL 后停掉。再用同一目录执行 `pnpm dsh boom --no-open`，看到 URL 后停掉。
- [ ] 用同一新建的 `DSH_HOME` 执行 `--dump-config`。`web` 的 bundles 是 `@deepseek-ai/dsh-base` 与 `@deepseek-ai/dsh-web-app`。`boom` 在 `dsh-web-app` 之后有 boom bundle。`headless`、`sdk`、`sdk-minimal`、`acp` 与 `PROFILE_TEMPLATES` 相同。`sdk-minimal` 只有 `@deepseek-ai/dsh-sdk-minimal`。

## 风险与依赖

- 构建因本机 Node 版本失败 → 使用仓库 `engines` 要求的 Node（`^22.19.0 || >=24`）与 `pnpm@11.7.0`。
