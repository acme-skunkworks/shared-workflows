---
title: Drop retired enrich-changelogs.yml references; document changelog-core enrich model
release_note: "Docs now describe the current release and enrichment model: the private release-orchestrator drives release-please as a kind:deploy target, post-merge enrichment runs in-repo via reusable-changelog-enrich.yml (@acme-skunkworks/changelog-core), and @v1 already carries the A-821 write-back (as of v1.5.0) so callers need no SHA pin."
created_at: "2026-07-13T16:14:00Z"
merged_at:
branch: a-949-docs-drop-retired-enrich-changelogsyml-references-document
pr:
commit:
merge_strategy:
author: rob@acmeskunkworks.io
co_authors: []
category: docs
breaking: false
issues:
  - A-949
stats:
  files_changed:
  loc_added:
  loc_removed:
  commits:
---

## Changed

**Docs corrected for the current changelog + release model ([A-949](https://linear.app/acme-skunkworks/issue/A-949))** —
the central `enrich-changelogs.yml` cron was retired in [A-801](https://linear.app/acme-skunkworks/issue/A-801) and the
[A-597](https://linear.app/acme-skunkworks/issue/A-597) release cutover is complete, but `CLAUDE.md`, `README.md` and
`changelog/README.md` still referenced the deleted in-repo `release-please.yml`
and the retired central cron.

- `CLAUDE.md` — replaced the future-tense "Phased rollout" note with the
  completed cutover: the private release-orchestrator drives release-please as a
  `kind: deploy` target and post-merge enrichment runs in-repo via
  `reusable-changelog-enrich.yml` (`mode: enrich`), not the retired
  `enrich-changelogs.yml` cron.
- `README.md` — documented that `@v1` includes the [A-821](https://linear.app/acme-skunkworks/issue/A-821) road-runner-bot
  write-back as of `v1.5.0`, so callers pin `@v1` with no SHA pin; fixed a broken
  link to the deleted `release-please.yml`.
- `changelog/README.md` — rewrote the release story to the orchestrator-driven
  model and replaced the stale "Phase B" note with the in-repo post-merge
  enrichment description.
- `.github/workflows/move-floating-major.yml` — rewrote the stale Phase A/B
  header comments (which still referenced the deleted in-repo `release-please.yml`
  and an unconfirmed first orchestrated release) to the completed-cutover reality.

## Fixed

**Self-host caller floated to `@v1`** — with the [A-821](https://linear.app/acme-skunkworks/issue/A-821) write-back now in
`@v1` (first shipped in `v1.5.0`), `.github/workflows/changelog-enrich.yml` no
longer needs its interim SHA pin. Floated it to `@v1` and dropped the
now-satisfied pin comment, so the repo dogfoods the `@v1` guidance it documents.
