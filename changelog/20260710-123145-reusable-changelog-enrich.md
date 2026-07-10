---
title: "Add reusable post-merge changelog enricher"
release_note: "New reusable workflow reusable-changelog-enrich.yml — thin CI wrapper around @acme-skunkworks/changelog-core enrich/finalise so every consumer fills post-merge changelog metadata (merged_at, commit, pr, stats, and version on npm) with its own GITHUB_TOKEN. Deploy callers use mode: enrich; npm callers use mode: finalise (enriches every merge, stamps version only on an untagged package.json / release-please cut). Writes only changelog/** with rebase-retry."
created_at: "2026-07-10T12:31:45Z"
merged_at:
branch: "a-793-phase-1-build-the-reusable-post-merge-changelog-enricher"
pr:
commit:
merge_strategy:
author: "rob@acmeskunkworks.io"
co_authors: []
category: feature
breaking: false
issues: ["A-793"]
stats:
  files_changed:
  loc_added:
  loc_removed:
  commits:
---

## Added

**`reusable-changelog-enrich.yml`** — the estate's post-merge changelog enricher
([A-793](https://linear.app/acme-skunkworks/issue/A-793)). Resolves the just-merged PR from the push SHA, runs
`pnpm exec changelog-core enrich` (and, for npm `mode: finalise`,
`changelog-core finalise` only when the package version is untagged), then
pushes **only** `changelog/**` with the repo's own `GITHUB_TOKEN`.

Install uses `--frozen-lockfile --ignore-scripts`. No dependency on a committed
or global changelog skill — `@acme-skunkworks/changelog-core` is the source of
truth. Callers own concurrency; the reusable never declares top-level
`concurrency:` ([A-621](https://linear.app/acme-skunkworks/issue/A-621)).

## Changed

- Document the workflow in `README.md` (catalogue, permissions matrix, deploy
  and npm caller stubs) and `CLAUDE.md` (layout + enrichment note).
