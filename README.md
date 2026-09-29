# release-flow

English | [简体中文](README.zh-CN.md)

Shared release / CI pipelines as GitHub reusable workflows — browsa, pi-feishu-notify and
agent-bridge all run one shared implementation, and each repo keeps only a thin shell of
configuration. Design goal: **prove it once, it works everywhere**; differences converge
into conventions instead of a per-repo copy.

## Conventions (uniform, no knobs)

- Version tags uniformly use the `v` prefix (`vX.Y.Z`, annotated); historical bare tags
  still participate in previous-tag computation
- Version bump PRs are always auto-merged with `--merge --delete-branch`; the bump commit
  is just the version files (`git add -u`) — each repo's own `postversion` hook keeps
  satellite files in sync (e.g. browsa's `manifest.json`)
- A release fast-forwards the `dev` branch along the way (a non-fast-forward only
  produces a warning)
- Artifacts are routed automatically by convention: if `package.json` has a `package`
  script → `npm run package` builds a zip attached to the GitHub Release; otherwise →
  `npm publish` (already-published versions are skipped idempotently; versions lower
  than npm's `latest` are rejected)
- Verification is unified: `npm test` + `npm run typecheck --if-present` +
  `npm run check-compat --if-present`, reused by content-tree hash (`.verify-pass` /
  `verify-<os>-node20-<treehash>`; PRs and releases share the same cache space)
- Version bump PR safety valve: the diff may only touch `package.json` /
  `package-lock.json` / `manifest.json`
- PAT dual mode: when `secrets.RELEASE_PAT` is configured → the bump commit carries
  `[skip ci]`, the PR is opened with the PAT, and `--admin` merges it directly (version
  PRs: zero tests, zero approvals); when not configured → fall back to CI + one admin
  approval (PRs created with `GITHUB_TOKEN` have their CI held at `action_required` —
  the anti-recursion policy)
- Idempotent self-healing: a re-run where the tag + release already exist still
  completes the version bump PR as usual (an empty diff means the version already
  reached main, so the PR is closed to wrap up)

## Per-repo thin shells

### ci.yml

```yaml
name: CI
on:
  pull_request: { branches: [main] }
  push: { branches: [main] }
jobs:
  test:
    name: Test + Compat Check   # ← the only per-repo item: the main ruleset matches on this name
    uses: xiaohuzai/release-flow/.github/workflows/ci.yml@v1
    with:
      skip_paths: '^(manifest\.json|lib/|test/|package\.json)'   # optional: run tests only when one of these source paths changes
```

### release.yml

```yaml
name: Release
on:
  workflow_dispatch:
    inputs:
      version:
        description: 'Version (e.g. 0.41.0); leave empty to use the current package.json value'
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
      skip_paths: ''              # optional, same as ci.yml
    secrets: inherit              # RELEASE_PAT / NPM_TOKEN
```

## One-time repo-side configuration (ruleset / settings)

- main ruleset: the required check name = the shell job's `name`; inject the
  **Repository admin bypass** (required for PAT-mode `--admin` direct merges)
- Settings → Actions → Workflow permissions: check **Allow GitHub Actions to create and
  approve pull requests** + **Allow auto-merge** (required by the no-PAT fallback path)
- Secrets: `RELEASE_PAT` (optional; the test-free, approval-free fast lane) and
  `NPM_TOKEN` (for npm-publishing repos)

## Upgrading

After a change to the shared repo is merged, move the `v1` tag to point at the new
commit (`git tag -f v1 && git push -f origin v1`); every repo picks up the new version
on its next run. For major behavior changes, validate with a real release in one repo
before moving the tag.
