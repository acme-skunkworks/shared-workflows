# Estate rulesets

GitHub **rulesets** export and import as JSON, so the estate's canonical
branch-protection lives here as version-controlled JSON
([`.github/rulesets/trunk.json`](../.github/rulesets/trunk.json)) rather than as
click-ops only. Enforcement is then reproducible and reviewable: the file is the
source of truth, and the per-repo apply below **replaces** whatever is live.

## What `trunk.json` encodes

A single `branch` ruleset on `~DEFAULT_BRANCH` (`main`), `enforcement: active`:

- **`deletion`** + **`non_fast_forward`** — the branch can't be deleted or
  force-pushed.
- **`pull_request`** — every change lands via a PR (so direct pushes to `main` are
  blocked), **squash-only** (`allowed_merge_methods: ["squash"]` — the PR title is
  the released commit subject), `0` required approvals, no code-owner review (the
  estate runs solo-maintainer, 0 approvals, no CODEOWNERS).
- **`required_status_checks`**, each **pinned to the GitHub Actions integration**
  (`integration_id: 15368`) so a forged commit status or a different App can't
  satisfy them (the anti-spoof requirement, A-418):
  - **`GO/NO GO`** — the per-repo aggregator check-run (see the `GO/NO GO` gate
    doc `docs/go-no-go-gate.md`, A-418). This is the single gate; it `needs:` every
    real CI job, so requiring it transitively requires them all.
  - **`pr-title / Validate PR title is a Conventional Commit`** — the estate-pinned
    Conventional-Commit title context (A-400/405), kept as its own required check
    because it also re-runs on title edits independently of the rest of CI.

This folds the two rulesets that were previously live on this repo into one: the
old `Trunk` (the real baseline) plus the redundant org-level `Protect main trunk`
(`deletion` + `non_fast_forward` only — a strict subset). See **Decommission**
below.

> **Block-direct-pushes** is the combined effect of the `pull_request` rule
> (changes must go through a PR) and `non_fast_forward` (no force-push) — there is
> no separate "no direct push" rule type.
>
> **No bypass actors.** `bypass_actors` is empty, so enforcement is strict: even an
> org admin cannot push directly or merge a non-conforming PR. Emergency access
> therefore means **temporarily toggling the ruleset to `evaluate`/`disabled`** (or
> adding a scoped bypass actor), not relying on an admin permission — there is no
> implicit admin override.

## Scope

Rulesets here apply to **org** repositories only
(`acme-skunkworks/<repo>`). Personal-account consumers are out of scope.

## The `pr-title` context has two forms — pick the one your repo emits

A reusable-workflow check renders as `<caller-job-id> / <job-name>`, so the
required context differs by **how** a repo runs the PR-title check:

| Repo                                                                                | Emitted context                                         |
| ----------------------------------------------------------------------------------- | ------------------------------------------------------- |
| **Consumers** (call `reusable-validate-pr-title.yml` with caller job id `pr-title`) | `pr-title / Validate PR title is a Conventional Commit` |
| **`shared-workflows` itself** (dogfoods the check **inline** in `ci.yml`)           | `Validate PR title is a Conventional Commit`            |

`trunk.json` carries the **consumer** form (the estate norm). When applying to
`shared-workflows` itself, swap that one context to the bare inline form first —
e.g.:

```bash
jq '(.rules[] | select(.type=="required_status_checks").parameters.required_status_checks[]
     | select(.context|test("Conventional Commit")).context)
    = "Validate PR title is a Conventional Commit"' \
  .github/rulesets/trunk.json > /tmp/trunk-shared-workflows.json
```

The `GO/NO GO` context is identical in both (a plain job name, not a reusable
check), so it never needs swapping.

## Prerequisite — the `GO/NO GO` check must exist first

A `required_status_checks` rule on a check the repo never emits leaves every PR
`Pending` forever. **Apply this ruleset only once the repo emits a `GO/NO GO`
check-run** (the aggregator from the gate doc `docs/go-no-go-gate.md`, A-418). For
`shared-workflows` that means after the `ci.yml` aggregator has merged to `main`.

## Per-repo apply process

The apply **replaces** the live ruleset — it does not add a second overlapping one.

1. **Find the live repo-level ruleset id** (the one to replace):

   ```bash
   gh api repos/<owner>/<repo>/rulesets \
     --jq '.[] | select(.source_type=="Repository") | "\(.id)\t\(.name)"'
   ```

2. **Update it in place** with the versioned JSON:

   ```bash
   # Consumers — apply trunk.json as-is:
   gh api --method PUT repos/<owner>/<repo>/rulesets/<id> \
     --input .github/rulesets/trunk.json
   ```

   ```bash
   # shared-workflows ONLY — apply the context-swapped file from the section above
   # (its inline pr-title context differs), never trunk.json directly:
   gh api --method PUT repos/acme-skunkworks/shared-workflows/rulesets/<id> \
     --input /tmp/trunk-shared-workflows.json
   ```

   Or, if the repo has no repo-level ruleset yet, **create** one with `--method
POST repos/<owner>/<repo>/rulesets` instead (same `--input` rule).

3. **Verify** the rules and the integration pin took:

   ```bash
   gh api repos/<owner>/<repo>/rulesets/<id> \
     --jq '.rules[] | select(.type=="required_status_checks").parameters.required_status_checks'
   ```

## Decommission the redundant org ruleset

`shared-workflows` also carried an **org-level** ruleset `Protect main trunk`
(`deletion` + `non_fast_forward` only) — a strict subset of `Trunk`, now redundant.
Because it is **org-sourced** it cannot be edited from this repo; remove it at
**org → Settings → Rules → Rulesets** (or
`gh api --method DELETE /orgs/<org>/rulesets/<id>`) once `trunk.json` is applied to
the member repos, so the baseline lives in one place.
