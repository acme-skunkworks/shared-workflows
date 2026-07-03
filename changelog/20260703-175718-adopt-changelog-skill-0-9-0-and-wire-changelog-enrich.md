---
title: "Adopt changelog skill 0.9.0 and wire changelog:enrich"
release_note:
created_at: "2026-07-03T17:57:18Z"
branch: a-675-adopt-changelog-0-9-0-enrich
category: chore
breaking: false
issues: ["A-675"]
---

## Changed

- Bumped the vendored `changelog` skill to **0.9.0**, which adds
  `scripts/enrich-changelog.mjs` — the deploy-target post-merge enricher — and
  exposed it as the `changelog:enrich` pnpm script. This lets the release
  orchestrator's enrichment cron fill this repo's post-merge changelog fields
  (`merged_at` / `commit` / `merge_strategy` / `pr` / `stats`) going forward, once
  shared-workflows is added to the enrich matrix ([A-675](https://linear.app/acme-skunkworks/issue/A-675) Step 4).
