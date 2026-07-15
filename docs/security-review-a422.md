# Pre-GA security review — shared workflows (A-422)

This is the sign-off record for [A-422](https://linear.app/acme-skunkworks/issue/A-422),
the pre-GA gate that declares the `reusable-*` workflows and Layer-1 composite actions
generally available. It reviews the **finished** trust surface (the workflows were already
authored and hardened — A-313/326/328/330/417/646), consciously blesses the floating-`@v1`
auto-propagation model, and confirms the org-level SHA-pin policy and Dependabot hygiene.

- **Reviewer:** Rob Easthope
- **Method:** read-only source review of every `reusable-*.yml` + `.github/actions/**`,
  plus live GitHub API probes of the org/repo policy and rulesets (evidence inline below).
- **Overall verdict:** **PASS to GA**, subject to the one open item in
  [§8](#8-findings--follow-ups) (ruleset source-of-truth drift — a hygiene fix, not a GA blocker).

The trust model this review validates is defined in
[ADR 0001 §D5/§D6](adr/0001-shared-ci-architecture-for-npm-packages.md) and codified for
consumers in [`SECURITY.md`](../SECURITY.md).

## 1. Surface — the publish path (`reusable-pkg-release.yml`)

This is the highest-value surface: it runs with `id-token: write` **inside each consumer**,
so the shared repo is part of every consumer's **publish trust boundary**.

| Control                                                                                                                                                                             | Where                              | Verdict |
| ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------- | ------- |
| Root `permissions: {}` (deny-all), per-job least privilege                                                                                                                          | `:189`, `:199/269/468/578`         | ✅      |
| `id-token: write` isolated to the two publish legs; `build` job holds no credential                                                                                                 | `:200`, `:271`, `:471`             | ✅      |
| `packages: write` isolated to the GitHub Packages leg, never the npm leg                                                                                                            | `:469-472` vs `:269-271`           | ✅      |
| npm OIDC Trusted Publishing — **no `NPM_TOKEN`**, `--provenance`                                                                                                                    | `:364-404`                         | ✅      |
| Build-once: one byte-identical tarball packed with `--ignore-scripts`, asserted `== 1 .tgz`, re-asserted in both publish legs — no build code runs next to a mintable token (A-328) | `:224-254`, `:340-355`, `:495-508` | ✅      |
| `push` + `ref == release-branch` gates on every publish/tag step                                                                                                                    | `:365-368`, `:412-415`, `:460-464` | ✅      |
| Version-vs-tag gate (publish iff `package.json` version has no matching tag)                                                                                                        | `:312-327`                         | ✅      |
| Registry-host pinning — GH Packages leg hard-codes `npm.pkg.github.com`, fails closed (A-330)                                                                                       | `:533-537`                         | ✅      |
| `persist-credentials: false` on every checkout (artipacked)                                                                                                                         | `:209/277/477`                     | ✅      |
| No `workflow_dispatch` — a dispatched run satisfies the same OIDC subject as a legitimate push and could publish a poisoned tarball with valid provenance (A-326)                   | `:94-95`, header `:43-48`          | ✅      |

**Residual (accepted, consumer-side):** two controls are enforced only in the **consumer
caller**, not here, because `workflow_call` cannot own them:

1. the branch-restricted `npm-release` **environment** (if absent, GitHub auto-creates it
   _unprotected_, losing the ref gate — a hard consumer prerequisite); and
2. the "no `workflow_dispatch`" rule on the caller trigger.

Both are documented in the file header and in `SECURITY.md`. npm Trusted Publishing
validates the **caller's** `workflow_ref`, not this callee, so it is **unproven by
inspection** — it must be verified with **one live publish** before estate rollout
([A-456](https://linear.app/acme-skunkworks/issue/A-456)).

## 2. Surface — secrets flow across the caller boundary

Every `reusable-*.yml` sets root `permissions: {}` and declares **typed** `secrets:` blocks
(no blanket `secrets: inherit` inside the callees). Secrets in play:

| Reusable                          | Secret                    | Consumed by                            | Empty-value guard       |
| --------------------------------- | ------------------------- | -------------------------------------- | ----------------------- |
| `reusable-claude.yml`             | `CLAUDE_CODE_OAUTH_TOKEN` | `claude-code-action` (`:117`)          | ✅ A-646 (`:93-109`)    |
| `reusable-claude-code-review.yml` | `CLAUDE_CODE_OAUTH_TOKEN` | `claude-code-action` (`:156`)          | ✅ A-646 (`:132-148`)   |
| `reusable-changelog-enrich.yml`   | `ROADRUNNER_PRIVATE_KEY`  | `create-github-app-token` (`:139-147`) | ✅ **added in this PR** |

All other reusables (`pkg-release`, `build-test`, `lint`, `load-repo-config`,
`validate-payload`, `validate-pr-title`) declare **no `secrets:` block** and rely only on
the automatic `GITHUB_TOKEN` — correct, since none needs a user secret.

**Finding fixed here:** `reusable-changelog-enrich.yml` previously lacked the A-646
empty-token guard the two Claude workflows carry — a `required: true` secret only forces
the caller to _map_ the secret, not to supply a non-empty _value_. An empty
`ROADRUNNER_PRIVATE_KEY` would have died deep inside JWT signing with an opaque error and
silently skipped the changelog write-back. A matching `🚦 Guard` step now fails fast.

**Verdict:** ✅ PASS (guard asymmetry closed).

## 3. Surface — third-party SHA-pinning

Every third-party `uses:` across `.github/workflows/**` and `.github/actions/**` is
SHA-pinned with a `# vX.Y.Z` comment. **No third-party action floats on a tag.**

**Finding fixed here:** `reusable-validate-payload.yml` pinned `actions/checkout` to an
**older v4.2.2** while every other file uses **v7.0.0**. Re-pinned to the estate-current
`9c091bb… # v7.0.0` — a consistency fix (not a floating-pin violation).

Internal Layer-1 composite actions are referenced by full cross-repo path, SHA-pinned to
the `v1.5.0` tag SHA (`ba77002…`). **One divergence noted, deferred:**
`reusable-pkg-release.yml:212/220` still pin `setup-project`/`build` to the pre-release SHA
`d0a5949` rather than `ba77002 # v1.5.0`. This is within
[A-618](https://linear.app/acme-skunkworks/issue/A-618)'s re-pin remit and is **not**
changed here (see [§8](#8-findings--follow-ups)). It is not a security regression — `d0a5949`
is a real, immutable in-repo SHA — only a pin-consistency item.

**Verdict:** ✅ PASS.

## 4. Surface — the go/no-go gate + ruleset

The committed source of truth is [`.github/rulesets/trunk.json`](../.github/rulesets/trunk.json)
(a repo-level ruleset) plus the org-level "Protect main trunk" ruleset. Both required
status checks are pinned to the GitHub Actions integration (`integration_id: 15368`) so they
**cannot be forged** by a non-Actions check of the same name.

**Live API probe (2026-07-15):**

```text
$ gh api /orgs/acme-skunkworks/actions/permissions
  { "sha_pinning_required": true, ... }                     # org-wide — see §5

$ gh api /repos/acme-skunkworks/shared-workflows/rulesets
  16137621  "Protect main trunk"  Organization  active      # deletion + non_fast_forward
  18130461  "Trunk"               Repository     active      # the trunk.json ruleset
```

The org ruleset "Protect main trunk" (`bypass_actors: []`) adds a second, org-owned layer of
`deletion` + `non_fast_forward` protection on top of the repo ruleset.

**Finding (open — see [§8](#8-findings--follow-ups)):** the **live** repo "Trunk" ruleset drifts
from the committed `trunk.json` in two respects:

|                    | Committed `trunk.json`                                  | Live ruleset `18130461`                                         |
| ------------------ | ------------------------------------------------------- | --------------------------------------------------------------- |
| `bypass_actors`    | `[]`                                                    | roadrunner-bot App (`actor_id: 2195582`, `bypass_mode: always`) |
| 2nd required check | `pr-title / Validate PR title is a Conventional Commit` | `validate-payload / Validate fanned payload`                    |

Neither drift weakens the gate: `GO/NO GO` (which itself aggregates `pr-title`) remains
required and integration-pinned, and the roadrunner-bot bypass is the **intended** estate
design — GitHub Actions (the `GITHUB_TOKEN`) cannot be a Trunk bypass actor (ADR 0004), so
the release orchestrator pushes release commits and moves tags under its App identity, which
therefore **must** hold the bypass. But the committed JSON is meant to be the auditable
source of truth (`docs/rulesets.md`), so it should be reconciled to match live enforcement.

**Verdict:** ✅ PASS (gate is sound); ⚠️ reconcile `trunk.json` to live (hygiene, §8).

## 5. Org-level SHA-pin policy — enforced org-wide

A-422 requires confirming the "block non-SHA-pinned actions" policy is enforced at **org**
level, not merely per repo. **Confirmed:**

```text
$ gh api /orgs/acme-skunkworks/actions/permissions
  { "enabled_repositories": "all", "allowed_actions": "all", "sha_pinning_required": true }
```

`sha_pinning_required: true` on `/orgs/acme-skunkworks/…` means every repo in the org
rejects an unpinned third-party `uses:` at workflow start-up — a missed pin cannot ship,
regardless of per-repo settings. This governs third-party `uses:` **actions**;
reusable-**workflow** calls are tag-exempt under the same rule, which is exactly what lets
consumer callers float on `@v1` ([§6](#6-floating-v1-auto-propagation--conscious-sign-off)).

**Verdict:** ✅ CONFIRMED (org-level).

## 6. Floating-`@v1` auto-propagation — conscious sign-off

Since [A-662](https://linear.app/acme-skunkworks/issue/A-662) (v1.0.3), consumer caller
stubs float on the `@v1` major tag rather than SHA-pinning each shared-workflow reference.
[`move-floating-major.yml`](../.github/workflows/move-floating-major.yml) force-moves the
lightweight `v1` ref onto each release commit (on `release: published`, `contents: write`,
orchestrator-published releases only — a `GITHUB_TOKEN`-created release does not fire the
trigger).

**The trade-off, stated plainly:** moving `v1` ships new reusable-workflow code into **every
consumer's publish trust boundary** with **no per-consumer SHA bump, no Dependabot PR, and no
per-consumer review gate**. This is the mechanism GA depends on for hands-off fleet rollout.

**This review consciously blesses it as acceptable, because:**

1. **The moved code already passed this repo's own gate.** `v1` only advances onto a commit
   that shipped a `vX.Y.Z` release, which required a green `GO/NO GO` (integration-pinned)
   and human review on `main` before release.
2. **Consumers opted in.** Floating `@v1` is the consumer's deliberate choice; a consumer
   wanting change-review-per-bump can pin `@<sha>` instead (still compliant).
3. **The blast radius inside the reusables stays pinned.** Every third-party `uses:` _within_
   the reusables remains SHA-pinned ([§3](#3-surface--third-party-sha-pinning)) and is
   bumped only via a reviewed Dependabot PR here — floating `@v1` moves _our_ reviewed code,
   never an unreviewed upstream jump.
4. **Only the orchestrator can move the tag.** The App-identity trigger nuance means a stray
   `GITHUB_TOKEN` release cannot advance `v1`.

**Sign-off:** ✅ auto-propagation into the publish boundary is **accepted** for GA.

## 7. Dependabot hygiene

[`.github/dependabot.yml`](../.github/dependabot.yml) runs **weekly, grouped** updates for
both `github-actions` and `npm`. The `github-actions` ecosystem scans every `uses:` in
`.github/workflows/**` and `.github/actions/**`, so the SHA-pinned third-party actions
**inside** the reusable workflows (`actions/checkout`, `anthropics/claude-code-action`,
`pnpm/action-setup`, …) are bumped in the weekly grouped `ci:` PR.

By design it does **not** bump the internal same-repo
`acme-skunkworks/shared-workflows/.github/actions/…@<sha>` self-references — those advance
with the floating-tag flow, not a registry bump.

**Verdict:** ✅ CONFIRMED.

## 8. Findings / follow-ups

| #   | Item                                                                                                                                       | Disposition                                                                                                                     |
| --- | ------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------- |
| 1   | `actions/checkout` v4.2.2 → v7.0.0 drift in `reusable-validate-payload.yml`                                                                | **Fixed in this PR.**                                                                                                           |
| 2   | Missing A-646 empty-token guard for `ROADRUNNER_PRIVATE_KEY` in `reusable-changelog-enrich.yml`                                            | **Fixed in this PR.**                                                                                                           |
| 3   | `reusable-pkg-release.yml:212/220` internal pins on `d0a5949` vs `ba77002 # v1.5.0`                                                        | **Deferred to [A-618](https://linear.app/acme-skunkworks/issue/A-618)** (its re-pin remit). Not a security regression.          |
| 4   | Live "Trunk" ruleset drifts from committed `trunk.json` (roadrunner-bot bypass actor; 2nd required check `validate-payload` vs `pr-title`) | **Open — reconcile the committed JSON to live** (or re-apply if the JSON is authoritative). Gate remains sound in the meantime. |
| 5   | npm OIDC Trusted Publishing unproven by inspection                                                                                         | **Verify with one live publish — [A-456](https://linear.app/acme-skunkworks/issue/A-456).**                                     |

Items 1–2 land here; 3–5 are tracked elsewhere. None blocks GA.
