# Security policy

This repository is the estate's home for **reusable GitHub Actions workflows**
(`workflow_call`) and Layer-1 composite actions. Because a consumer's release job calls
`reusable-pkg-release.yml` with `id-token: write`, this repo sits inside every consumer's
**publish trust boundary** — the security posture below is load-bearing, not decorative.

The full design rationale lives in
[ADR 0001 §D5/§D6](docs/adr/0001-shared-ci-architecture-for-npm-packages.md); the pre-GA
review that signed this surface off is
[`docs/security-review-a422.md`](docs/security-review-a422.md) (A-422).

## Reporting a vulnerability

**Do not open a public issue for a security problem.** Report it privately via GitHub's
[**Report a vulnerability**](https://github.com/acme-skunkworks/shared-workflows/security/advisories/new)
flow (Security → Advisories), or email the maintainer at <hello@robeasthope.com>. Please
include the affected workflow/action, a description, and a reproduction or proof-of-concept
if you have one. You will get an acknowledgement, and we will coordinate a fix and
disclosure with you.

## Supported versions

Consumers pin the floating major tag (`@v1`), which always tracks the latest `v1.x.y`
release. Only the current major line receives security fixes; there is no back-porting to
superseded majors.

## Supply-chain posture

- **Assume-exfiltratable public CI.** Workflows are written as if any secret reaching a
  runner could leak. PR-triggered workflows take **no secrets** and never use
  `pull_request_target`; anything privileged (publishing, tagging, changelog write-back) is
  **push-to-`main`** or release-triggered only.
- **Third-party actions are SHA-pinned** to a full 40-hex commit with a `# vX.Y.Z` comment,
  and bumped only via reviewed weekly Dependabot PRs. This is enforced **org-wide** by the
  `sha_pinning_required` org policy — an unpinned third-party `uses:` is rejected at
  workflow start-up, in every repo.
- **Reusable-workflow calls float `@v1`.** Reusable-**workflow** references are tag-exempt
  under `sha_pinning_required`, so consumer caller stubs pin `@v1` and receive shared-code
  changes when the release orchestrator moves the `v1` tag onto a released commit. That
  moved code has already passed this repo's own `GO/NO GO` gate and review; consumers
  wanting change-review-per-bump may pin `@<sha>` instead. See ADR 0001 and A-662.
- **Least-privilege permissions.** Every `reusable-*.yml` sets root `permissions: {}` and
  grants each job only the scopes it needs. `id-token: write` is confined to the two publish
  legs of `reusable-pkg-release.yml` and the OIDC-using Claude workflows.
- **Typed secrets, guarded.** Reusable workflows declare **typed** `secrets:` inputs (not a
  blanket `secrets: inherit`) and fail fast with a clear message if a required secret is
  mapped but empty.

## The publish trust boundary

`reusable-pkg-release.yml` publishes to npm via **OIDC Trusted Publishing** (no `NPM_TOKEN`)
with `--provenance`, mirrors to GitHub Packages, and builds the tarball **once** so no
build-time code runs next to a mintable token. Two controls are the **consumer's**
responsibility, because a `workflow_call` callee cannot own them:

1. a branch-restricted `npm-release` **environment** in the consumer repo (if absent, GitHub
   silently auto-creates it _unprotected_, losing the ref gate); and
2. **no `workflow_dispatch`** on the caller — a dispatched run satisfies the same npm OIDC
   subject as a legitimate post-merge push and could publish a poisoned tarball with valid
   provenance.

npm Trusted Publishing validates the **caller's** `workflow_ref`, so each consuming package
registers its own caller filename as the npmjs.com Trusted Publisher.

## Branch protection

`main` is the only protected branch. Human and consumer-token changes never push to it
directly — they land via a squash-merged PR, gated on the integration-pinned `GO/NO GO`
check. The **one** exception is approved, gated App-identity automation running under the
road-runner-bot App (post-merge `changelog/**` write-back and release commits/tags), which
holds an explicit ruleset bypass because GitHub Actions' own `GITHUB_TOKEN` cannot be a
Trunk bypass actor (ADR 0004). See [`docs/go-no-go-gate.md`](docs/go-no-go-gate.md),
[`docs/rulesets.md`](docs/rulesets.md), and
[`docs/security-review-a422.md`](docs/security-review-a422.md) §4.
