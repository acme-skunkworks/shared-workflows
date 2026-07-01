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

### Required caller permissions

A called reusable workflow's job **cannot be granted more than the caller's
`GITHUB_TOKEN` holds**. The estate default is `default_workflow_permissions:
read`, so a caller that declares **no** top-level `permissions:` block hands the
reusable a read-only token — the reusable's job then requests scopes it can't be
granted and the run dies at compile time with a **`startup_failure`**: no jobs,
no logs, and the REST API does not expose the reason (`gh run view` only offers a
generic "workflow file issue" hint; annotations 404). **Read the one-line error
on the Actions UI run page.** This is the single most common adoption trap
(A-621) — and the usual reason a consumer's Claude review "never posts".

So every caller stub below declares the top-level `permissions:` block its
reusable needs. Copy the whole stub, not just the `uses:` line. Note that a
`permissions:` block resets every scope you _don't_ list to `none`, so it must
name **every** scope the reusable's job requests:

| Reusable workflow                 | Required caller `permissions:`                                                                  |
| --------------------------------- | ----------------------------------------------------------------------------------------------- |
| `reusable-validate-pr-title.yml`  | `pull-requests: read`                                                                           |
| `reusable-lint.yml`               | `contents: read`                                                                                |
| `reusable-build-test.yml`         | `contents: read`                                                                                |
| `reusable-claude.yml`             | `contents: read`, `pull-requests: read`, `issues: read`, `id-token: write`, `actions: read`     |
| `reusable-claude-code-review.yml` | `contents: read`, `pull-requests: write`, `issues: read`, `id-token: write`                     |
| `reusable-pkg-release.yml`        | `contents: write`, `id-token: write`, `issues: write`, `packages: write`, `attestations: write` |

`id-token: write` is **required**, not optional, on both Claude workflows —
claude-code-action exchanges an OIDC token for its short-lived GitHub token, so
dropping it fails the run outright.

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

# Required — see "Required caller permissions" above. Omit this and the run
# startup_failures under default_workflow_permissions: read.
permissions:
  contents: read
  pull-requests: read
  issues: read
  id-token: write
  actions: read

jobs:
  claude:
    uses: acme-skunkworks/shared-workflows/.github/workflows/reusable-claude.yml@<sha> # v1.0.0
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

# Required — see "Required caller permissions" above. Omit this and the review
# never posts: the run startup_failures under default_workflow_permissions: read.
permissions:
  contents: read
  pull-requests: write
  issues: read
  id-token: write

jobs:
  claude-review:
    uses: acme-skunkworks/shared-workflows/.github/workflows/reusable-claude-code-review.yml@<sha> # v1.0.0
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

# Required — see "Required caller permissions" above. The check reads the PR
# title, so it needs pull-requests:read (NOT contents:read); the wrong scope
# startup_failures under default_workflow_permissions: read.
permissions:
  pull-requests: read

jobs:
  pr-title: # ← keep this job id stable across the estate (see below)
    uses: acme-skunkworks/shared-workflows/.github/workflows/reusable-validate-pr-title.yml@<sha> # v1.0.0
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
uses: acme-skunkworks/shared-workflows/.github/workflows/reusable-claude.yml@1a2b3c4… # v1.0.0
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

Releases here are tagged (`vX.Y.Z`); use the SHA the tag points at. The current
release is **`v1.0.0`** — the full history (`v0.1.0`–`v1.0.0`) is in
[`changelog/`](changelog/), and each tag has a matching GitHub release.

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
