---
title: Add reusable-validate-commits.yml base..head commitlint gate
release_note: New reusable workflow lints every commit in a PR's base..head range against @acme-skunkworks/commitlint-config (floated to latest), under the same no-secrets trust model as the PR-title gate.
created_at: '2026-07-23T12:10:27Z'
merged_at: '2026-07-23T13:05:09Z'
branch: a-981-build-reusable-validate-commitsyml-basehead-commitlint-gate
pr: 84
commit: 6ccb2ab
merge_strategy:
author: rob@acmeskunkworks.io
co_authors: []
category: feature
breaking: false
issues:
  - A-981
stats:
  files_changed: 6
  loc_added: 278
  loc_removed: 26
  commits:
---

## Added

- **`reusable-validate-commits.yml`** — `workflow_call` gate that checks out the
  PR with full history, installs `@commitlint/cli` +
  `@acme-skunkworks/commitlint-config` into an ephemeral prefix (floated to
  latest, public npm, no registry auth), and runs
  `commitlint --from <base.sha> --to <head.sha>`. Same no-secrets trust model as
  `reusable-validate-pr-title.yml`; no top-level `concurrency:`. Estate-canonical
  status-check context:
  `commits / Validate commits are Conventional Commits` ([A-981](https://linear.app/acme-skunkworks/issue/A-981);
  closes [shared-workflows#72](https://github.com/acme-skunkworks/shared-workflows/issues/72)).
- Inline dogfood of the same steps in `ci.yml`, wired into the `GO/NO GO`
  aggregator (this repo cannot call its own reusables under
  `sha_pinning_required`).

## Changed

- README catalogues the new reusable, caller stub, and float-latest ruleset
  model; `docs/rulesets.md` and `CLAUDE.md` document the consumer vs inline
  check-context forms.
