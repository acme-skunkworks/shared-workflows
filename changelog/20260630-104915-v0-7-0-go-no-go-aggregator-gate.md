---
title: "Add the GO/NO GO aggregator gate pattern + reference docs"
release_note: "The canonical GO/NO GO release gate: an always()-running aggregator job that concludes only when every real CI job passed or legitimately skipped, dogfooded here and documented for consumers."
version: "0.7.0"
created_at: "2026-06-30T10:49:15Z"
category: feature
breaking: false
issues: ["A-418"]
branch: a-418-define-gono-go-aggregator-pattern-integration-pinned-ruleset
merged_at: "2026-06-30T10:49:15Z"
commit: 79b6e52
merge_strategy: squash
pr: 16
stats:
  loc_added: 109
  loc_removed: 0
  files_changed: 3
  commits: 3
---

## Added

- **The `GO/NO GO` aggregator gate**, dogfooded in this repo's `ci.yml`: an
  `if: always()` job that `needs:` every real CI job (`pr-title`, `workflows`,
  `markdown`) and concludes `success` only when all passed or legitimately
  skipped. Its intrinsic `GO/NO GO` check-run is the single gate the estate
  ruleset requires (pinned to the GitHub Actions integration so it cannot be
  forged) and the release orchestrator polls.
- **`docs/go-no-go-gate.md`** — the canonical consumer aggregator with its
  footguns (never path-filter the gate; `always()` is mandatory; treat `skipped`
  against an allowlist; keep the `GO/NO GO` name exact) and the dual-accept
  rollout note. Linked (with ADR 0001) from `README.md`.
