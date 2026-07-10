---
title: Make validate-payload caller repo-owned
release_note: >-
  The dogfood validate-payload.yml caller is now repo-owned onboarding state
  (A-780), pinned at @v1 — not overwritten by the release-orchestrator fan-out.
  Consumers place the stub once; Dependabot owns bumps. Enables revoking
  workflows:write from road-runner-bot.
created_at: "2026-07-10T15:42:47Z"
merged_at:
branch: a-780-make-validate-payload-repo-owned-one-time-onboarding-revoke
pr:
commit:
merge_strategy:
category: chore
breaking: false
issues:
  - A-780
stats:
  files_changed:
  loc_added:
  loc_removed:
  commits:
---

## Changed

**Repo-owned `validate-payload.yml`** — drop the VENDORED fan-out header, pin
the reusable at `@v1`, and document in the README that consumers place this
caller once at onboarding ([A-780](https://linear.app/acme-skunkworks/issue/A-780)). The status context
`validate-payload / Validate fanned payload` is unchanged so existing rulesets
keep matching. The release-orchestrator no longer vendors this file; that lets
`road-runner-bot` drop `workflows: write`.
