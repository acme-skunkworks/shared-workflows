---
title: Honour the skip-review label in Claude Code Review
release_note: The reusable Claude Code Review workflow now skips any PR carrying the `skip-review` label, matching CodeRabbit's existing behaviour — so a single label spares an automated PR from both review bots.
created_at: "2026-07-07T10:52:06Z"
merged_at:
branch: a-716-skip-claude-code-review-on-every-orchestrator
pr:
commit:
merge_strategy:
category: feature
breaking: false
issues:
  - A-716
stats:
  files_changed:
  loc_added:
  loc_removed:
  commits:
---

## Changed

- `reusable-claude-code-review.yml`'s job `if:` now also skips a PR that carries
  the `skip-review` label, alongside its existing draft / `release-please--` /
  dependabot skips. A single label therefore opts an automated PR out of **both**
  the CodeRabbit SaaS review and the Claude Code Review workflow, bringing Claude
  into parity with CodeRabbit's `!skip-review` denylist in the canonical
  `.coderabbit.yaml`.
- `claude-code-review.yml` (the inline copy that reviews this repo's own PRs)
  gains the same clause, kept in sync with the reusable per house style.
- `CLAUDE.md` documents the cross-bot behaviour.

This spares Claude review quota on the release-orchestrator's unattended,
auto-merging automation PRs — fan-out rollouts and changelog-enrichment PRs alike
— which carry no new behaviour. It fails safe: a forgotten label means more
review, not less.
