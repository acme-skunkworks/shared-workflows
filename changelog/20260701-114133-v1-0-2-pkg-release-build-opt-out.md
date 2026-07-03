---
title: "1.0.2 — pkg-release build opt-out for build-less packages"
release_note: "reusable-pkg-release.yml gains a `build: false` opt-out so config-only / bundle-only packages release without a `build` script."
version: "1.0.2"
created_at: "2026-07-01T11:41:33Z"
category: fix
breaking: false
issues: ["A-624"]
branch: a-624-reusable-pkg-release-add-build-optout
merged_at: "2026-07-01T11:41:33Z"
commit: 9febdb1
merge_strategy: squash
pr: 22
stats:
  loc_added: 18
  loc_removed: 0
  files_changed: 2
  commits: 1
---

## Fixed

- **`reusable-pkg-release.yml` failed the release for build-less packages.** Its
  build job ran `pnpm run build` unconditionally, so a package with no `build`
  script (config-only / bundle-only) died with
  `ERR_PNPM_NO_SCRIPT  Missing script: build`. This hit **markdownlint-config**
  and **agent-skills** the moment their Wave 3 migrations merged — the first
  `pkg-release` run on `main` failed on both. No publish was ever at risk: it
  fails at the verification-build step, before the version-vs-tag gate.

## Added

- **A `build` opt-out input.** `reusable-pkg-release.yml` now takes a `build`
  boolean (default `true`, mirroring `reusable-build-test.yml`) and gates the
  `🏗️ Build` step on `if: ${{ inputs.build }}`. `npm pack` still packs the
  tarball from the package sources, so a build-less consumer sets `build: false`
  and releases cleanly. The opt-out is documented in `CLAUDE.md`.

## Consumer note

Build-less publishers bump their `pkg-release.yml` caller to the `v1.0.2` SHA and
pass `with: { build: false }`. This release includes everything in `v1.0.1` (the
A-621 claude-review deadlock + caller-permissions fixes).
