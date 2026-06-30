---
title: "1.0.0 — graduate shared-workflows to a stable public surface"
release_note: "First stable release. The reusable workflows and composite actions are settled; consumers can pin a vX.Y.Z tag (+ Dependabot) instead of a moving SHA."
version: "1.0.0"
created_at: "2026-06-30T15:53:52Z"
category: feature
breaking: false
issues: ["A-585"]
---

## Added

- **First stable release.** Graduates `shared-workflows` to `1.0.0`, marking the
  product surface settled and the public API stable enough for consumers to pin a
  `vX.Y.Z` tag (+ Dependabot bumps) rather than a bare commit SHA. The complete
  surface at 1.0.0:
  - **Layer-1 composite actions** — `setup-project`, `eslint`, `lint-markdown`,
    `lint-yaml`, `build`, `typecheck`, `test-vitest`, `test-bats`, `shellcheck`,
    `changelog-validate`.
  - **Layer-2 reusable workflows** — `reusable-claude`,
    `reusable-claude-code-review`, `reusable-validate-pr-title`, `reusable-lint`,
    `reusable-build-test`, `reusable-pkg-release`.
  - **Estate gates + reference docs** — the `GO/NO GO` aggregator gate and the
    versioned canonical ruleset.
- **Backfilled the full release history** (A-585): the `changelog/` entries and
  the `v0.1.0`–`v0.8.0` git tags + GitHub releases were reconstructed from the
  project's merged PRs, so the tag history is complete from the first feature
  rather than truncated at adoption.

## Changed

- `reusable-lint.yml` and `reusable-build-test.yml` now pin their sibling Layer-1
  composite actions to the `v0.8.0` tag instead of a bare pre-release commit SHA,
  so Dependabot can track and bump the same-repo refs.
- Re-synced the shared `@acme-skunkworks/agent-skills` bundles (wave 3).
  (Repo-internal tooling; invisible to consumers.)
