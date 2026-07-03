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

Reference a workflow from any repo by its path, pinned to the **floating major
tag `@v1`** (see [Versioning](#versioning)) — new reusable releases land at your
caller automatically, with no per-repo SHA bump. Triggers live in your caller;
the reusable workflow holds the job.

### Required caller permissions

A called reusable workflow's job **cannot be granted more than the caller's
`GITHUB_TOKEN` holds**. The estate default is `default_workflow_permissions:
read`, so a caller that declares **no** top-level `permissions:` block hands the
reusable a read-only token — the reusable's job then requests scopes it can't be
granted and the run dies at compile time with a **`startup_failure`**: no jobs,
no logs, and the REST API does not expose the reason (`gh run view` only offers a
generic "workflow file issue" hint; annotations 404). **Read the one-line error
on the Actions UI run page.** This is the single most common adoption trap
(A-621) — and the usual reason a consumer's Claude review 'never posts'.

So every caller stub below declares the top-level `permissions:` block its
reusable needs (a job-level block on the calling job — as the
`reusable-pkg-release.yml` header shows — works identically). Copy the whole
stub, not just the `uses:` line. Note that a `permissions:` block resets every
scope you _don't_ list to `none`, so it must name **every** scope the reusable's
job requests:

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

### Required secret: `CLAUDE_CODE_OAUTH_TOKEN`

The two Claude workflows authenticate via **one** secret,
`CLAUDE_CODE_OAUTH_TOKEN`. It is **not** `ANTHROPIC_API_KEY` — claude-code-action
prints an empty `ANTHROPIC_API_KEY` as an unused alt-auth input, which is a red
herring, not the cause of a failed run (A-646).

- **There is no org-level secret.** Each consuming repo defines its **own
  repository Actions secret** named `CLAUDE_CODE_OAUTH_TOKEN`; the caller stub
  hands it to the reusable via `secrets: inherit`.
- **Missing / empty on a normal run** now **fails fast** with a clear guard step
  error, instead of an opaque failure deep inside claude-code-action's OIDC
  exchange with no review posted.
- **Dependabot PRs are skipped by design.** A Dependabot-triggered run reads the
  Dependabot secret store, not Actions secrets, so the token is empty there — the
  review job skips (neutral) rather than failing. This is expected, not a
  misconfigured secret.

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
# fails at startup (startup_failure) under default_workflow_permissions: read.
permissions:
  contents: read
  pull-requests: read
  issues: read
  id-token: write
  actions: read

jobs:
  claude:
    uses: acme-skunkworks/shared-workflows/.github/workflows/reusable-claude.yml@v1
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

# Concurrency lives HERE (the caller), not in the reusable: a reusable runs as
# this job, so a top-level `concurrency:` inside the reusable resolves to the
# same group and deadlock-cancels the run (A-621). This reusable declares none.
concurrency:
  group: ${{ github.workflow }}-${{ github.event.pull_request.number }}
  cancel-in-progress: true

# Required — see "Required caller permissions" above. Omit this and the review
# never posts: the run fails at startup (startup_failure) under default_workflow_permissions: read.
permissions:
  contents: read
  pull-requests: write
  issues: read
  id-token: write

jobs:
  claude-review:
    uses: acme-skunkworks/shared-workflows/.github/workflows/reusable-claude-code-review.yml@v1
    secrets: inherit
```

Inputs: `timeout_minutes` (default `30`), `track_progress` (default `true`),
`use_sticky_comment` (default `true`). Draft PRs, `release-please--*` branches and
Dependabot PRs are skipped automatically (see [Required secret](#required-secret-claude_code_oauth_token)).

### `reusable-validate-pr-title.yml`

```yaml
# .github/workflows/validate-pr-title.yml
name: Validate PR title

on:
  pull_request:
    types: [opened, edited, synchronize, reopened] # `edited` re-runs on title fixes

# Required — see "Required caller permissions" above. The check reads the PR
# title, so it needs pull-requests:read (NOT contents:read); the wrong scope
# fails the run at startup (startup_failure) under default_workflow_permissions: read.
permissions:
  pull-requests: read

jobs:
  pr-title: # ← keep this job id stable across the estate (see below)
    uses: acme-skunkworks/shared-workflows/.github/workflows/reusable-validate-pr-title.yml@v1
```

**Required-check context (A-400 / A-405).** The reusable job is named
`Validate PR title is a Conventional Commit`, and a reusable-workflow check
renders as `<caller-job-id> / <job-name>`. Use the caller job id **`pr-title`**
so every repo reports the same context —
`pr-title / Validate PR title is a Conventional Commit` — which a single
branch-protection rule can pin estate-wide. Input: `types` (newline-separated
allow-list; defaults to the estate's Conventional Commit types).

## Versioning

Pin every reusable-workflow caller to the **floating major tag `@v1`**:

```yaml
uses: acme-skunkworks/shared-workflows/.github/workflows/reusable-claude.yml@v1
```

`v1` moves forward onto each new `vX.Y.Z` release, so non-breaking fixes and
features arrive at your caller with no SHA bump. GitHub permits a **tag** ref for
a reusable-workflow caller even under `sha_pinning_required` (the reusable-workflow
exception) — so this stays policy-compliant. Note this is the **one** place a
floating tag is allowed: true `uses:` **actions** (including `claude-code-action`
_inside_ the reusables) must still be SHA-pinned; that pin lives centrally in the
reusable and Dependabot bumps it there, so consumers never see it.

A future breaking release ships as `v2` and does **not** move `v1`; opt in
deliberately by bumping your caller to `@v2`.

Still add the `github-actions` ecosystem to each consumer's `.github/dependabot.yml`
— it keeps the repo's own third-party actions current and will surface a future
`v2`. It no longer churns the `@v1` reusable refs every release (that churn is
exactly what the floating tag removes):

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

Releases here are tagged (`vX.Y.Z`), and the floating **`v1`** tag tracks the
latest `v1.x`. The current release is **`v1.0.2`** — the full history
(`v0.1.0`–`v1.0.2`) is in [`changelog/`](changelog/), and each tag has a matching
GitHub release. Prefer `@v1`; if you need to pin an exact commit, use the SHA a
specific `vX.Y.Z` tag points at.

## Contributing

`ci.yml` lints the workflow YAML (actionlint + yamllint), the Markdown, and the
PR title.

> **Why doesn't this repo consume its own reusable workflows via `./`?** The org
> enforces `sha_pinning_required`, which rejects local `uses: ./…` refs
> (`startup_failure`), and a cross-repo self-reference is circular before the
> first tagged SHA exists. So `ci.yml`'s PR-title check is **inline** rather than
> a caller of `reusable-validate-pr-title.yml`. Consumers reference the reusable
> workflows by cross-repo tag `@v1` (compliant — the reusable-workflow tag
> exception) — see [How to consume](#how-to-consume).

See [CLAUDE.md](CLAUDE.md) for conventions and local testing with
[`act`](https://github.com/nektos/act).

## Licence

[MIT](LICENSE) © 2025 Rob Easthope
