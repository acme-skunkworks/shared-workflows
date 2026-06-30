# ADR 0001 — Shared CI architecture for the npm-package estate (layered reusable workflows + per-repo `GO/NO GO` gate)

- **Status:** Proposed (architecture deliverable for the Shared Workflows project — direction,
  not implementation). Direction chosen: **layered building blocks (composite actions + reusable
  `workflow_call` workflows) consumed by thin per-repo callers, with the release gate owned by a
  per-repo `GO/NO GO` aggregator**. See §5–§6.
- **Date:** 2026-06-25
- **Deciders:** Rob Easthope
- **Related:** A-411 (stand up shared-workflows), A-412 (`GO/NO GO` release gate) and its children
  A-415/416/417/418/419/420/421/425, A-403/405 (PR-title gate, estate lockstep), A-422 (pre-GA
  SHA-pin + security review), A-428 (shared Claude pair callers); release-orchestrator ADR 0001
  (event-driven triggering); A-384 (drifting per-repo `infrastructure/` copies).
- **Scope:** The CI/release workflows of the estate's **published npm packages** —
  `eslint-config`, `markdownlint-config`, `npm-package-template`, `agent-skills`. The private
  turbo/changesets monorepos (`hecate`, `waterleaf`, `protomolecule`) and the standalone app
  (`climbwell`) are **explicitly out of scope** here; a monorepo tier is noted as future work (§8).
  Octavo-class **deploy targets** are out of scope for v1 but the design leaves a documented seam
  for them (§5.4, §8).

---

## 1. Context

The estate is consolidating per-repo, copy-pasted CI onto a single authoritative home —
`acme-skunkworks/shared-workflows` — with consumer repos keeping thin, SHA-pinned caller stubs
(A-384, A-411). At the same time the `release-orchestrator` has gone live: it opens release-please
PRs on each target's behalf and **merges them only when a named check-run goes green**. The CI we
build must therefore satisfy two masters at once — be **fast and low-maintenance** for small
single-package libraries, and **expose exactly the pass/fail signal the orchestrator polls for**.

This ADR was commissioned to answer a specific question: _take the estate's most fleshed-out CI
(Tempest) as a reference and decide how to build shared, performant workflows that slot into the
upcoming `GO/NO GO` release gate._ The short answer is that **Tempest is the right teacher and the
wrong template** — §3 explains why — and that the shared workflows should be **lean, layered, and
gate-agnostic**, leaving the gate to a tiny per-repo aggregator.

### 1.1 What Tempest actually is

`tempest/.github/workflows/checks.yml` is a 1,157-line, single-workflow CI for a large pnpm + Turbo
monorepo with Supabase, Playwright E2E, SQL migration safety, and auto-committed generated types. It
earns its complexity:

- a 10-flag `detect-changes` job gates every downstream job on `git diff origin/main...HEAD`
  (ADR-0004 in that repo);
- `pnpm turbo run <task> --filter=...[origin/main]` runs only changed packages **and their
  dependents**;
- a **dual-layer cache** with a _dedicated_ `setup-and-cache` save job to avoid the multi-job
  save-contention bug (their GH-231 / GH-264), with all other jobs restoring read-only via a
  `setup-project` composite action;
- vitest CI-hardening (`pool: forks`, `maxWorkers: 2`, `retry: 1`, raised timeouts,
  `--max-old-space-size`);
- `concurrency` with `cancel-in-progress: true`.

Every one of those decisions is a response to monorepo scale. None of them is free to maintain.

### 1.2 What the npm packages actually are

The four in-scope repos are **single-package** libraries that already converge hard:

- **Release:** all on **release-please via `road-runner-bot`** (the orchestrator opens the PR), dual
  publish to **npm (OIDC Trusted Publishing) + GitHub Packages**, provenance/attestation on.
- **CI skeleton:** near-identical `ci.yml` — `🔬 Build & Lint` (build + eslint + markdown +
  changelog validation) · `Validate PR title is a Conventional Commit` · `📝 YAML & Workflows`
  (yamllint + actionlint) · `🧪 Infrastructure scripts` (shellcheck + vitest + bats).
- **Shared apparatus:** a local `load-repo-config` action reading a per-repo `repo-config.yaml`;
  exact-key (no `restore-keys`) caches for `yamllint`/`actionlint`/`bats` to defeat stale hits.
