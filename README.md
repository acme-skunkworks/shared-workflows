# shared-workflows

> Reusable [GitHub Actions workflows](https://docs.github.com/en/actions/using-workflows/reusing-workflows)
> shared across the Acme Skunkworks estate.

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)

One authoritative copy of each common workflow, edited once and consumed
everywhere via `workflow_call`. Instead of copy-pasting the same `claude.yml`
into a dozen repos, each repo keeps a small **caller stub** that points here.

**Further reading:** [ADR 0001](docs/adr/0001-shared-ci-architecture-for-npm-packages.md)
(the layered architecture) and [the `GO/NO GO` gate](docs/go-no-go-gate.md) (the per-repo
release gate pattern).

## Available workflows

| Workflow                          | Purpose                                                            | Secrets                   |
| --------------------------------- | ------------------------------------------------------------------ | ------------------------- |
| `reusable-claude.yml`             | Interactive `@claude` on issues / PR comments / reviews.           | `CLAUDE_CODE_OAUTH_TOKEN` |
| `reusable-claude-code-review.yml` | Automated Claude review on pull requests.                          | `CLAUDE_CODE_OAUTH_TOKEN` |
| `reusable-validate-pr-title.yml`  | Enforce a Conventional Commit PR title (the squash-merge subject). | — (uses `GITHUB_TOKEN`)   |

> **Why `reusable-` prefixes?** It lets a consumer repo (and this repo, which
> dogfoods its own workflows) keep a same-named caller stub — e.g. `claude.yml`
> calling `reusable-claude.yml` — without a filename collision.

## How to consume

Reference a workflow from any repo by its path, **pinned to a commit SHA**, and
let Dependabot keep the SHA current (see [Versioning](#versioning)). Triggers
live in your caller; the reusable workflow holds the job.

### `reusable-claude.yml`

```yaml
# .github/workflows/claude.yml
name: Claude Code

on:
  issue_comment:
    types: [created]
  pull_request_review_comment:
    types: [created]
  issues:
    types: [opened, assigned]
  pull_request_review:
    types: [submitted]

jobs:
  claude:
    uses: acme-skunkworks/shared-workflows/.github/workflows/reusable-claude.yml@<sha> # v0.1.0
    secrets: inherit
```

The maintainer-only author-association gate (OWNER / MEMBER / COLLABORATOR) is
built in. Inputs: `timeout_minutes` (default `30`), `claude_args` (default empty).

### `reusable-claude-code-review.yml`

```yaml
# .github/workflows/claude-code-review.yml
name: Claude Code Review

on:
  pull_request:
    types: [opened, synchronize, ready_for_review, reopened]
    # Keep paths-ignore HERE — paths filters aren't valid on workflow_call, and
    # editing the claude workflow files can't be reviewed by the bot anyway
    # (the app-token exchange requires the file to match the default branch).
    paths-ignore:
      - ".github/workflows/claude.yml"
      - ".github/workflows/claude-code-review.yml"

jobs:
  claude-review:
    uses: acme-skunkworks/shared-workflows/.github/workflows/reusable-claude-code-review.yml@<sha> # v0.1.0
    secrets: inherit
```

Inputs: `timeout_minutes` (default `30`), `track_progress` (default `true`),
`use_sticky_comment` (default `true`). Draft PRs and `release-please--*` branches
are skipped automatically.

### `reusable-validate-pr-title.yml`

```yaml
# .github/workflows/validate-pr-title.yml
name: Validate PR title

on:
  pull_request:
    types: [opened, edited, synchronize, reopened] # `edited` re-runs on title fixes

permissions:
  contents: read

jobs:
  pr-title: # ← keep this job id stable across the estate (see below)
    uses: acme-skunkworks/shared-workflows/.github/workflows/reusable-validate-pr-title.yml@<sha> # v0.1.0
```

**Required-check context (A-400 / A-405).** The reusable job is named
`Validate PR title is a Conventional Commit`, and a reusable-workflow check
renders as `<caller-job-id> / <job-name>`. Use the caller job id **`pr-title`**
so every repo reports the same context —
`pr-title / Validate PR title is a Conventional Commit` — which a single
branch-protection rule can pin estate-wide. Input: `types` (newline-separated
allow-list; defaults to the estate's Conventional Commit types).

## Versioning

Pin every `uses:` to a **commit SHA**, with the human-readable tag in a trailing
comment, and let Dependabot bump it — exactly how the estate pins third-party
actions:

```yaml
uses: acme-skunkworks/shared-workflows/.github/workflows/reusable-claude.yml@1a2b3c4… # v0.1.0
```

Add the `github-actions` ecosystem to each consumer's `.github/dependabot.yml`
(it covers reusable-workflow refs):

```yaml
version: 2
updates:
  - package-ecosystem: github-actions
    directory: "/"
    schedule:
      interval: weekly
    groups:
      actions:
        patterns: ["*"]
    commit-message:
      prefix: "ci"
      include: scope
```

Releases here are tagged (`vX.Y.Z`); use the SHA the tag points at.

## Contributing

`ci.yml` lints the workflow YAML (actionlint + yamllint), the Markdown, and the
PR title.

> **Why doesn't this repo consume its own reusable workflows via `./`?** The org
> enforces `sha_pinning_required`, which rejects local `uses: ./…` refs
> (`startup_failure`), and a cross-repo self-reference is circular before the
> first tagged SHA exists. So `ci.yml`'s PR-title check is **inline** rather than
> a caller of `reusable-validate-pr-title.yml`. Consumers reference the reusable
> workflows by cross-repo `@<sha>` (compliant) — see [How to consume](#how-to-consume).

See [CLAUDE.md](CLAUDE.md) for conventions and local testing with
[`act`](https://github.com/nektos/act).

## Licence

[MIT](LICENSE) © 2025 Rob Easthope
