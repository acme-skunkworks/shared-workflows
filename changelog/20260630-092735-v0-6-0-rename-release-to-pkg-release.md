---
title: "Rename the release workflow to pkg-release across both layers"
release_note: "Rename the package-publish workflow release → pkg-release (reusable file + caller stub) to mark it a package release, not an app deployment — settled before the fleet rollout when it is cheapest."
version: "0.6.0"
created_at: "2026-06-30T09:27:35Z"
category: refactor
breaking: true
issues: ["A-543"]
---

## Breaking

- **`reusable-release.yml` → `reusable-pkg-release.yml`** (and the consumer caller
  stub `release.yml` → `pkg-release.yml`). The behaviour is identical, but the
  product workflow's **path** and display name (`Reusable / Package release`)
  changed. **Migration:** anyone pinning
  `…/.github/workflows/reusable-release.yml@<sha>` must repoint to
  `reusable-pkg-release.yml`. Blast radius was nil — no release tag and no
  consumers existed yet. The filename **is** the npm Trusted Publisher OIDC
  subject, so it is settled now, before the A-420 fleet rollout.

## Changed

- Renamed the Linear key prefix `SK-` → `A-` across docs and comments (no
  behavioural change).
- Renamed the release-gate check-run `go/no-go` → `GO/NO GO` in workflow comments,
  settling the canonical name ahead of the aggregator gate.
- Adopted and re-synced the shared `@acme-skunkworks/agent-skills` bundles into
  `.claude`/`.agents`, and gitignored the preflight skill's scratch output.
  (Repo-internal tooling; invisible to consumers.)
