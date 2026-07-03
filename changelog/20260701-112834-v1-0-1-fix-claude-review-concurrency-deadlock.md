---
title: "1.0.1 — fix the Claude-review caller concurrency deadlock"
release_note: "Fixes the reusable Claude review deadlocking every caller at startup; consumers can now adopt the caller stub. Also documents the caller permissions each reusable needs."
version: "1.0.1"
created_at: "2026-07-01T11:28:34Z"
category: fix
breaking: false
issues: ["A-621"]
branch: a-621-fix-reusable-claude-review-concurrency-deadlock
merged_at: "2026-07-01T11:28:35Z"
commit: ff71514
merge_strategy: squash
pr: 21
stats:
  loc_added: 44
  loc_removed: 3
  files_changed: 4
  commits: 3
---

## Fixed

- **`reusable-claude-code-review.yml` deadlocked every caller at startup.** The
  reusable declared its own top-level `concurrency:`; because a called reusable
  runs as the caller's job, that group resolved to the same value the caller sets
  in its own top-level `concurrency:`, and GitHub cancelled the run at startup ("a
  deadlock was detected for concurrency group … between a top level workflow and
  'claude-review'"). This broke Claude review for every consumer that adopted the
  caller — with no jobs, no logs and no REST-API annotation, so it was invisible
  to `actionlint` and only surfaced when a human noticed the reviews had stopped.
  The top-level `concurrency:` is removed; concurrency now lives solely in the
  **caller stub**, where the `pull_request` trigger sits.

## Added

- **CI guard against the deadlock recurring.** `ci.yml` now fails the build if any
  `reusable-*.yml` reintroduces a top-level `concurrency:` — the one structural
  invariant behind the bug, and something neither `actionlint` nor the REST API
  can catch. Job-level `concurrency:` (indented) is unaffected, and the inline
  self-hosted copies keep theirs because they are standalone workflows, not
  `workflow_call` callees.
- **Required-caller-`permissions:` documentation.** Under the org default of
  `default_workflow_permissions: read`, a caller with no `permissions:` block
  hands the reusable a read-only token and the run dies at startup
  (`startup_failure`) with the reason hidden from the REST API. The README now
  spells out the top-level `permissions:` block each reusable's caller must
  declare, and the caller-stub snippets carry the correct scopes.

## Consumer note

Re-adopters pin the caller to the `v1.0.1` SHA (Dependabot bumps it thereafter).
The estate-wide rollout of the fixed caller is tracked separately in A-623.
