# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repo

Home for the Acme Skunkworks estate's **reusable GitHub Actions workflows**
(`workflow_call`). One authoritative copy of each common workflow lives here;
consumer repos keep thin caller stubs that point at it (SHA-pinned + Dependabot).
This replaces the previous copy-paste-per-repo approach.

The product is the `reusable-*.yml` files under `.github/workflows/`. Everything
else (this repo's own `ci.yml`, the caller stubs, lint configs) exists to lint
and dogfood them.

## British English

Write all prose in British English — code comments, documentation, commit
messages, PR titles/bodies, and any user-facing strings.

- **Spelling:** use British forms — _colour_, _behaviour_, _organisation_,
  _licence_ (noun), _analyse_, _recognise_.
- **Grammar/punctuation:** _whilst_/_amongst_ acceptable; single quotes for
  quoting where appropriate.
- **Scope vs. identifiers:** prose only. Do **not** apply it to identifiers,
  action inputs, or third-party API field names that mirror upstream spelling
  (e.g. `track_progress`, `paths-ignore`).

## Layout

```
.github/
├── actions/                             # PRODUCT: Layer-1 composite actions (pick-and-mix)
│   ├── setup-project/                   #   pnpm + Node-from-.nvmrc + store cache
│   ├── eslint/ lint-markdown/ lint-yaml/    #   lint mix-ins
│   ├── build/ typecheck/ test-vitest/ test-bats/  #   build/test mix-ins
│   ├── shellcheck/ changelog-validate/  #   infra/changelog mix-ins
│   └── README.md                        #   the action catalogue + rationale
├── workflows/
│   ├── reusable-claude.yml              # PRODUCT: interactive @claude
│   ├── reusable-claude-code-review.yml  # PRODUCT: PR review
│   ├── reusable-validate-pr-title.yml   # PRODUCT: conventional PR title
│   ├── reusable-lint.yml                # PRODUCT: coarse lint bundle (Layer 2)
│   ├── reusable-build-test.yml          # PRODUCT: coarse build/test bundle (Layer 2)
│   ├── reusable-release.yml             # PRODUCT: build-once → npm OIDC + Packages mirror (Layer 2)
│   ├── claude.yml                       # self-host: inline @claude on THIS repo
│   ├── claude-code-review.yml           # self-host: inline PR review on THIS repo
│   └── ci.yml                           # self-CI: actionlint + yamllint + markdownlint + inline PR-title
└── dependabot.yml                       # weekly grouped github-actions + npm bumps
```

## How the reusable workflows work

- Each `reusable-*.yml` is `on: workflow_call`. The **triggering events stay in
  the caller stub**; the original event payload is still available to the
  reusable workflow via the `github` context, so author-association gates,
  `github.head_ref` skips, and `github.event.pull_request.number` all evaluate
  exactly as they did inline.
- **Secrets** flow via `secrets: inherit` from the caller. `GITHUB_TOKEN` is
  available automatically and need not be declared.
- **Permissions** are declared in the reusable job and are capped by the
  caller/org token settings. `id-token: write` on the Claude workflows is
  **required** for claude-code-action's OIDC token exchange — do not drop it
  (A-329).
- **`paths-ignore`** cannot be set on `workflow_call`, so the Claude review's
  paths-ignore lives in the caller stub.
- The `reusable-` prefix avoids a filename collision with the same-named caller
  stub in a consumer.
- **A Layer-2 workflow references its sibling Layer-1 actions by full cross-repo
  path, SHA-pinned** (`acme-skunkworks/shared-workflows/.github/actions/…@<sha>`),
  never `./.github/actions/…`: inside a `workflow_call` job a `./` path resolves
  against the **caller's** workspace, not this repo
  ([community #18601](https://github.com/orgs/community/discussions/18601)), and
  `sha_pinning_required` rejects an unpinned local ref. The full-path form is also
  what populates `github.action_path` so `lint-yaml`'s `.yamllint.yml` injection
  resolves. `reusable-lint.yml` pins to the actions' commit (no release tag exists
  yet); the release process maintains it once tags land.

## The release workflow (`reusable-release.yml`, A-417)

`reusable-release.yml` ports the estate's hardened release flow (build-once →
npm OIDC Trusted Publishing → GitHub Packages mirror → tag + GitHub release;
A-328/326/323) into a `workflow_call`. Unlike the lint/build-test bundles it
publishes, so it carries some unique rules:

- **No `secrets:` block.** The npm leg authenticates via OIDC Trusted Publishing
  (no `NPM_TOKEN`); the Packages leg, `gh release create` and the failure issue
  all use the automatic `GITHUB_TOKEN`. Only **permissions** cross the boundary —
  the caller grants the superset (`contents`/`id-token`/`issues`/`packages`/
  `attestations: write`); each reusable job narrows from there.
- **npm validates the _caller_, not this callee.** npm Trusted Publishing matches
  the OIDC `workflow_ref` (the consumer's own `release.yml` caller filename +
  repo + environment), not the reusable `job_workflow_ref`. So each consuming
  package registers **its own caller filename** as the npmjs.com Trusted
  Publisher, and `id-token: write` must sit on **both** the caller job and the
  publish jobs. Unproven by inspection — verify with one live publish before
  estate rollout.
- **Consumer prerequisite — branch-protected `npm-release` environment.** A
  reusable workflow's `environment:` resolves in the **caller** repo; each
  consumer must define `npm-release` restricted to the release branch (A-326).
  If it is absent GitHub silently auto-creates it **unprotected**, losing the ref
  gate — so it is a hard prerequisite, not a default.
- **Caller owns the trigger; no `workflow_dispatch`.** `workflow_call` cannot own
  `on: push`/`concurrency`; the caller sets push-to-`main` +
  `cancel-in-progress: false` and must **not** add `workflow_dispatch` — a
  dispatched run satisfies the same npm OIDC subject as a legitimate post-merge
  push and could publish a poisoned tarball with valid provenance (A-326).
- **The two publish scripts are inlined** (not the consumer's per-repo
  `infrastructure/scripts/publish-*.sh`), centralising the logic and killing the
  drift A-384 targets. The `load-repo-config` outputs the per-repo copy reads
  (registries, scope, node-version file) become `with:` inputs instead.
- **Build reuses Layer-1** `setup-project` + `build`; the publish legs are
  hand-rolled (they need `setup-node`'s `registry-url`/`scope`) and pin
  `pnpm/action-setup` + `setup-node` to the **same SHAs** `setup-project` uses, so
  the toolchain cannot drift between the build and the publish.

## Composite actions (Layer 1)

ADR 0001 (`docs/adr/0001-shared-ci-architecture-for-npm-packages.md`, §5.7) splits
the estate's CI into two layers: **Layer 1 = granular composite actions** under
`.github/actions/` (the pick-and-mix primitives) and **Layer 2 = the coarse
`reusable-*.yml` workflows** that compose them. Splitting at the action layer is
free (steps in the calling job — no extra runner/install); splitting at the
workflow layer is not (a job each). So composability lives in the actions; the
reusable workflows stay few and pay setup once. See `.github/actions/README.md`.

- Each action takes plain `with:` inputs; in the reusable workflows those are
  fed from each consumer's `repo-config.yaml` (the per-repo `load-repo-config`
  stays local). Composite actions inherit the calling job's permissions.
- `lint-yaml` injects **this repo's own** `.yamllint.yml` (resolved relative to
  `github.action_path`) so consumers carry no local copy (A-438). The root file
  stays the single source of truth — keep it in sync, don't fork a copy.
