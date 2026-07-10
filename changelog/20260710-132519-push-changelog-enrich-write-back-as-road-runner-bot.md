---
title: "Push changelog enrich write-back as road-runner-bot"
release_note: "reusable-changelog-enrich.yml now mints a road-runner-bot installation token and pushes changelog/** as that bot (ADR 0004), instead of github-actions[bot] / GITHUB_TOKEN. Callers must pass secrets: inherit so ROADRUNNER_PRIVATE_KEY reaches the job."
created_at: "2026-07-10T13:25:19Z"
merged_at:
branch: "a-821-phase-2-patch-reusable-changelog-enrich-to-push-as-road"
pr:
commit:
merge_strategy:
author: "rob@acmeskunkworks.io"
co_authors: []
category: fix
breaking: false
issues: ["A-821"]
stats:
  files_changed:
  loc_added:
  loc_removed:
  commits:
---

## Fixed

**`reusable-changelog-enrich.yml`** — write-back now authenticates as
`road-runner-bot[bot]` ([A-821](https://linear.app/acme-skunkworks/issue/A-821)).
Actions cannot be a Trunk bypass actor on this org (HTTP 422; ADR 0004 /
[A-794](https://linear.app/acme-skunkworks/issue/A-794)), so the job mints a
repo-scoped installation token via `actions/create-github-app-token` (same pin
as the orchestrator) from org var `ROADRUNNER_CLIENT_ID` + secret
`ROADRUNNER_PRIVATE_KEY`, then stages **only** `changelog/**` and pushes with
rebase-retry. App-token pushes re-fire workflows; the resolve step's "no
associated PR" path is the loop guard.

## Changed

- Caller stubs require `secrets: inherit`; job `GITHUB_TOKEN` scopes narrow to
  `contents: read` + `pull-requests: read` (write is on the App token).
- `README.md` and `CLAUDE.md` replace the stale "GITHUB_TOKEN / path-scoped
  Actions bypass" wording with ADR 0004 (road-runner-bot Trunk bypass) and
  document the org credential prerequisites for canary callers.
