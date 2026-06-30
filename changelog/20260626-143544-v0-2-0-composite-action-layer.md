---
title: "Add the Layer-1 composite-action layer (setup-project + lint/test mix-ins)"
release_note: "Ten granular composite actions consumers pick-and-mix — the Layer-1 primitives the coarse reusable workflows compose."
version: "0.2.0"
created_at: "2026-06-26T14:35:44Z"
category: feature
breaking: false
issues: ["A-440"]
---

## Added

- **Layer-1 composite actions** under `.github/actions/` (ADR 0001 §5.1–§5.3,
  §5.7):
  - `setup-project` — pnpm + Node-from-`.nvmrc` + restore-only pnpm store cache,
    then install (the keystone).
  - `eslint`, `lint-markdown`, `lint-yaml` — lint mix-ins.
  - `typecheck`, `test-vitest`, `test-bats` — build/test mix-ins.
  - `shellcheck`, `changelog-validate` — infra/changelog mix-ins.
- `lint-yaml` injects **this repo's own** `.yamllint.yml` (resolved relative to
  `github.action_path`) so consumers carry no local copy.
- `.github/actions/README.md` — the action catalogue and the
  split-at-the-action-layer rationale.

## Changed

- Recorded the layered-CI direction in ADR 0001
  (`docs/adr/0001-shared-ci-architecture-for-npm-packages.md`), added one release
  earlier as the architecture's decision record.
