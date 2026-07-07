---
title: Finalise the canonical .coderabbit.yaml with estate review settings
release_note: The canonical .coderabbit.yaml is now a full estate standard (British-English review prose, a chill review profile, vendored/generated path filters and estate-policy path instructions) rather than just the skip-review denylist, ahead of its verbatim fan-out to consumers.
created_at: '2026-07-06T22:18:09Z'
merged_at: '2026-07-06T22:29:29Z'
branch: a-732-finalise-canonical-coderabbit-yaml
pr: 48
commit: 34baaf5
merge_strategy: squash
category: chore
breaking: false
issues:
  - A-732
stats:
  files_changed: 3
  loc_added: 109
  loc_removed: 7
  commits:
---

## Changed

- The canonical `.coderabbit.yaml` grows from the `skip-review` denylist into the
  estate's full CodeRabbit standard, ahead of the [A-712](https://linear.app/acme-skunkworks/issue/A-712)
  verbatim fan-out. It now also sets `language: en-GB` (British-English review
  prose), an explicit `reviews.profile: chill` (balanced signal-to-noise),
  `path_filters` excluding vendored/generated trees (`pnpm-lock.yaml`, `dist`,
  `node_modules`, and the re-vendored `.claude/skills` + `.agents/skills`
  bundles), and `path_instructions` encoding estate policy as reviewer guidance
  (British English for Markdown prose; SHA-pinning and the
  no-top-level-`concurrency:` rule for `.github/**`).
- `CLAUDE.md`'s _CodeRabbit review_ section documents the fuller config.

The three skip conventions (`!skip-review`, the `enrich entry for` title skip and
the `road-runner-bot[bot]` username skip) are preserved verbatim so the fan-out
can never regress a consumer. Dependabot PRs stay reviewed by design
([A-732](https://linear.app/acme-skunkworks/issue/A-732)).
