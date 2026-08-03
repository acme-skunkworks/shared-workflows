---
title: Give the GitHub Packages mirror its own publish gate
release_note: A GitHub Packages mirror failure can now be retried by re-running the release run — previously the mirror job was skipped for good once the npm leg had tagged the release, silently leaving the version on npm but never on GitHub Packages.
created_at: '2026-08-03T11:06:30Z'
merged_at: '2026-08-03T11:20:39Z'
branch: a-457-pkg-release-decouple-the-github-packages-mirror-from-the-npm
pr: 92
commit: 2cf5a7d
merge_strategy:
author: hello@robeasthope.com
co_authors: []
category: fix
breaking: false
issues:
  - A-457
stats:
  files_changed: 3
  loc_added: 198
  loc_removed: 27
  commits:
---

## Fixed

- `reusable-pkg-release.yml`'s GitHub Packages mirror gated on
  `needs.release.outputs.should_publish` — the version-vs-tag check — but the
  `release` job **creates that tag**. Once the npm leg had gone green the gate
  read false for that version for ever, so a mirror-only failure could never be
  retried: the job was **skipped**, and a skip is not a failure, so
  `notify-failure` stayed quiet too. npm held the version, GitHub Packages never
  did, and nothing said so. The mirror now gates on a separate `mirror_publish`
  output: `should_publish` **or** "the release tag exists and points at this
  run's commit" — precisely a re-run of the release run, which preserves
  `GITHUB_SHA` (the tag is cut with `--target "$GITHUB_SHA"`).

- The mirror job now probes GitHub Packages for the version **before**
  attesting, and gates both the attestation and the publish on that result.
  Previously `actions/attest-build-provenance` ran unconditionally, so any
  replay of an already-successful leg minted a throwaway Sigstore attestation.

- `⬆️ Upload tarball artifact` sets `overwrite: true`. `upload-artifact` fails on
  a name collision by default, and "Re-run all jobs" replays the build job into a
  run that already holds an `npm-tarball` from the first attempt — so the
  documented re-run-all recovery path died at the build step regardless of the
  publish gate.

## Changed

- The `notify-failure` runbook now spells out both retry routes ("Re-run failed
  jobs" within the tarball's 24-hour retention, or "Re-run all jobs" at any
  point) and states plainly that pushing a new commit is **not** a retry.

`mirror_publish` deliberately does not fire on a later push to the release
branch. HEAD has moved on by then, so re-packing it would publish _different
bits_ under an already-published version — desyncing the two registries by
content rather than by presence, which is worse than the gap being repaired.

Deferring tag creation until after the mirror — the fix originally proposed on
[A-457](https://linear.app/acme-skunkworks/issue/A-457) — was rejected. The tag's existence is what flips a merged release PR from
`autorelease: pending` to `autorelease: tagged`, and until that flip
`release-please release-pr` aborts for the repo and opens no further release
PRs. A mirror hiccup would have stalled the package's releases entirely rather
than leaving one version unmirrored.
