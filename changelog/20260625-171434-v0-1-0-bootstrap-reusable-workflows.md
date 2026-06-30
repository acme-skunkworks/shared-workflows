---
title: "Bootstrap shared-workflows with the Claude pair + PR-title reusable workflows"
release_note: "First release: three reusable (workflow_call) workflows consumers SHA-pin to — interactive @claude, automated PR review, and a Conventional Commit PR-title gate."
version: "0.1.0"
created_at: "2026-06-25T17:14:34Z"
category: feature
breaking: false
issues: ["A-414"]
---

## Added

- **`reusable-claude.yml`** — interactive `@claude`, with the maintainer
  author-association gate (OWNER/MEMBER/COLLABORATOR) built in.
- **`reusable-claude-code-review.yml`** — automated Claude PR review, with the
  draft + `release-please--*` skips built in.
- **`reusable-validate-pr-title.yml`** — Conventional Commit PR-title check
  (canonical job name `Validate PR title is a Conventional Commit`).
- Repo scaffolding: weekly grouped Dependabot, the self-hosted hardened `@claude`
  - review dogfood workflows (`claude.yml`, `claude-code-review.yml`), the lean
    self-CI (`ci.yml`), and the husky hook set.

Stands up the repo as the estate's single home for reusable `workflow_call`
workflows, replacing the copy-paste-per-repo approach; no consumer repos are
touched.
