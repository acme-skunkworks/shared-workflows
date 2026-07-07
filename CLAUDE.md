# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

Claude Code reads only `CLAUDE.md`, so the `@AGENTS.md` line below imports the canonical shared
block (which Cursor reads from `AGENTS.md` natively). Estate-wide guidance lives there;
repo-specific guidance follows below.

@AGENTS.md

## Repo

Home for the Acme Skunkworks estate's **reusable GitHub Actions workflows**
(`workflow_call`). One authoritative copy of each common workflow lives here;
consumer repos keep thin caller stubs that point at it (SHA-pinned + Dependabot).
This replaces the previous copy-paste-per-repo approach.

The product is the `reusable-*.yml` files under `.github/workflows/`. Everything
else (this repo's own `ci.yml`, the caller stubs, lint configs) exists to lint
and dogfood them.

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
│   ├── reusable-pkg-release.yml         # PRODUCT: build-once → npm OIDC + Packages mirror (Layer 2)
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
- **Top-level `concurrency:` must NOT appear in a `reusable-*.yml`.** A called
  reusable runs as the caller's job, so a workflow-level concurrency group
  resolves to the same value the caller declares and GitHub deadlock-cancels the
  run at startup ("a deadlock was detected … between a top level workflow and
  '<job>'"). It is invisible to `actionlint` and to the REST API (A-621), so a
  guard step in `ci.yml` fails the build if any `reusable-*.yml` reintroduces
  one. Concurrency lives in the **caller stub**, where the trigger sits; the
  inline self-hosted copies (`claude-code-review.yml`, `ci.yml`) keep theirs
  because they are standalone workflows, not `workflow_call` callees.
- The `reusable-` prefix avoids a filename collision with the same-named caller
  stub in a consumer.
- **A Layer-2 workflow references its sibling Layer-1 actions by full cross-repo
  path, SHA-pinned** (`acme-skunkworks/shared-workflows/.github/actions/…@<sha>`),
  never `./.github/actions/…`: inside a `workflow_call` job a `./` path resolves
  against the **caller's** workspace, not this repo
  ([community #18601](https://github.com/orgs/community/discussions/18601)), and
  `sha_pinning_required` rejects an unpinned local ref. The full-path form is also
  what populates `github.action_path` so `lint-yaml`'s `.yamllint.yml` injection
  resolves. `reusable-lint.yml` pins these to the **`v0.8.0`** tag's SHA
  (`@7f543bef1… # v0.8.0`) — the first release whose tag carries every composite
  action; Dependabot bumps it like any other tag pin. (The full release history
  was backfilled in A-585: `v0.1.0`–`v0.8.0` plus the `v1.0.0` stable release.)

## The pkg-release workflow (`reusable-pkg-release.yml`, A-417)

`reusable-pkg-release.yml` ports the estate's hardened release flow (build-once →
npm OIDC Trusted Publishing → GitHub Packages mirror → tag + GitHub release;
A-328/326/323) into a `workflow_call`. It is named **`pkg-release`** everywhere
(this Layer-2 file `reusable-pkg-release.yml`; each consumer's caller stub
`pkg-release.yml`) to mark it a _package_ release — an npm publish, not an app
deployment (A-543). Unlike the lint/build-test bundles it publishes, so it
carries some unique rules:

- **No `secrets:` block.** The npm leg authenticates via OIDC Trusted Publishing
  (no `NPM_TOKEN`); the Packages leg, `gh release create` and the failure issue
  all use the automatic `GITHUB_TOKEN`. Only **permissions** cross the boundary —
  the caller grants the superset (`contents`/`id-token`/`issues`/`packages`/
  `attestations: write`); each reusable job narrows from there.
- **npm validates the _caller_, not this callee.** npm Trusted Publishing matches
  the OIDC `workflow_ref` (the consumer's own `pkg-release.yml` caller filename +
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
- **`build: false` for build-less packages.** The verification build is gated on a
  `build` input (default `true`, mirroring `reusable-build-test.yml`). A config-only
  or bundle-only consumer with no `build` script (markdownlint-config, agent-skills)
  passes `build: false` — else the build step fails `ERR_PNPM_NO_SCRIPT`. `npm pack`
  still packs the tarball from the package sources regardless.

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
the real `pull_request` trigger. Both also carry the A-646 empty-token guard step
(a `🚦 Guard` that fails fast when `CLAUDE_CODE_OAUTH_TOKEN` is empty — the
required secret is that token, **not** `ANTHROPIC_API_KEY`), and the review copy
carries the A-646 `github.actor != 'dependabot[bot]'` skip (a Dependabot run can't
read Actions secrets, so it would authenticate with an empty token). **Keep these
in sync with their `reusable-*` counterparts** — same rule as the inline PR-title
gate.

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

## Releasing this repo (A-597)

This repo publishes **no npm package** — a "release" is a `vX.Y.Z` git tag + a
GitHub Release over the reusable workflows and composite actions. The flow is
modelled on **octavo**, the estate's `kind: deploy` (non-publishing)
release-orchestrator target.

- **release-please** (`release-please-config.json`, `release-type: node`,
  `skip-changelog: true`; version state in `.release-please-manifest.json` +
  `package.json`) maintains the release PR from the Conventional-Commit history
  and, on merge, cuts the `vX.Y.Z` tag + GitHub Release. It does **not** write a
  root `CHANGELOG.md` — the dated [`changelog/`](changelog/) entries remain the
  curated human record.
- **`group-pull-request-title-pattern` is mandatory (A-677).** With
  `separate-pull-requests: false` release-please titles the combined release PR
  from its _group_ pattern, whose default (`chore: release main`) carries no
  `${version}`/`${component}`. The orchestrator's `release-please github-release`
  step then can't parse the merged title and cuts **no** tag/Release (so
  `move-floating-major` never fires). We pin
  `"chore${scope}: release${component} ${version}"` so the title round-trips —
  the same key must sit in every `kind: deploy` target (octavo) and is documented
  as a prerequisite in the orchestrator's onboarding docs. It does **not** affect
  the tag (`include-component-in-tag: false` keeps tags bare `vX.Y.Z`).
- **`move-floating-major.yml`** force-moves the floating major tag (`v1`, …) onto
  each release commit — shared-workflows-specific (consumers pin `@v1`, A-662),
  which octavo has no equivalent of. Phase-independent.
- **Phased rollout (A-597):** _Phase A (current)_ — the in-repo
  `release-please.yml` drives releases directly. Because a `GITHUB_TOKEN`-opened
  PR does not trigger this repo's `on: pull_request` CI, **`GO/NO GO` must not be
  a required check on `main`** while Phase A is live (the release PR's checks
  never run, so a required gate would wedge it). _Phase B_ — hand release-please
  off to the private release-orchestrator (add shared-workflows to its
  `orchestrate-releases.yml` / `enrich-changelogs.yml` matrices), delete
  `release-please.yml`, enable forward changelog authoring (`send-it`
  `changelog: true` + the `changelog` skill), and make `GO/NO GO` required.

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

## Agent skills

This repo adopts the shared `@acme-skunkworks/agent-skills` bundles, installed via
[skills.sh](https://skills.sh) under `.claude/skills/` (mirrored to `.agents/skills/`
for Cursor). Each skill reads its own `config.json`, reconciled to this repo's facts by
the `initialise-skills` skill (base branch, package roots, Linear team / workspace,
issue keys). The installed skills are:

- **`/send-it`** — bundle uncommitted work into atomic Conventional Commits, run the
  lint preflight, set a Conventional Commits PR title, push, open or update a draft PR,
  and move linked Linear issues to **In Review**.
- **`/preflight`** — change-gated, branch-scoped lint preflight on the current branch.
- **`/linear-sync`** — transition the Linear issue(s) linked to the current branch.
- **`/cleanup-repo`** — prune merged branches and worktrees, then filesystem cruft,
  behind a single confirmation gate (with a `--dry-run` preview).
- **`/triage-pr`** — drive a draft PR with failing CI to merge-ready: fix in-scope CI
  failures, then action unresolved AI review feedback.

This repo runs the shared **`changelog`** flow (A-597): the `changelog` skill is
installed, `send-it` authors a dated `changelog/` entry per branch
(`changelog: true` in its `config.json`), and CI validates entries against the
contract via the `📓 Changelog` job in the `GO/NO GO` aggregator. Post-merge
enrichment (filling `merged_at` / `commit` / `stats`) is **not yet wired** — that
is the estate-wide deploy-target enrichment build (A-597 Layer B); until it lands
those fields stay blank, exactly as on octavo today.

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

### CodeRabbit review + the `skip-review` label (A-667)

CodeRabbit (a separate SaaS bot from the Claude review workflows) auto-reviews
every PR and shares a fair-usage quota. Estate-wide fan-outs open half a dozen
mechanical re-sync PRs at once, which would otherwise burn that quota and throttle
the PRs bringing new functionality. To opt a rollout PR out, label it
**`skip-review`**.

`.coderabbit.yaml` at the repo root drives this with a **denylist**:

```yaml
reviews:
  auto_review:
    labels:
      - "!skip-review"
```

The `!`-prefix is a negative match — CodeRabbit reviews every PR _except_ those
labelled `skip-review`. It **fails safe**: a forgotten label means more review,
not less. CodeRabbit reads this file from the PR's **base branch**, so a change to
it only takes effect once merged to `main`.

The label is repo-local (create it with `gh label create skip-review --color
ededed --description 'Fanned-out rollout PR — CodeRabbit skips it'` before it can
be applied). Fanning the config + label out across the estate — and having
re-sync PRs opened with `--label skip-review` — belongs to the private
`release-orchestrator` (it already holds the cross-repo `road-runner-bot`
credential), **not** this repo, which has no cross-repo write capability by
design. Agent-facing guidance for agents to apply the label will live in the
estate's shared `AGENTS.md` once A-668 ships.

#### Claude Code Review honours the same label (A-716)

The `skip-review` label is cross-bot: `reusable-claude-code-review.yml`'s job
`if:` also skips a PR carrying it (alongside its existing draft / `release-please--`
/ dependabot skips). So a single label opts an automated PR out of **both** the
CodeRabbit SaaS review and the Claude Code Review workflow. The
`release-orchestrator` applies it to every automation PR it opens — fan-out
rollouts and changelog-enrichment PRs alike — so neither bot spends quota on an
unattended, auto-merging PR that carries no new behaviour.

#### The canonical config is more than the denylist (A-732)

Because A-712 fans `.coderabbit.yaml` out **verbatim** to every consumer flagged
`coderabbit: true`, this file is the estate's single source of truth and must be a
**strict superset** of every skip convention — anything omitted reverts that
consumer to CodeRabbit's own defaults. Beyond the `skip-review` denylist it carries:

- **`language: "en-GB"`** — review prose in British English, matching house style.
- **`reviews.profile: "chill"`** — a balanced signal-to-noise default (fewer
  nitpicks than `assertive`, more than `quiet`).
- **Two more skips** under `auto_review`: an `ignore_title_keywords` entry
  (`"enrich entry for"`) and an `ignore_usernames` entry
  (`"road-runner-bot[bot]"`) silence the orchestrator's mechanical
  changelog-enrich PRs (the username skip also covers road-runner-bot's
  release-please PRs). **Dependabot PRs are deliberately not skipped** —
  dependency bumps warrant a review (fail-safe toward more review).
- **`path_filters`** excluding vendored/generated trees (`pnpm-lock.yaml`,
  `**/dist/**`, `**/node_modules/**`, and the re-vendored `.claude/skills/**` +
  `.agents/skills/**` bundles) — reviewing them is noise.
- **`path_instructions`** encoding two estate policies as reviewer guidance:
  British English for `**/*.md` prose (scoped to prose, not identifiers), and for
  `.github/**` the SHA-pin-with-`# vX.Y.Z`-comment rule plus the no-top-level-
  `concurrency:`-in-`reusable-*.yml` rule.

Keep this file a superset when editing: stripping a skip would silently re-review
those PRs across the whole estate. Brace-expansion globs (`{md,mdx}`) are **not**
documented in CodeRabbit's schema, so path patterns avoid them.
