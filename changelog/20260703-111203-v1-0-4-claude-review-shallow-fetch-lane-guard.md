---
title: "1.0.4 — fix Claude-review shallow-fetch, add the empty-lane guard, name the required secret in caller stubs"
release_note: "Fixes the shared Claude review dying at a shallow git fetch (no review ever posted), fails a lint/build-test bundle fast when every lane is disabled, and names CLAUDE_CODE_OAUTH_TOKEN inline in the caller stubs."
version: "1.0.4"
created_at: "2026-07-03T11:12:03Z"
category: fix
breaking: false
issues: ["A-445", "A-636", "A-665"]
---

## Fixed

- **The shared Claude PR-review workflow died at a shallow `git fetch`.** With
  `track_progress` on, `claude-code-action` runs an internal
  `git fetch origin --depth=20 <head-branch>`, but the review checked out the PR
  **merge** ref at `fetch-depth: 1` — a shallow clone with no head-branch ref — so
  the fetch failed with exit 1 _after_ OIDC succeeded and no review ever posted
  (observed on octavo #90 and the shared-agents-md / portcullis review runs).
  `reusable-claude-code-review.yml` (and the inline self-host copy, kept in sync)
  now check out at `fetch-depth: 0` so every branch ref is present (A-636).
- **A lint/build-test bundle with every lane disabled exited green.** It checked
  out, ran nothing and passed — a silent success a consumer's `GO/NO GO`
  aggregator counts as a verification that never happened. Both bundles now fail
  fast at a `🚫 Validate lane selection` guard before checkout when no lane is
  enabled (A-445).

## Added

- **The lane-selection guard** in `reusable-lint.yml` and
  `reusable-build-test.yml`, with the rationale documented in ADR 0001 §5.7 as a
  narrow exception to the per-repo opt-out model (A-445).
- **The required secret is named inline in the Claude caller stubs.** Both stub
  templates (README) and the `reusable-claude*.yml` "Consume from another repo"
  header snippets carry a comment on `secrets: inherit`: _"Requires
  CLAUDE_CODE_OAUTH_TOKEN in THIS repo's Actions secrets — NOT ANTHROPIC_API_KEY."_
  It copies into every consumer stub so triage is pointed at the right secret from
  the stub itself, not just the README prose (A-665). The A-646 fail-fast guard
  for an empty token already shipped in `v1.0.3`.

## Consumer note

Consumers on the floating `@v1` tag pick this up automatically once `v1` moves to
`v1.0.4`. Two repos still pinned to the `v1.0.2` SHA — **shared-agents-md** and
**portcullis** — are red on the A-636 shallow-fetch bug and must migrate their
Claude callers to `@v1` (or re-pin to `v1.0.4`) to recover; that consumer
migration is tracked separately. This release only changes the Claude-review job
body (the `fetch-depth` fix) and the lint/build-test bundles; the interactive
`@claude` job body is unchanged.
