---
title: Record the A-1100 concurrency fix in the GO/NO GO gate docs and dogfood
release_note: The canonical GO/NO GO gate docs now forbid cancelling a running gate run, and this repo's own ci.yml dogfoods cancel-in-progress: false so a superseded CI run can no longer mint a false-red GO/NO GO check.
created_at: "2026-08-04T08:23:39Z"
merged_at:
branch: a-1101-go-no-go-concurrency-footgun
pr:
commit:
author: rob@acmeskunkworks.io
co_authors: []
category: fix
breaking: false
issues:
  - A-1101
stats:
  files_changed:
  loc_added:
  loc_removed:
  commits:
---

## Fixed

- **`.github/workflows/ci.yml` ([A-1101](https://linear.app/acme-skunkworks/issue/A-1101))** —
  sets `cancel-in-progress: false`. Previously, when concurrency superseded an
  in-flight run, every job in it reported `cancelled`; the `always()` aggregator
  still executed; and the verdict that allows only `success` or `skipped` minted
  a **failure** check-run for a run that never failed. Concurrency can now only
  cancel a _pending_ run, which has materialised no jobs and therefore mints no
  check-runs at all. This also subsumes [A-961](https://linear.app/acme-skunkworks/issue/A-961)
  for this repo — the narrower release-branch carve-out is unnecessary once
  `false` covers every branch.

## Changed

- **`docs/go-no-go-gate.md`** records three new footguns beside the existing
  [A-418](https://linear.app/acme-skunkworks/issue/A-418) set: concurrency must
  never cancel a running gate run; `!cancelled()` on the aggregator is a bypass
  rather than a fix; and the gate must never go green off a run that skipped its
  real CI. The canonical snippet now includes the safe concurrency block, and a
  short note documents where template-lineage consumers still drift from the
  reference verdict shape (`env: NEEDS_JSON`, `set -euo pipefail`, job-level
  `permissions:`).

## Notes

`!cancelled()` on the aggregator was deliberately **not** taken. A `skipped`
check-run counts as success for required status checks, and cancelling a run
needs only `actions: write`, so skipping the gate on cancellation would let
anyone merge failing code by clicking "Cancel workflow". `cancelled` stays out
of the verdict allowlist for the same reason.
