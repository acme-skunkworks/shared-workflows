---
title: Add a reusable fanned-payload validation check
release_note: 'New reusable workflow reusable-validate-payload.yml — the fan-out estate''s payload-exercising REQUIRED check. A skills consumer sets skills:true (every .claude/skills bundle must ship a named SKILL.md, and .claude/skills.lock must be valid JSON) and a CodeRabbit consumer sets coderabbit:true (.coderabbit.yaml must be present and parse). It closes the A-714 trap: the fan-out spine accepts mergeable_state unstable, so without a payload check a broken bundle stays green and merges (and, on octavo, deploys); this check turns that into blocked. Job name "Validate fanned payload", caller job id validate-payload, so the ruleset-pinnable context is "validate-payload / Validate fanned payload".'
created_at: '2026-07-07T17:58:21Z'
merged_at: '2026-07-07T18:15:58Z'
branch: a-738-provision-a-required-payload-exercising-check-on-every-fan
pr: 53
commit: 28fc4ed
merge_strategy: squash
category: feature
breaking: false
issues:
  - A-738
stats:
  files_changed: 2
  loc_added: 245
  loc_removed: 0
  commits:
---

## Added

**`reusable-validate-payload.yml`** — the estate's fan-out **payload-exercising
required check** (A-738). The release-orchestrator fan-out spine (A-711) accepts
`mergeable_state: unstable` as mergeable (so a skipped review bot doesn't wedge
the merge), which means a consumer whose only required checks are `GO/NO GO` and
`pr-title` has **nothing** validating the fanned payload — a broken
`.claude/skills/**` bundle or an unparseable `.coderabbit.yaml` reaches
`unstable` and the spine merges (and, on octavo, deploys) it. This closes that
[A-714](https://linear.app/acme-skunkworks/issue/A-714) trap.

What it validates is **declared by the caller**, never auto-detected (a required
check that silently skips an absent payload is the green-but-blind hole it exists
to close): `skills: true` requires every `.claude/skills/**` and
`.agents/skills/**` bundle to ship a SKILL.md whose frontmatter names the skill,
plus a valid-JSON `.claude/skills.lock`; `coderabbit: true` requires a present,
parseable `.coderabbit.yaml`. The validator is a zero-dependency inline Node
script (a broken payload must fail even if it also breaks the repo's own
toolchain); YAML parsing loads `js-yaml` via `npx`.

The job is named **`Validate fanned payload`** with caller job id
`validate-payload`, giving the stable, ruleset-pinnable context
`validate-payload / Validate fanned payload`. Consumers reference it at `@v1`;
the release-orchestrator caller-file adapter stamps the `skills`/`coderabbit`
inputs from each repo's fleet.json profile (A-738 rollout).
