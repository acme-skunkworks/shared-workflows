---
title: "Version the canonical estate ruleset + add the apply docs"
release_note: "The estate branch-protection ruleset as reviewable JSON (.github/rulesets/trunk.json) plus the per-repo apply guide — branch protection is now reproducible, not click-ops only."
version: "0.8.0"
created_at: "2026-06-30T10:49:25Z"
category: feature
breaking: false
issues: ["A-425"]
---

## Added

- **`.github/rulesets/trunk.json`** — the estate's canonical GitHub ruleset
  versioned as JSON so branch-protection is reproducible and reviewable. Folds
  the two previously-live rulesets into one: `deletion` + `non_fast_forward` +
  `pull_request` (squash-only, 0 approvals) + `required_status_checks` **pinned
  to the GitHub Actions integration** (`integration_id: 15368`, anti-spoof),
  requiring `GO/NO GO` and the
  `pr-title / Validate PR title is a Conventional Commit` context.
- **`docs/rulesets.md`** — the per-repo apply process (`gh api PUT/POST` to
  **replace**, not add), the prerequisite that the `GO/NO GO` check must exist
  first, the two `pr-title` context forms (consumer caller vs this repo's inline
  dogfood), and the org-only scope.