- Tool versions/pins (pnpm, Node, yamllint, actionlint) **mirror `ci.yml`** so the
  dogfooded self-CI and the shipped actions never drift. Keep them aligned.
- **This repo does not dogfood the actions via `uses:`** — same
  `sha_pinning_required` reason as the inline workflows below. They are authored
  here and consumed cross-repo by `@<sha>`; Layer 2 (A-415/416) wires them.

### Why the PR-title check is inline (and there are no `./` callers)

The org enforces **`sha_pinning_required`**, which rejects local
`uses: ./.github/workflows/…` reusable references — they aren't SHA-pinnable, so
they fail at startup ([community #170337](https://github.com/orgs/community/discussions/170337)).
A cross-repo self-reference (`acme-skunkworks/shared-workflows/…@<sha>`) is
circular before the first tagged SHA exists. So this repo does **not** consume
its own reusable workflows; `ci.yml` inlines the PR-title check instead (a
SHA-pinned copy of `reusable-validate-pr-title.yml`'s step, same canonical job
name). Keep the inline copy in sync if the reusable one changes.

Consumers are unaffected: they reference the reusable workflows by cross-repo
`@<sha>`, which **is** SHA-pinning compliant.

### Why the Claude workflows are inline too

For the same reason, dogfooding `@claude` and Claude review **on this repo** uses
SHA-pinned **inline** copies — `claude.yml` and `claude-code-review.yml` — not
caller stubs pointing at the `reusable-*` versions. The stock output of
`/install-github-app` (floating `@v4`/`@v1` tags, no author-association gate)
won't even start here — `sha_pinning_required` rejects unpinned actions — so
these inline copies port the hardened `reusable-claude*.yml` bodies verbatim:
SHA-pinned actions, the A-313 author-association gate, `persist-credentials:
false`, timeouts and review concurrency. The `workflow_call` `inputs` become
literals and the review's `paths-ignore` (invalid on `workflow_call`) moves onto
the real `pull_request` trigger. **Keep these in sync with their `reusable-*`
counterparts** — same rule as the inline PR-title gate.

### Status-check context (A-400 / A-405)

`reusable-validate-pr-title.yml`'s job is named
`Validate PR title is a Conventional Commit`. A reusable-workflow check renders
as `<caller-job-id> / <job-name>`, so the caller job id must be **`pr-title`**
everywhere, giving the estate-canonical context
`pr-title / Validate PR title is a Conventional Commit` that one required-check
rule pins across the estate. Do **not** rename either half.

## Pinning

Third-party actions are SHA-pinned with a `# vX.Y.Z` comment (estate policy);
Dependabot bumps them weekly (grouped). Keep the same discipline when adding or
updating any `uses:` line.

## Commands

```bash
pnpm install         # install dev tooling + the husky hooks (prepare)
pnpm lint:md         # markdownlint-cli2 over Markdown
pnpm lint:yaml       # yamllint . (needs yamllint on PATH)
pnpm lint:workflows  # actionlint (needs actionlint on PATH)
pnpm format          # prettier --write .
pnpm format:check    # prettier --check .
```

Local tooling install (macOS): `brew install yamllint actionlint`.

## Linting and formatting

- **Markdown** — `.markdownlint-cli2.jsonc` extends `@acme-skunkworks/markdownlint-config`.
- **YAML** — `.yamllint.yml`: syntax errors fail; style rules are warnings
  (Prettier owns formatting). Workflow truthy values (`on`, `off`, …) allowed.
- **Workflows** — `actionlint` validates schema + expression syntax.
- **Prettier** — `pnpm format`; `.prettierignore` excludes `node_modules` and
  `pnpm-lock.yaml`.

This repo deliberately omits the wider estate's `infrastructure/scripts/` + bats
apparatus — it holds workflows, not a published package, so CI installs
`yamllint`/`actionlint` inline and keeps `ci.yml` lean.

## Local hooks

`pnpm install` runs `prepare` (`husky`), installing `.husky/`:

- **pre-commit** — `lint-staged`: prettier, markdownlint, and read-only
  yamllint/actionlint on staged files (best-effort; skips with a hint if a tool
  isn't installed).
- **commit-msg** — strips `Co-Authored-By: Claude … <noreply@anthropic.com>`
  trailers (Claude is tooling, not a contributor).
- **pre-push** — blocks direct pushes to `main` (open a PR instead) and runs
  yamllint + actionlint as the last gate before CI. Bot identities bypass.

Hooks are dormant in CI: `HUSKY=0` is set on the only job that runs
`pnpm install` (the `markdown` job in `ci.yml`) — that's where husky's `prepare`
would otherwise install them. Other jobs never install deps, so they need no
override.

## Testing workflows locally with `act`

`.actrc` pins `ubuntu-latest` to `catthehacker/ubuntu:act-latest`. Example:

```bash
act pull_request -W .github/workflows/ci.yml
```

The Claude workflows need `CLAUDE_CODE_OAUTH_TOKEN`; pass it via
`act --secret CLAUDE_CODE_OAUTH_TOKEN=…` (never commit a `.secrets` file — it's
gitignored).

## Git / PR flow

The repo squash-merges and uses the PR title + description as the merge message,
so the PR title must be a Conventional Commit (enforced by the inline `pr-title`
job in `ci.yml`; consumers use `reusable-validate-pr-title.yml`).
Never push to `main` directly — branch and open a PR.
