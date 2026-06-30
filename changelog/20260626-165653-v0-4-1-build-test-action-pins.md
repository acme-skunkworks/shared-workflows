---
title: "Fix reusable-build-test.yml action pins so the build lane resolves"
release_note: "Repoint reusable-build-test.yml's six Layer-1 pins to the SHA where the new build action exists, so a caller's build lane no longer fails to resolve."
version: "0.4.1"
created_at: "2026-06-26T16:56:53Z"
category: fix
breaking: false
issues: ["A-444"]
---

## Fixed

- **`reusable-build-test.yml` build lane was broken.** Its six Layer-1 `uses:`
  pins were left on the `132d746` pre-release placeholder, but the `build` action
  is new in the previous release and does not exist at that commit — a caller
  resolving the `build` step failed (action not found). Bumped all six pins to
  `d0a5949`, the first commit where `build` coexists with the other five, so the
  workflow is internally uniform and resolves. (`reusable-lint.yml` is
  untouched — its actions all predate the bump.)
