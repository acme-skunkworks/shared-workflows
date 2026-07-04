---
title: Fix orchestrated deploy-target releases never cutting a tag/Release
release_note: Orchestrated releases of this repo now cut the `vX.Y.Z` tag and GitHub Release automatically, instead of silently producing nothing on merge.
created_at: '2026-07-03T19:46:03Z'
branch: a-677-deploy-target-release-title-pattern
category: fix
breaking: false
issues:
  - A-677
merged_at: '2026-07-03T19:42:59Z'
commit: e4711bf
merge_strategy: squash
pr: 42
stats:
  loc_added: 36
  loc_removed: 1
  files_changed: 3
---

## Fixed

- Added `group-pull-request-title-pattern` to `release-please-config.json` so the
  combined release PR is titled `chore: release shared-workflows ${version}`
  instead of the versionless default `chore: release main`. With
  `separate-pull-requests: false`, release-please titles the combined PR from its
  _group_ pattern; the default carries no `${version}`/`${component}`, so the
  orchestrator's `release-please github-release` step could not parse the merged
  title and cut **no** tag and **no** GitHub Release — leaving
  `move-floating-major` inert and forcing a manual five-artefact finish (as on
  v1.1.1). The pattern change makes the title round-trip; tags stay bare
  `vX.Y.Z` (`include-component-in-tag: false` is untouched). The same key is
  required in every `kind: deploy` target and is now documented as an onboarding
  prerequisite ([A-677](https://linear.app/acme-skunkworks/issue/A-677)).
