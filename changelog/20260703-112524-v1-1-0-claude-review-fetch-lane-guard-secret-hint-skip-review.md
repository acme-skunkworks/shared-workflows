---
title: "1.1.0 — Claude-review shallow-fetch fix, empty-lane guard, secret-hint stubs, and a CodeRabbit skip-review opt-out"
release_note: "Fixes the shared Claude review dying at a shallow git fetch, fails a lint/build-test bundle fast when every lane is disabled, names CLAUDE_CODE_OAUTH_TOKEN inline in the caller stubs, and adds a skip-review CodeRabbit opt-out for rollout PRs."
version: "1.1.0"
created_at: "2026-07-03T11:25:24Z"
category: feature
breaking: false
issues: ["A-445", "A-636", "A-665", "A-667"]
---

## Added

- **A `skip-review` CodeRabbit opt-out** (A-667). A repo-root `.coderabbit.yaml`
  denylist (`labels: ["!skip-review"]`) reviews every PR **except** those carrying
  the `skip-review` label, so estate fan-outs can spare the shared fair-usage quota
  for PRs that bring new functionality. This is a **repo-internal dogfood** — it
  governs how _this_ repo is reviewed and does not change the reusable-workflow /
  composite-action product; it is the `feature` that makes this a minor release.
  Estate-wide distribution is tracked separately (A-669).
- **A lane-selection guard** in `reusable-lint.yml` and `reusable-build-test.yml`
  (A-445), documented in ADR 0001 §5.7 as a narrow exception to the per-repo
  opt-out model.
- **The required secret is named inline in the Claude caller stubs** (A-665). Both
  stub templates (README) and the `reusable-claude*.yml` `@v1` "Consume from another
  repo" header snippets carry a comment on `secrets: inherit`: _"Requires
  CLAUDE_CODE_OAUTH_TOKEN in THIS repo's Actions secrets — NOT ANTHROPIC_API_KEY."_
  so triage is pointed at the right secret from the stub itself. The A-646 fail-fast
  guard for an empty token already shipped in `v1.0.3`.

## Fixed

- **The shared Claude PR-review workflow died at a shallow `git fetch`** (A-636).
  With `track_progress` on, `claude-code-action` runs an internal
  `git fetch origin --depth=20 <head-branch>`, but the review checked out the PR
  **merge** ref at `fetch-depth: 1` — a shallow clone with no head-branch ref — so
  the fetch failed with exit 1 _after_ OIDC succeeded and no review ever posted
  (observed on octavo #90 and the shared-agents-md / portcullis review runs).
  `reusable-claude-code-review.yml` (and the inline self-host copy, kept in sync)
  now check out at `fetch-depth: 0` so every branch ref is present.
- **A lint/build-test bundle with every lane disabled exited green** (A-445). It
  checked out, ran nothing and passed — a silent success a consumer's `GO/NO GO`
  aggregator counts as a verification that never happened. Both bundles now fail
  fast at a `🚫 Validate lane selection` guard before checkout when no lane is
  enabled.

## Consumer note

Consumers on the floating `@v1` tag pick up the product fixes automatically once
`v1` moves to `v1.1.0`. Two repos still pinned to the `v1.0.2` SHA —
**shared-agents-md** and **portcullis** — are red on the A-636 shallow-fetch bug
and must migrate their Claude callers to `@v1` (or re-pin to `v1.1.0`) to recover;
that migration is tracked in A-671. The A-667 CodeRabbit opt-out is repo-internal
and does not reach consumers. The interactive `@claude` job body is unchanged.
