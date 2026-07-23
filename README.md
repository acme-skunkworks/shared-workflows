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

| Workflow                          | Purpose                                                                                       | Secrets                   |
| --------------------------------- | --------------------------------------------------------------------------------------------- | ------------------------- |
| `reusable-claude.yml`             | Interactive `@claude` on issues / PR comments / reviews.                                      | `CLAUDE_CODE_OAUTH_TOKEN` |
| `reusable-claude-code-review.yml` | Automated Claude review on pull requests.                                                     | `CLAUDE_CODE_OAUTH_TOKEN` |
| `reusable-validate-pr-title.yml`  | Enforce a Conventional Commit PR title (the squash-merge subject).                            | — (uses `GITHUB_TOKEN`)   |
| `reusable-validate-commits.yml`   | Enforce Conventional Commits on every commit in a PR's `base..head` range (A-981).            | — (uses `GITHUB_TOKEN`)   |
| `reusable-lint.yml`               | Coarse lint bundle — ESLint, markdownlint, yamllint/actionlint, changelog-validate (Layer 2). | — (uses `GITHUB_TOKEN`)   |
| `reusable-build-test.yml`         | Coarse build/test bundle — build, typecheck, Vitest, ShellCheck, bats (Layer 2).              | — (uses `GITHUB_TOKEN`)   |
| `reusable-pkg-release.yml`        | Build-once → npm OIDC Trusted Publishing → GitHub Packages mirror → tag + release (Layer 2).  | — (OIDC + `GITHUB_TOKEN`) |
| `reusable-load-repo-config.yml`   | Load + allowlist-validate `infrastructure/repo-config.yaml` → job outputs (Layer 2, A-779).   | — (uses `GITHUB_TOKEN`)   |
| `reusable-validate-payload.yml`   | Fan-out payload check — skills bundles and/or `.coderabbit.yaml` (Layer 2).                   | — (uses `GITHUB_TOKEN`)   |
| `reusable-changelog-enrich.yml`   | Post-merge changelog enrich / finalise via `@acme-skunkworks/changelog-core` (Layer 2).       | `ROADRUNNER_PRIVATE_KEY`  |

> **Why `reusable-` prefixes?** It lets a consumer repo (and this repo, which
> dogfoods its own workflows) keep a same-named caller stub — e.g. `claude.yml`
> calling `reusable-claude.yml` — without a filename collision.

## How to consume

