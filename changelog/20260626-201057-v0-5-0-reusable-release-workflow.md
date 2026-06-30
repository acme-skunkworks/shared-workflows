---
title: "Add the reusable release workflow (build-once → npm OIDC → Packages mirror)"
release_note: "The last commodity Layer-2 workflow: the estate's hardened release flow (build-once → npm OIDC Trusted Publishing → GitHub Packages mirror → tag + GitHub release) as one workflow_call source of truth."
version: "0.5.0"
created_at: "2026-06-26T20:10:57Z"
category: feature
breaking: false
issues: ["A-417"]
---

## Added

- **`reusable-release.yml`** (`on: workflow_call`) — ports the estate's hardened
  release flow into one source of truth, replacing the per-repo `release.yml`
  copies that would otherwise drift. Three least-privileged jobs:
  - **Build & pack** (`contents: read`) — unprivileged build-once; packs the
    single tarball both legs ship.
  - **Release (npm)** (`contents`/`id-token`/`issues: write`) — OIDC Trusted
    Publishing (no `NPM_TOKEN`), version-vs-tag gate, tag + GitHub release, and a
    failure issue.
  - **Publish to GitHub Packages** (`packages`/`id-token`/`attestations: write`)
    — mirror leg + build-provenance attestation.