- **Divergence is small and legitimate:** `markdownlint-config` has no build artefact (the config
  _is_ the package); `agent-skills` adds a hard `validate:skills` metadata gate and ships
  unbuilt bundles.

A full build + lint + test for any of these runs in **seconds**. The dominant cost is runner
spin-up, `pnpm install`, and tool download — not compute. That single fact drives the performance
posture in §5.5.

### 1.3 The orchestrator's interface (the hard constraint)

`orchestrate-releases.yml` decides "go" by polling the **GitHub Checks API** on the release PR's
head SHA for a check-run **whose `.name` matches a literal string**, taking the most recent run by
monotonic `id`, every 30 s for ~12 min, and merging only on `conclusion == success`
(`--match-head-commit` TOCTOU guard, A-334). Today that literal is `🔬 Build & Lint`. The estate is
mid-migration to a single, purpose-built gate named **`GO/NO GO`** (A-412): an `if: always()`
aggregator job that `needs:` every real CI job, whose **intrinsic check-run is the gate**. Because a
check-run can only be minted by a GitHub App (the repo's own Actions), it **cannot be forged** by a
push-scoped token or a fork — and a ruleset pins the required reporting integration to GitHub Actions
(A-418/A-425). This is the signal our shared workflows must make it trivial — and unforgeable — for
a consumer to emit.

---

## 2. Decision drivers

| #   | Driver                              | Why it matters                                                                                                                                                                                                                  |
| --- | ----------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| D1  | **Single source of truth**          | Kill the drift across copy-pasted `ci.yml`/`infrastructure/` (A-384). One place to bump a SHA or a rule.                                                                                                                        |
| D2  | **Orchestrator-compatibility**      | CI must emit the exact gate check-run the orchestrator polls (`GO/NO GO`, migrating from `🔬 Build & Lint`), on `release-please--*` PRs, as a required check.                                                                   |
| D3  | **Performance for small repos**     | Optimise the actual bottleneck (spin-up, install, tool fetch), not imaginary compute. Don't import monorepo machinery a single package can't amortise.                                                                          |
| D4  | **Flexibility without forking**     | Repos share 90% but each keeps a legitimate 10% (no-build config package, skills metadata gate, Octavo's CircleCI input). The design must absorb that without per-repo copies of the shared logic.                              |
| D5  | **Security / supply-chain**         | Public repos, assumed-exfiltratable CI. PR-triggered workflows take **no secrets** and never use `pull_request_target`; anything privileged is push-to-`main` only; all cross-repo `uses:` are SHA-pinned (A-411 scope, A-422). |
| D6  | **`sha_pinning_required` org rule** | Local `uses: ./.github/workflows/…` reusable refs are rejected at startup; only cross-repo `@<sha>` is compliant. This shapes _how_ repos consume the shared workflows and why this repo dogfoods inline.                       |
| D7  | **Low maintenance ceiling**         | Solo-maintainer estate (0 required approvals, no CODEOWNERS). The architecture must be boring to keep green.                                                                                                                    |

---

## 3. Is Tempest the right model? (the core assessment)

**Verdict: adopt Tempest's _techniques_ that pay off at small scale; reject its _shape_.**

| Tempest technique                                         | Keep for npm packages? | Rationale                                                                                                             |
| --------------------------------------------------------- | ---------------------- | --------------------------------------------------------------------------------------------------------------------- |
| `concurrency` + `cancel-in-progress`                      | **Yes**                | Free, saves wasted runs on rapid pushes. Already estate-standard.                                                     |
| Composite `setup-project` action (pnpm + Node + cache)    | **Yes**                | This is precisely the "layer 1" building block (§5.2). High reuse, low surface.                                       |
| Exact-key tool caches (the npm repos already do this)     | **Yes**                | The real perf win at this scale is _not re-downloading actionlint/yamllint/bats_.                                     |
| Vitest CI-hardening (forks/workers/retry)                 | **Partial**            | Useful as defaults in the shared `build-test`, but small suites rarely need it. Parameterise, don't mandate.          |
| 10-flag `detect-changes` + path-gated jobs                | **No (defer)**         | A single-package repo has nothing to skip; the diff plumbing is pure maintenance cost. Reserve for the monorepo tier. |
| `turbo run --filter=...[origin/main]`                     | **No (defer)**         | No dependency graph to prune in a one-package repo.                                                                   |
| Dual-layer cache with dedicated save job (contention fix) | **No (defer)**         | Solves a many-parallel-jobs problem the npm repos don't have.                                                         |
| Supabase / Playwright / SQL-safety jobs                   | **No**                 | Repo-specific; not commodity.                                                                                         |

The conclusion the requester floated — "if Tempest is a poor model I'm open to other approaches" — is
half-right: it's a poor _template_ to clone but an excellent _catalogue of solved problems_ to borrow
from selectively. The shared workflows take the cheap, high-leverage half and leave the
monorepo-scale half on the shelf until a monorepo actually consumes them.

---

## 4. Options considered

### 4.1 Packaging shape

- **A — Reusable `workflow_call` workflows only.** Simplest to consume; one opinionated whole-job
  file per concern. But a consumer that needs to vary one step (e.g. `markdownlint-config`'s missing
  build) must either fork the workflow or drown it in `with:` flags.
- **B — Composite actions only.** Maximum flexibility, but every repo re-writes its own job
  scaffolding and there is no central place that emits canonical job names — re-introducing drift.
- **C — Layered: composite actions _and_ thin reusable workflows (chosen).** Low-level actions
  (setup/cache) are the reusable atoms; reusable workflows compose them into standard jobs; repos
  with standard needs call the workflow, repos that must customise drop to the actions. Matches the
  existing `reusable-*` direction and A-411's "commodity logic shared, `GO/NO GO` stays per-repo".

**Chosen: C.** It is the only option that delivers D1 _and_ D4 simultaneously.

### 4.2 Who owns the `GO/NO GO` gate check-run

- **A — A shared reusable workflow emits `GO/NO GO` directly.** Every consumer gets the gate "for
  free", but it boxes in repos that add their own checks (the aggregator must `needs:` _those_ too,
  which a central workflow can't see).
- **B — A per-repo `GO/NO GO` aggregator that `needs:` the shared jobs (chosen).** Each repo keeps a
  ~10-line `if: always()` aggregator that depends on its real jobs (shared callers _and_ any local
  extras) and emits the intrinsic `GO/NO GO` check-run. The shared workflows stay gate-agnostic
  (A-416 explicitly: build-test is "`🔬 Build & Lint`-equivalent, **minus the gate naming**"). This
  is already the shipped reference in `npm-package-template` (A-424, Done).

**Chosen: B.** The gate must be able to see _all_ of a repo's jobs, including local ones; only the
repo itself can express that `needs:` list. This also keeps the unforgeability property local to the
repo's own Actions (D5).

### 4.3 Performance posture

- **A — Full Tempest-grade machinery everywhere.** Rejected (§3): cost without benefit at this scale.
- **B — Lean now, measured upgrades later (chosen).** Ship the cheap wins (concurrency-cancel,
  setup composite, exact-key caches); add change-detection/sharding only where a measured CI time
  justifies it. Keeps the maintenance ceiling low (D7) and leaves an evidence-based path to scale.

**Chosen: B.**

---

## 5. Architecture

### 5.1 Layer map

```
┌─ Consumer repo (e.g. eslint-config) ───────────────────────────────┐
│  .github/workflows/ci.yml         ← thin caller stubs + triggers   │
│    ├─ uses: …/reusable-lint.yml@<sha>                              │
│    ├─ uses: …/reusable-build-test.yml@<sha>                        │
│    ├─ uses: …/reusable-validate-pr-title.yml@<sha>                 │
│    ├─ <local extra jobs, e.g. validate:skills>                     │
│    └─ go-no-go: needs:[all of the above]  if: always()  ← THE GATE │
│  .github/workflows/pkg-release.yml      ← caller stub (pkg release)│
│  repo-config.yaml + load-repo-config (stays local)                 │
└────────────────────────────────────────────────────────────────────┘
        │ cross-repo, SHA-pinned, Dependabot-bumped
        ▼
┌─ acme-skunkworks/shared-workflows ─────────────────────────────────┐
│  Layer 2 — reusable workflows (on: workflow_call)                   │
│    reusable-lint.yml (A-415) · reusable-build-test.yml (A-416)    │
│    reusable-validate-pr-title.yml (A-403, shipped)                 │
│    reusable-pkg-release.yml (A-417) · reusable-claude*.yml (shipped)│
│  Layer 1 — composite actions (action.yml)                          │
│    setup-project (pnpm + Node-from-.nvmrc + caches)                │
│  Governance — versioned estate rulesets JSON (A-425)              │
└────────────────────────────────────────────────────────────────────┘
```

### 5.2 Layer 1 — composite actions (the atoms)

A `setup-project` composite action (the portable distillate of Tempest's) does pnpm install + Node
setup via `node-version-file: .nvmrc` + the exact-key tool/store caches. The pnpm store cache
restores at job start and saves in a single post-job step (`cache: pnpm`); the contention-avoidance
_dedicated save job_ pattern is **not** needed at single-package scale (one setup per workflow, not
nine). Consumed by the reusable workflows and available to any repo that needs to hand-assemble a
non-standard job.

### 5.3 Layer 2 — reusable workflows (the commodity jobs)

`on: workflow_call`, parameterised by `with:` inputs (e.g. `build: false` for
`markdownlint-config`), **never** receiving secrets on PR triggers (D5). The four canonical units —
`lint`, `build-test`, `pr-title`, `pkg-release` — plus the already-shipped Claude pair. Crucially the
build/lint/test workflows **do not name themselves the gate**; the caller's job id + the reusable
job name compose the check context, and the gate name is applied by the aggregator in 5.4.

The PR-title workflow keeps its load-bearing name: the caller job id must be `pr-title` and the job
name `Validate PR title is a Conventional Commit`, giving the estate-pinned context
`pr-title / Validate PR title is a Conventional Commit` (A-403/405). **Do not rename either half.**

### 5.4 The gate — per-repo `GO/NO GO` aggregator

Each consumer keeps a tiny job:

```yaml
go-no-go:
  name: GO/NO GO
  needs: [build-and-lint, pr-title, yaml-lint, infra] # + any local jobs
  if: ${{ always() }} # MANDATORY, else it skips and never reports
  runs-on: ubuntu-latest
  steps:
    - name: ⚖️ Verdict
      run:
        | # fail if any needed job failed/cancelled; skip-allowlist for release-please--* path-skips
        …
```

Its **intrinsic check-run** `GO/NO GO` is what the orchestrator polls and what the ruleset requires
(pinned to the GitHub Actions integration, so it is unforgeable — A-418/425). For **Octavo-class
deploy targets** (future, §8) the same aggregator gains a bounded-poll step that reads CircleCI's
typegen status as an **input** (A-421) — CircleCI is never the authority and the orchestrator never
talks to it; the gate still runs on GitHub Actions.

### 5.5 Performance posture (lean now, measured later)

Per D3, the optimisation targets are spin-up/install/tool-fetch, addressed by: `concurrency` +
`cancel-in-progress`; exact-key caches for `yamllint`/`actionlint`/`bats` + pnpm store; `pnpm install
--frozen-lockfile`; jobs that genuinely parallelise (lint ∥ build-test ∥ yaml ∥ infra) fanning in to
the aggregator. Deliberately **omitted** until a measured CI time demands them: change-detection,
turbo filtering, test sharding, dual-layer save jobs. If/when a repo's CI time crosses a pain
threshold, add the _specific_ technique from the Tempest catalogue (§3) that the data points at —
not the whole apparatus.

### 5.6 Why this repo dogfoods inline, not via `./` callers (D6)

`sha_pinning_required` rejects `uses: ./.github/workflows/…`, and a cross-repo self-reference is
circular before the first tagged SHA exists. So `shared-workflows` itself runs SHA-pinned **inline**
copies of the PR-title and Claude logic (per its CLAUDE.md), kept in sync with the `reusable-*`
bodies. Consumers are unaffected — they reference `acme-skunkworks/shared-workflows/…@<sha>`, which
**is** compliant. This is a property of the host repo, not a wart in the architecture, but it is the
reason "just call your own reusable workflow" is not on the table here.

### 5.7 Granularity — where "pick-and-mix" lives

The estate wants composable CI: not everything jammed into one monolithic lint workflow, but a
pick-and-mix of checks a repo opts into. The key constraint that decides _where_ to split is a
GitHub Actions cost asymmetry:

- A **reusable workflow** (`workflow_call`) is a **job** — it gets its own fresh runner and its own
  `pnpm install`. Splitting one workflow into five tiny ones costs **5× spin-up + install**.
- A **composite action** runs as **steps inside the calling job** — no extra runner, no extra
  install. Splitting here is essentially free, and a failing step still shows a red ✗ at step
  granularity in the job log.

At single-package scale **setup dominates runtime** (§1.2), so splitting at the _workflow_ layer is
the expensive axis and splitting at the _action_ layer is the cheap one. Therefore:

**Pick-and-mix lives at the composite-action layer; reusable workflows are a few coarse bundles.**

- **Granular composite actions (the mix-ins, split freely):** `setup-project`, `eslint`,
  `lint-markdown`, `lint-yaml` (yamllint + actionlint), `build`, `typecheck`, `test-vitest`,
  `test-bats`, `shellcheck`, `changelog-validate`. A repo composes exactly the ones it needs inside
  one job — one setup, N checks.
- **Coarse reusable workflows (the convenience bundles, kept few):** `reusable-lint.yml`
  (A-415 — setup + eslint + markdown + yaml + changelog) and `reusable-build-test.yml`
  (A-416 — setup + build + typecheck + test + infra) each pay setup **once** and run several
  checks; plus the standalone-by-nature `reusable-validate-pr-title.yml` (no setup),
  `reusable-pkg-release.yml` (different trigger + secrets), and the Claude pair. **Deliberately not**
  one reusable workflow per linter — that multiplies setup for no benefit a step boundary doesn't
  already give.
- **Two escape hatches for partial opt-out**, in increasing order of cost:
  1. **Boolean `with:` inputs** on the coarse workflows (`markdown: false`, `eslint: false`) — drop a
     sub-check without leaving the bundle. (This is how `markdownlint-config` sets `build: false` and
     how `agent-skills` ran `eslint: false` pre-A-394.)
  2. **Drop to the action layer** — assemble a bespoke job from the composite actions when the
     standard bundle doesn't fit.
  3. **Promote a check to its own job/workflow** only when a repo genuinely wants it as a _separate
     required check-run_ or for parallelism worth the extra setup — the exception, not the default.
- **The gate is granularity-agnostic.** The `GO/NO GO` aggregator just `needs:` whatever jobs the
  repo wired — one coarse job or six fine ones — so the gate never forces the choice (§5.4).

Net: maximum composability where it's free (actions/steps), minimum setup tax where it's expensive
(jobs/workflows), and a clean ladder from "call the bundle" → "tweak an input" → "assemble from
actions" → "promote to its own check".

---

## 6. Consequences

### Good

- **One source of truth** for the commodity 90% (D1); SHA bumps and rule edits happen once
  (Dependabot-bumped in consumers).
- **Orchestrator-clean:** the gate is a per-repo concern that can see all jobs, and the shared
  workflows never need to know the gate name (D2/D4).
- **Cheap to run and to own:** lean posture matches the real cost curve; nothing to maintain that a
  single package can't amortise (D3/D7).
- **Secure by construction:** no secrets on PR triggers, no `pull_request_target`, SHA-pinned
  everywhere, unforgeable check-run gate (D5).

### Bad / costs

- **The transitional double-name window.** During rollout the orchestrator dual-accepts
  `🔬 Build & Lint` _and_ `GO/NO GO` (A-419); until every served repo emits `GO/NO GO` and the
  orchestrator goes single-name, both must be kept working. This is real coordination cost (§7).
- **Inline dogfooding drift risk** in this repo (the `reusable-*` ↔ inline copies must be kept in
  sync — an existing, accepted CLAUDE.md hazard).
- **Per-repo aggregator boilerplate** (~10 lines × N repos) is duplicated by design (option 4.2-B);
  the trade is correctness (gate sees local jobs) over DRY.

### Neutral

- Per-repo `repo-config.yaml` + `load-repo-config` **stay local** (A-411) — values reach the shared
  workflows via `with:` inputs, not by centralising the config.
- Monorepos and the app keep their bespoke CI until a monorepo tier is actually built (§8).

---

## 7. Migration & rollout (maps to existing Linear)

The forward path is already ticketed under **A-412**; this ADR ratifies it rather than inventing it:

1. **Author the building blocks** — A-415 (`lint`), A-416 (`build-test`), A-417 (`release`),
   PR-title shipped (A-403); `setup-project` composite. (A-411 umbrella, In Progress.)
2. **Define the gate pattern + ruleset** — A-418; version the rulesets in this repo — A-425.
3. **Reference consumer** — `npm-package-template` already adopts shared callers + local `GO/NO GO`
   (A-424/413, **Done**) and is the canonical `GO/NO GO` emitter.
4. **Roll out across the fleet** — A-420: add callers + local aggregator + required-`GO/NO GO`
   ruleset to `eslint-config`, `agent-skills`, `markdownlint-config` (and Octavo, via A-421),
   **dual-running** alongside `🔬 Build & Lint`.
5. **Flip the orchestrator** — A-419: `CHECK_NAME` dual-accept → `GO/NO GO`-only once every served
   repo emits it (keep the A-334 TOCTOU guard).
6. **Shared Claude pair callers** — A-428 (and per-repo, e.g. A-433 Octavo).
7. **Pre-GA gate** — A-422: SHA-pin policy enforcement + security review of the shared workflows.

### 7.1 Decommission the old gate immediately — A-437

The _forward_ migration is well covered, but the **decommission of `🔬 Build & Lint` as the gate** was
only **implicit** in A-419's "then `GO/NO GO`-only" clause — not its own tracked step, so it risked
being "finished" while stale scaffolding lingered. That gap is now closed by **A-437** (child of
A-412, raised alongside this ADR).

The policy decision is to **collapse the double-name window as soon as `GO/NO GO` is in place and
verified** across every served repo — not to leave `🔬 Build & Lint` dual-accepted indefinitely.
A-437 fires immediately after A-420 (fleet emits a required `GO/NO GO`) + A-419 (orchestrator
dual-accepting), and deletes:

- the orchestrator's dual-accept of `🔬 Build & Lint`, leaving it polling `GO/NO GO` **only** (keep
  the A-334 TOCTOU guard);
- the `🔬 Build & Lint` required-check in each repo's ruleset, swapped for `GO/NO GO` pinned to the
  GitHub Actions integration;
- the transitional _"do NOT rename… dual-accept"_ comments in `npm-package-template`'s `ci.yml` and
  any shared docs.

The `🔬 Build & Lint` **job** is not removed — it survives as an ordinary CI job feeding the
aggregator's `needs:`. Only its **gate role** and the transitional scaffolding go.

---

## 8. Adjacent work — shared config-package sweep

The shared _workflows_ are only half the consolidation story; the other half is the shared _configs_
they run. The estate already extracts linters/formatters into publishable `@acme-skunkworks/*`
packages under the **Open source** initiative. Tempest is the canonical baseline here — its lint
stack is the **superset** the Octavo project will draw from — so the sweep is "diff Tempest's config
files against the projects that already exist, and create projects for the gaps."

### 8.1 Config projects that already exist (Open source initiative)

| Config       | Package                                | Linear project      | State                              |
| ------------ | -------------------------------------- | ------------------- | ---------------------------------- |
| ESLint       | `@acme-skunkworks/eslint-config`       | eslint-config       | In Progress (published)            |
| markdownlint | `@acme-skunkworks/markdownlint-config` | markdownlint-config | In Progress (published)            |
| TypeScript   | `@acme-skunkworks/tsconfig`            | tsconfig            | Idea (baseline from Tempest; A-96) |
| Vitest       | `@acme-skunkworks/vitest-config`       | vitest-config       | Idea (baseline from Tempest)       |
| Stylelint    | `@acme-skunkworks/stylelint-config`    | style-lint          | Idea (Tailwind + standard)         |

### 8.2 Gaps found — and how they were resolved

Diffing Tempest's config files against the table above left two genuine gaps. Both are now actioned:

- **`prettier-config` → new publishable package.** Tempest ships `.prettierrc.json` (JSON-formatting
  overrides, plus the `prettier-plugin-astro` / `prettier-plugin-tailwindcss` plugins); all four npm
  packages carry Prettier + a local `.prettierignore` with no shared preset. Stood up as the
  **prettier-config** project under Open source (Idea), scaffolded from `npm-package-template`,
  baseline from Tempest — the natural home to encode the recurring A-378 / GH-848 formatter
  friction once.
- **yamllint → centralised in `shared-workflows`, not a package.** `.yamllint.yml` is **copy-pasted
  verbatim into all four npm repos** (and Tempest) — the per-repo drift A-384 exists to kill. The
  decision is **own-it-in-the-workflow** rather than publish-as-package: yamllint is CI-coupled and
  its `extends` support is weaker than ESLint's, so the canonical config lives in `shared-workflows`
  and is injected by `reusable-lint.yml`, leaving consumers with no local copy. Tracked as **A-438**
  (Shared Workflows project, related to A-415/384). This sets the estate convention: **editor-time
  configs are published packages (eslint/markdown/prettier/tsconfig/vitest); CI-time configs are
  centralised in the workflow (yamllint).**

Repo-specific configs are **not** candidates: Squawk (`.squawk.toml`) and SQLFluff are
Supabase/Postgres-only; `actionlint` has no config file (defaults) and low ROI; lint-staged/husky
are per-repo glue.

### 8.3 The Octavo angle

When Octavo lands on shared CI it will want the **web-app** slice of Tempest's stack — eslint,
prettier, tsconfig, markdown, yaml, and (once it has CSS) stylelint — but **not** the SQL/Supabase
configs. Standing up the new **prettier-config** package now, alongside the already-planned
tsconfig/vitest/stylelint projects, means Octavo consumes ready-made `@acme-skunkworks/*` presets
rather than re-deriving them from Tempest; its **yamllint** comes for free from the shared
`reusable-lint.yml` (§8.2), not a package. (Note the standing caveat that Octavo's CI itself is
moving to CircleCI for IPv6/Supabase typegen — A-421 — but its config _packages_ are runner-agnostic
and unaffected.)

## 9. Readiness audit — what to get in sooner rather than later

A quick health-check of the four in-scope npm packages and the configs feeding them:

- **Package manager / Node: clean.** All four are uniformly `pnpm@10.33.0`, `engines.node >=22`,
  pinned via `.nvmrc` (`22`). **No `yarn.lock` anywhere in the estate** — no yarn holdouts to migrate.
  The shared `setup-project` action can hard-code `node-version-file: .nvmrc` + pnpm with confidence.
- **File-type lint coverage: one asymmetry, now being closed.** All four lint Markdown, YAML, shell,
  and workflows (`lint:md` / `lint:yaml` / `lint:sh` / `lint:workflows`). `eslint-config` and
  `npm-package-template` additionally run `build` + `eslint` + `tsc`; **`agent-skills` runs no
  ESLint and no `tsc`** despite shipping a `tsconfig.json` (type-check-only, `noEmit`, covering its
  7 `infrastructure/scripts/**/*.ts`) — the config exists but no `tsc` script ever runs it (its
  `include` also carries a stale, empty `infrastructure/send-it/**/*.ts`). Decision taken to bring it
  to parity: **A-394** (revived) wires up `tsc` + adopts `@acme-skunkworks/eslint-config` for the
  infra `.ts` scripts. The shareable skill code is `.mjs` (zero-dep ESM, not in any tsconfig), which
  the TS-oriented preset doesn't yet cleanly lint standalone — **A-439** (eslint-config) validates/
  extends `.mjs` coverage and unblocks the `.mjs` half. Sequencing matters: the shared `build-test`
  caller (A-416) should not assume an ESLint/`tsc` lane in every repo until A-394 lands.
- **Config drift to retire (feeds §8.2).** `.yamllint.yml` and the `.markdownlint-cli2.jsonc` wrapper
  are duplicated across the four repos. markdownlint already extends the shared package; **yamllint
  does not** — close that by centralising it in `shared-workflows`/`reusable-lint.yml` (A-438), so
  consumers carry no local copy, landing the A-384 win.
- **Known formatter friction.** A-378 (eslint ↔ Prettier `jsonc` array conflict) and GH-848
  (`turbo.json` lint-staged loop) are recurring; a shared `prettier-config` is the natural place to
  encode the resolution once rather than per-repo `.prettierignore` patches.

None of these block the workflow rollout, but centralising yamllint (A-438) and the new
`prettier-config` package are the highest-leverage "do it while we're in here" items.

## 10. Open questions & future work

- **Monorepo tier.** When `hecate`/`waterleaf`/`protomolecule` adopt shared CI, the deferred Tempest
  machinery (change-detection, turbo filtering, dual-layer cache) becomes relevant. That is a
  separate ADR; this one deliberately does not design it, but §3's catalogue is the starting point.
- **Octavo / deploy targets.** In scope only as a documented seam (§5.4 + A-421). Note the
  standing tension: Octavo is moving its IPv6/Supabase typegen to **CircleCI**, yet its **gate stays
  on GitHub Actions** with CircleCI as a bounded-poll input. The orchestrator never depends on
  CircleCI.
- **`load-repo-config` home.** Stays local per A-411; revisit only if a security-review (A-422)
  argues for centralising the registry-URL validation.
- **Orchestrator latency.** Independent of this ADR but adjacent: release-orchestrator ADR 0001
  proposes replacing the 15-min cron poll with an external Cloudflare trigger + App webhook. If
  adopted, the gate's pass signal could be consumed faster — no change required here.
