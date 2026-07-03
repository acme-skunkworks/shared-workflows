---
title: "Add the reusable-build-test Layer-2 bundle + build action"
release_note: "Coarse build/test bundle (reusable-build-test.yml) composing build → typecheck → test → shellcheck → bats behind per-check opt-outs, plus the new build composite action."
version: "0.4.0"
created_at: "2026-06-26T16:23:59Z"
category: feature
breaking: false
issues: ["A-416"]
branch: sk-416-reusable-build-testyml-workflow
merged_at: "2026-06-26T16:23:59Z"
commit: d0a5949
merge_strategy: squash
pr: 7
stats:
  loc_added: 222
  loc_removed: 7
  files_changed: 5
  commits: 2
---

## Added

- **`reusable-build-test.yml`** (`on: workflow_call`) — the build/test half: one
  job pays `setup-project` once, then composes `build` → `typecheck` → `test`
  (Vitest) → `shellcheck` → `bats`, each behind a boolean opt-out (ADR 0001
  §5.7). Gate naming is deliberately omitted (the gate stays in each consumer's
  local `GO/NO GO` aggregator).
- **`build` composite action** (`pnpm run build`) — a verification build that
  proves the package compiles/bundles cleanly and discards the output;
  Layer-1 had shipped no `build` action.

## Changed

- Bumped the embedded `anthropics/claude-code-action` pin
  (`1.0.152 → 1.0.157`) in `reusable-claude.yml` /
  `reusable-claude-code-review.yml` via grouped Dependabot.
