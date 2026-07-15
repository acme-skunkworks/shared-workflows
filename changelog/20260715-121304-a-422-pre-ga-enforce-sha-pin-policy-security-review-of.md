---
title: Pre-GA security review sign-off and reusable-workflow hardening
release_note: >-
  reusable-changelog-enrich.yml now fails fast with a clear message when
  ROADRUNNER_PRIVATE_KEY is mapped but empty, instead of dying deep inside JWT
  signing and silently skipping the changelog write-back.
created_at: "2026-07-15T12:13:04Z"
branch: a-422-pre-ga-enforce-sha-pin-policy-security-review-of-shared
author: rob@acmeskunkworks.io
co_authors: []
category: fix
breaking: false
issues:
  - A-422
merged_at: ""
commit: ""
pr:
stats:
  loc_added:
  loc_removed:
  files_changed:
---

## Changed

- `reusable-changelog-enrich.yml` gains the [A-646](https://linear.app/acme-skunkworks/issue/A-646) empty-token guard the two
  Claude workflows already carry: a `🚦 Guard` step fails fast when
  `ROADRUNNER_PRIVATE_KEY` is mapped but empty (a `required: true` secret only
  forces the caller to map it, not to supply a non-empty value), rather than
  failing opaquely during App-token minting and silently skipping the write-back
  ([A-422](https://linear.app/acme-skunkworks/issue/A-422)).
- `reusable-validate-payload.yml` re-pins `actions/checkout` from the stale
  `v4.2.2` to the estate-current `v7.0.0` used by every other workflow — a
  SHA-pin consistency fix.

## Added

- The pre-GA security review sign-off ([`docs/security-review-a422.md`](../docs/security-review-a422.md)):
  a review of the publish path, the cross-boundary secrets flow, third-party
  SHA-pinning, and the go/no-go ruleset, with a conscious sign-off of the
  floating-`@v1` auto-propagation into each consumer's publish trust boundary,
  plus confirmation that `sha_pinning_required` is enforced org-wide.
- A public-facing [`SECURITY.md`](../SECURITY.md) — the supply-chain posture and
  vulnerability-reporting channel the repo previously lacked.

No consumer-facing change beyond the fail-fast guard: callers still float the
reusable workflows at `@v1`.
