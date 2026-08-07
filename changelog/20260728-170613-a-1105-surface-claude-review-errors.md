---
title: Surface the real error when Claude Code Review fails
release_note: A failed Claude review now reports the SDK's actual error message instead of an opaque "result is_error:true", and the reusable workflow requests actions:read so the github_ci MCP server installs.
created_at: "2026-07-28T17:06:13Z"
merged_at: "2026-07-28T17:44:52Z"
branch: a-1105-surface-claude-review-errors
pr: 90
commit: d992357
merge_strategy:
author: hello@robeasthope.com
co_authors: []
category: feature
breaking: false
issues:
  - A-1105
stats:
  files_changed: 3
  loc_added: 181
  loc_removed: 0
  commits:
---

## Added

- A failure-only step that reads the named result fields back out of
  `claude-execution-output.json` and re-emits them as a `::error::` annotation.
  `claude-code-action` runs the SDK with `show_full_output: false` (the secure
  default), so, until now, every SDK failure surfaced as nothing but
  `Claude execution failed: result is_error:true` — the real message was written
  to the execution log and never read back. The step extracts only the result
  record's scalar fields and its `result`/`error` message, truncated; it
  deliberately does **not** enable `show_full_output` or dump the file, both of
  which would also expose the prompt and the PR diff.

## Fixed

- The reusable workflow now requests `actions: read`. `claude-code-action` offers
  Claude a `github_ci` MCP server (`get_ci_status`, `get_workflow_run_details`,
  `download_job_log`) and keeps those tools in `--allowedTools` unconditionally,
  but without the scope it logged that the `github_ci` MCP server requires
  `actions: read` permission and skipped CI server installation on every run —
  leaving Claude holding tool names backed by no server. Consumers must add the
  same scope to their caller stub, since a called reusable workflow's job cannot
  hold more than the caller's token grants.

A-1105 was diagnosed as a GitHub App installation-token failure, on the strength
of a zero-byte `curl` body in the log. That `curl` is the action's _post-job_
token revocation (`DELETE /installation/token`), where an empty body is the
correct response and which runs only after the job has already failed. The actual
cause on `release-orchestrator` was an expired `CLAUDE_CODE_OAUTH_TOKEN`, whose
signature — `num_turns: 1`, `total_cost_usd: 0` — was invisible for eight days
precisely because of the gap this entry closes.
