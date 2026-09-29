# release-flow

[English](README.md) | 简体中文

共享的发版 / CI 流程（reusable workflows）——browsa、pi-feishu-notify、agent-bridge 共用一份实现，
per-repo 只留薄壳配置。设计目标：**一处证实、处处生效**；差异收敛到约定，而非每仓一份拷贝。

## 约定（uniform，无旋钮）

- 版本号 tag 统一 `v` 前缀（`vX.Y.Z`，annotated）；历史裸 tag 也能参与 previous-tag 计算
- 版本号 PR 统一 `--merge --delete-branch` 自动合并；bump 提交即版本文件（`git add -u`，
  每仓靠自己的 `postversion` 钩子同步附属文件——如 browsa 的 manifest.json）
- 发版顺手快进 `dev` 分支（非 fast-forward 仅 warning）
- 产物按约定自动分流：package.json 带 `package` script → `npm run package` 打 zip 挂
  GitHub Release；否则 → `npm publish`（已发布版本幂等跳过、低于 npm latest 拒绝）
- 验证统一 `npm test` + `npm run typecheck --if-present` + `npm run check-compat --if-present`，
  按内容树哈希复用（`.verify-pass` / `verify-<os>-node20-<treehash>`，PR 与发版同一缓存空间）
- 版本号 PR 安全阀：diff 只允许 `package.json` / `package-lock.json` / `manifest.json`
- PAT 双模式：`secrets.RELEASE_PAT` 已配 → bump 带 `[skip ci]` + PAT 建 PR + `--admin`
  直合（版本 PR 零测试零批准）；未配 → 回落 CI + 管理员批准一次（GITHUB_TOKEN 建的 PR
  其 CI 停在 action_required，防递归策略）
- 幂等自愈：tag+release 已在的重跑照常补齐版本号 PR（空差异=版本号已进 main 则关闭收尾）

## 各仓薄壳

### ci.yml

```yaml
name: CI
on:
  pull_request: { branches: [main] }
  push: { branches: [main] }
jobs:
  test:
    name: Test + Compat Check   # ← 唯一 per-repo 项：main ruleset 按此名匹配
    uses: xiaohuzai/release-flow/.github/workflows/ci.yml@v1
    with:
      skip_paths: '^(manifest\.json|lib/|test/|package\.json)'   # 可选：仅这些源码路径改动才跑测试
```

### release.yml

```yaml
name: Release
on:
  workflow_dispatch:
    inputs:
      version:
        description: '版本号（如 0.41.0），留空用 package.json 当前值'
        required: false
        default: ''
permissions:
  contents: write
  pull-requests: write
jobs:
  release:
    uses: xiaohuzai/release-flow/.github/workflows/release.yml@v1
    with:
      version: ${{ inputs.version }}
      skip_paths: ''              # 可选，同 ci.yml
    secrets: inherit              # RELEASE_PAT / NPM_TOKEN
```

## 仓库侧一次性配置（ruleset / settings）

- main ruleset：required check 名 = 薄壳 job 的 `name`；注入 **Repository admin bypass**
  （PAT 模式 `--admin` 直合所需）
- Settings → Actions → Workflow permissions：勾选 **Allow GitHub Actions to create and
  approve pull requests** + **Allow auto-merge**（无 PAT 兜底路径所需）
- secret `RELEASE_PAT`（可选，免测免批直通）、`NPM_TOKEN`（npm 发布仓）

## 升级

共享仓改动合入后把 `v1` tag 移动指向新提交（`git tag -f v1 && git push -f origin v1`），
各仓下次运行自动吃到新版；行为大改时先在一个仓真实发版验证再移 tag。
