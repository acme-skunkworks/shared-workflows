---
title: "Roll out in-repo post-merge changelog enricher"
release_note: "shared-workflows now fills post-merge changelog metadata in-repo via changelog-enrich.yml (mode: enrich, secrets: inherit), adopting @acme-skunkworks/changelog-core and dropping the cron-era changelog:enrich script."
created_at: "2026-07-10T14:27:59Z"
merged_at:
branch: "a-800-phase-3-roll-out-in-repo-enricher-to-shared-workflows-deploy"
pr:
commit:
merge_strategy:
author: "rob@acmeskunkworks.io"
co_authors: []
category: chore
breaking: false
issues: ["A-800"]
stats:
  files_changed:
  loc_added:
  loc_removed:
  commits:
---

## Changed

**In-repo enricher ([A-800](https://linear.app/acme-skunkworks/issue/A-800))** — this deploy target now calls
`reusable-changelog-enrich.yml` from `.github/workflows/changelog-enrich.yml` on
push to `main` (`mode: enrich`, `secrets: inherit`), pinned at the [A-821](https://linear.app/acme-skunkworks/issue/A-821) merge
SHA until `@v1` moves. Trunk bypass for `road-runner-bot` and selected access to
`ROADRUNNER_*` credentials are applied so write-back can land on protected
`main` (ADR 0004).

- Add `@acme-skunkworks/changelog-core` and point `validate:changelog` at it;
  drop the cron-era `changelog:enrich` script.
- CI `📓 Changelog` installs deps and runs `pnpm run validate:changelog`.
- `CLAUDE.md` records the self-host caller.