Reference a workflow from any repo by its path, using the **floating major tag
`@v1`** (see [Versioning](#versioning)) — new reusable releases land at your
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
| `reusable-validate-commits.yml`   | `contents: read`                                                                                |
| `reusable-lint.yml`               | `contents: read`                                                                                |
| `reusable-build-test.yml`         | `contents: read`                                                                                |
| `reusable-claude.yml`             | `contents: read`, `pull-requests: read`, `issues: read`, `id-token: write`, `actions: read`     |
| `reusable-claude-code-review.yml` | `contents: read`, `pull-requests: write`, `issues: read`, `id-token: write`                     |
| `reusable-pkg-release.yml`        | `contents: write`, `id-token: write`, `issues: write`, `packages: write`, `attestations: write` |
| `reusable-load-repo-config.yml`   | `contents: read`                                                                                |
| `reusable-validate-payload.yml`   | `contents: read`                                                                                |
| `reusable-changelog-enrich.yml`   | `contents: read`, `pull-requests: read`                                                         |

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
    # Requires CLAUDE_CODE_OAUTH_TOKEN in THIS repo's Actions secrets — NOT
    # ANTHROPIC_API_KEY (an empty ANTHROPIC_API_KEY in the log is a red herring).
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
    # Requires CLAUDE_CODE_OAUTH_TOKEN in THIS repo's Actions secrets — NOT
    # ANTHROPIC_API_KEY (an empty ANTHROPIC_API_KEY in the log is a red herring).
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

### `reusable-validate-commits.yml`

```yaml
# .github/workflows/validate-commits.yml
name: Validate commits

on:
  pull_request:
    types: [opened, synchronize, reopened]

# Required — see "Required caller permissions" above. The check checks out the
# repo to read the commit range, so it needs contents:read. Do NOT pass
# secrets: inherit — this reusable declares no secrets (same trust model as
# reusable-validate-pr-title.yml).
permissions:
  contents: read

jobs:
  commits: # ← keep this job id stable across the estate (see below)
    uses: acme-skunkworks/shared-workflows/.github/workflows/reusable-validate-commits.yml@v1
```

**Required-check context (A-400 / A-405).** The reusable job is named
`Validate commits are Conventional Commits`, and a reusable-workflow check
renders as `<caller-job-id> / <job-name>`. Use the caller job id **`commits`**
so every repo reports the same context —
`commits / Validate commits are Conventional Commits` — which a single
branch-protection rule can pin estate-wide.

**Ruleset.** Floats [`@acme-skunkworks/commitlint-config`](https://www.npmjs.com/package/@acme-skunkworks/commitlint-config)
to latest on every run (public npm, no registry auth) and lints
`github.event.pull_request.base.sha..head.sha`. Types live in that package —
they are intentionally **not** a workflow input, so this gate and the local
husky pre-push hook cannot drift. Merge / revert / fixup / squash subjects rely
on commitlint's `defaultIgnores`. Input: `node-version` (default `"22"`).

### `reusable-lint.yml`

```yaml
# .github/workflows/lint.yml
name: Lint

on:
  pull_request:

# Required — see "Required caller permissions" above. The bundle only checks
# out and lints, so contents:read is enough (the org default already grants it,
# but a permissions: block resets every other scope to none — keep this list in
# step with any sibling caller job that needs more).
permissions:
  contents: read

jobs:
  lint: # ← keep this job id stable (renders as `lint / Lint`)
    uses: acme-skunkworks/shared-workflows/.github/workflows/reusable-lint.yml@v1
```

Each lane has a boolean opt-out (`eslint`, `markdown`, `yaml`, `actionlint`,
`changelog` — all default `true`), alongside pass-throughs `node-version-file`,
`eslint-args`, `markdown-globs`, `yaml-paths`, `yamllint-version`,
`actionlint-version` and `changelog-script`. Disabling **every** lane is a hard
error, not a silent green pass — the run fails fast before checkout (A-445).

### `reusable-build-test.yml`

```yaml
# .github/workflows/build-test.yml
name: Build & Test

on:
  pull_request:

# Required — see "Required caller permissions" above. Build and test only need
# contents:read; a permissions: block resets every other scope to none, so keep
# this list in step with any sibling caller job that needs more.
permissions:
  contents: read

jobs:
  build-test: # ← keep this job id stable (renders as `build-test / Build & Test`)
    uses: acme-skunkworks/shared-workflows/.github/workflows/reusable-build-test.yml@v1
```

Each lane has a boolean opt-out — `build`, `typecheck`, `test` and `shellcheck`
default `true`; `bats` and `coverage` default `false` — alongside pass-throughs
`node-version-file`, `build-script`, `tsconfig`, `test-args`, `shellcheck-paths`,
`shellcheck-severity` and `bats-paths`. As with the lint bundle, disabling
**every** lane fails fast rather than passing green (A-445).

### `reusable-pkg-release.yml`

```yaml
# .github/workflows/pkg-release.yml
name: Package release

# NO workflow_dispatch — a dispatched run satisfies the same npm OIDC subject as
# a legitimate post-merge push and could publish a poisoned tarball (A-326).
on:
  push:
    branches: [main]

# Concurrency lives HERE (the caller), not in the reusable: cancel-in-progress:
# false queues releases rather than cancelling an in-flight publish.
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: false

# Required — see "Required caller permissions" above. The caller grants the
# SUPERSET of the reusable's per-job scopes; each publish job narrows from here.
permissions:
  contents: write
  id-token: write
  issues: write
  packages: write
  attestations: write

jobs:
  release:
    uses: acme-skunkworks/shared-workflows/.github/workflows/reusable-pkg-release.yml@v1
    with:
      npm-scope: "@acme-skunkworks"
```

**Consumer prerequisite:** define a branch-protected **`npm-release`**
environment restricted to the release branch. A reusable workflow's
`environment:` resolves in the caller repo; if it is absent GitHub silently
auto-creates it **unprotected**, losing the ref gate (A-326). There is **no**
`secrets:` block — the npm leg publishes via OIDC Trusted Publishing (no
`NPM_TOKEN`) and the Packages/tag/release legs use the automatic `GITHUB_TOKEN`.
`npm-scope` is the only required input; a build-less package (config- or
bundle-only, no `build` script) passes `build: false`. Other notable inputs:
`node-version-file`, `publish-github-packages`, `changelog-dir` and `tag-prefix`.

### `reusable-load-repo-config.yml`

Loads and allowlist-validates the caller's `infrastructure/repo-config.yaml`
into job outputs (A-779). Prefer this over a local `.github/actions/load-repo-config`
copy — the composite is SHA-pinned inside the reusable; consumers float `@v1`.

```yaml
# .github/workflows/ci.yml (excerpt)
jobs:
  config:
    uses: acme-skunkworks/shared-workflows/.github/workflows/reusable-load-repo-config.yml@v1
    permissions:
      contents: read

  lint:
    needs: config
    uses: acme-skunkworks/shared-workflows/.github/workflows/reusable-lint.yml@v1
    with:
      node-version-file: ${{ needs.config.outputs.node_version_file }}
```

Outputs: `default_branch`, `node_version_file`, `npm_registry_url`, `npm_scope`,
`github_packages_registry_url`. Callers map only the subset their downstream
jobs need.

### `reusable-validate-payload.yml`

Fan-out payload check (A-738). Callers declare what must be present — never
auto-detected. The job name is `Validate fanned payload`; callers **must** use
the job id `validate-payload` so the ruleset-pinnable context is
`validate-payload / Validate fanned payload`.

The caller stub is **repo-owned** (A-780): place it once at onboarding (copy
below), pin `@v1` (or let Dependabot bump a SHA), and make the status context
required on `main`. The release-orchestrator does **not** vendor or refresh this
file.

```yaml
# .github/workflows/validate-payload.yml
name: Validate fanned payload

on:
  pull_request:
    branches: [main]

permissions:
  contents: read

jobs:
  validate-payload:
    uses: acme-skunkworks/shared-workflows/.github/workflows/reusable-validate-payload.yml@v1
    with:
      skills: true # require well-formed .claude/skills/** (+ .agents/skills/**)
      coderabbit: true # require a parseable .coderabbit.yaml
```

### `reusable-changelog-enrich.yml`

Post-merge fill of dated `changelog/` entries via
[`@acme-skunkworks/changelog-core`](https://www.npmjs.com/package/@acme-skunkworks/changelog-core)
(A-793 / A-821). Resolves the just-merged PR from the push SHA, runs `enrich`
and (optionally) `finalise`, then pushes **only** `changelog/**` as
`road-runner-bot[bot]`. Write-back mints a repo-scoped installation token from
org var `ROADRUNNER_CLIENT_ID` + secret `ROADRUNNER_PRIVATE_KEY` — Actions
cannot be a Trunk bypass actor on this org (ADR 0004 / A-794), so the bypass
actor is road-runner-bot and the path limit is workflow discipline (stage only
`changelog/**`). Callers **must** pass `secrets: inherit`.

`@v1` includes the road-runner-bot write-back (A-821) as of **v1.5.0**, so pin
callers to `@v1` like any other reusable workflow — no SHA pin is needed. (Early
adopters pinned the A-821 merge SHA directly while `@v1` still predated it, before
v1.5.0 moved the floating major onto that commit.)

**Consumer prerequisites:**

- Add `@acme-skunkworks/changelog-core` as a devDependency so
  `pnpm exec changelog-core` resolves from the lockfile.
- Org secret `ROADRUNNER_PRIVATE_KEY` and org var `ROADRUNNER_CLIENT_ID` must
  be visible to the caller (grant selected access — public repos cannot see
  `visibility: private` vars/secrets).
- Protected `main` must allow road-runner-bot as a Trunk bypass actor
  (ADR 0004).

#### Deploy targets (`mode: enrich`)

```yaml
# .github/workflows/changelog-enrich.yml
name: Changelog enrich

on:
  push:
    branches: [main]

concurrency:
  group: changelog-enrich-${{ github.ref }}
  cancel-in-progress: false

permissions:
  contents: read
  pull-requests: read

jobs:
  changelog-enrich:
    uses: acme-skunkworks/shared-workflows/.github/workflows/reusable-changelog-enrich.yml@v1
    with:
      mode: enrich
    secrets: inherit
```

#### npm targets (`mode: finalise`)

Add a sibling job alongside the `pkg-release.yml` caller so it inherits that
workflow's concurrency group. `finalise` enriches the associated PR on every
merge and stamps `version` only when `package.json`'s version has no matching
git tag (a release-please cut):

```yaml
# Extra job in .github/workflows/pkg-release.yml (alongside `release:`)
changelog-enrich:
  uses: acme-skunkworks/shared-workflows/.github/workflows/reusable-changelog-enrich.yml@v1
  with:
    mode: finalise
  permissions:
    contents: read
    pull-requests: read
  secrets: inherit
```

`pull-requests: read` is required for the commits→pulls resolution API;
`contents: read` covers checkout + resolve — write to `main` uses the App
token, not `GITHUB_TOKEN`. A-793's "no publish scopes" constraint (no
`id-token` / `packages` / `attestations`) still holds. Other inputs:
`node-version-file` (default `.nvmrc`) and `changelog-dir` (default `changelog`).

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

Releases here are tagged (`vX.Y.Z`) automatically by release-please (driven by
the private release-orchestrator as a `kind: deploy` target), and the floating
**`v1`** tag tracks the latest `v1.x`. The full history lives in [`changelog/`](changelog/)
and on the [releases page](https://github.com/acme-skunkworks/shared-workflows/releases),
each tag with a matching GitHub release. Prefer `@v1`; if you need to pin an exact
commit, use the SHA a specific `vX.Y.Z` tag points at.

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
