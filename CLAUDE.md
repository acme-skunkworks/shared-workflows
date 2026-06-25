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
├── workflows/
│   ├── reusable-claude.yml              # PRODUCT: interactive @claude
│   ├── reusable-claude-code-review.yml  # PRODUCT: PR review
│   ├── reusable-validate-pr-title.yml   # PRODUCT: conventional PR title
│   ├── claude.yml                       # dogfood caller → reusable-claude.yml
│   ├── claude-code-review.yml           # dogfood caller → reusable-claude-code-review.yml
│   ├── validate-pr-title.yml            # dogfood caller → reusable-validate-pr-title.yml
│   └── ci.yml                           # self-CI: actionlint + yamllint + markdownlint
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
  (ASW-329).
- **`paths-ignore`** cannot be set on `workflow_call`, so the Claude review's
  paths-ignore lives in the caller stub.
- The `reusable-` prefix avoids a filename collision with the same-named caller
  stub in this repo and in consumers.

### Status-check context (SK-400 / SK-405)

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

Hooks are dormant in CI (`HUSKY=0` set in workflow jobs).

## Testing workflows locally with `act`

`.actrc` pins `ubuntu-latest` to `catthehacker/ubuntu:act-latest`. Example:

```bash
act pull_request -W .github/workflows/validate-pr-title.yml
```

The Claude workflows need `CLAUDE_CODE_OAUTH_TOKEN`; pass it via
`act --secret CLAUDE_CODE_OAUTH_TOKEN=…` (never commit a `.secrets` file — it's
gitignored).

## Git / PR flow

The repo squash-merges and uses the PR title + description as the merge message,
so the PR title must be a Conventional Commit (enforced by `validate-pr-title.yml`).
Never push to `main` directly — branch and open a PR.
