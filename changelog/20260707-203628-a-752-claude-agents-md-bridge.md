---
branch: a-752-claude-agents-md-bridge
title: add @AGENTS.md import bridge (A-752)
category: docs
breaking: false
issues:
  - A-752
created_at: '2026-07-07T20:36:28Z'
merged_at: '2026-07-07T20:53:26Z'
commit: fbaafb4
pr: 58
merge_strategy: squash
stats:
  loc_added: 27
  loc_removed: 13
  files_changed: 2
version:
release_note: ''
---

## Changed

- Add the `@AGENTS.md` import bridge to `CLAUDE.md` so Claude Code loads the fanned shared block at session start
- Remove the duplicated British English section now covered by upstream `AGENTS.md`
