---
title: "Add the reusable-lint Layer-2 bundle"
release_note: "Coarse lint bundle (reusable-lint.yml) that pays setup-project once, then composes the shared eslint / markdown / yaml / changelog checks behind per-check opt-outs."
version: "0.3.0"
created_at: "2026-06-26T15:19:26Z"
category: feature
breaking: false
issues: ["A-415"]
branch: sk-415-reusable-lintyml-workflow
merged_at: "2026-06-26T15:19:26Z"
commit: b67e60c
merge_strategy: squash
pr: 6
stats:
  loc_added: 189
  loc_removed: 9
  files_changed: 6
  commits: 3
---

## Added

- **`reusable-lint.yml`** (`on: workflow_call`) — one job that pays
  `setup-project` once, then composes the Layer-1 lint actions: `eslint`,
  `lint-markdown`, `lint-yaml` (yamllint + actionlint), `changelog-validate`
  (ADR 0001 §5.7). Each sub-check has a boolean opt-out
  (`eslint`/`markdown`/`yaml`/`changelog`) so a consumer can drop one without
  leaving the bundle.

## Changed

- `lint-yaml`'s default `actionlint-version` bumped `1.7.5 → 1.7.12`, kept in
  step with this repo's self-CI pins so the dogfooded and shipped tooling never
  drift.
