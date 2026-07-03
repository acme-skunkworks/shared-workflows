# Changelog

One Markdown file per release, capturing what changed and why. This is the
curated, per-release, machine-readable history of `shared-workflows` — there is
no root `CHANGELOG.md`.

Unlike the estate's npm packages (`eslint-config`, `agent-skills`), this repo
**publishes no npm package**: a "release" here is a **git tag (`vX.Y.Z`) + a
GitHub release** over the reusable workflows and composite actions, nothing more.
Each entry carries a `version` tying it to the tag it shipped in, and the GitHub
release notes are sourced from the matching entry's body.

**Releases are automated (A-597).** On merge to `main`,
[`release-please.yml`](../.github/workflows/release-please.yml) maintains a
release PR from the Conventional-Commit history; merging it bumps
[`package.json`](../package.json) + [`.release-please-manifest.json`](../.release-please-manifest.json)
and cuts the annotated `vX.Y.Z` tag and its GitHub Release. Consumers pin their
callers to `@v1` (A-662), so
[`move-floating-major.yml`](../.github/workflows/move-floating-major.yml) then
force-moves the floating major tag onto the release commit — otherwise the
release would never reach `@v1` consumers.

`v1` is a **lightweight** tag: on a perpetually-moving ref an annotated tag's
embedded tagger/timestamp/message only reflect the last force-move, not a release,
so they mislead. The permanent `vX.Y.Z` tags stay annotated — that is where the
release audit trail lives.

A breaking change ships as `v2` (a new floating major) and must **not** move `v1`;
`move-floating-major.yml` handles this — it only ever touches the release's own
major tag.

> **Phase B (A-597):** release-please will be driven by the private
> release-orchestrator (shared-workflows joins as a `kind: deploy` target, like
> octavo), which also enriches these entries post-merge with the `merged_at` /
> `commit` / `stats` fields. Until that cutover, the in-repo `release-please.yml`
> above drives releases.

The entries up to and including `v1.0.0` were **hand-authored as a backfill**
(A-585), reconstructing the project's history from its merged PRs. Forward entries
follow the same schema.

## File naming

```text
changelog/YYYYMMDD-HHMMSS-<slug>.md
```

- Timestamp is UTC and matches `created_at` in the frontmatter (for the backfill,
  the merge time of the release-triggering PR).
- Slug: lowercase, non-alphanumerics replaced with `-`, repeats collapsed,
  ~60-char cap on a word boundary.

## Frontmatter schema

Entries are authored by the installed [`changelog`](../.claude/skills/changelog/)
skill and validated by its `validate-changelog.mjs`. The authoritative contract
is [`changelog-contract.md`](../.claude/skills/changelog/references/changelog-contract.md);
the schema below is the local summary.

```yaml
---
title: "Concise summary of the change"
release_note: "One-sentence user-facing summary" # optional; string or null
created_at: "2026-06-30T10:49:15Z" # set once; never overwritten
merged_at: # post-merge; filled by the release orchestrator
branch: "a-597-feature-slug" # stable lookup key for enrichment
pr: # filled at /send-it time once the PR exists
commit: # 7-char merge SHA; post-merge
merge_strategy: # squash | merge | rebase; post-merge
version: # vX.Y.Z (no v) the release cut; blank on non-releasing entries
author: "you@example.com"
co_authors: []
category: feature # feature | fix | chore | docs | refactor | perf
breaking: false
issues: ["A-418"] # Linear issue IDs
affected_packages: [] # this repo ships no packages — stays empty
stats: # post-merge; filled by enrichment
  files_changed:
  loc_added:
  loc_removed:
  commits:
---
```

### Required fields

`title`, `created_at`, `category`, `breaking`. Everything else is optional
(validated by type when present). Forward entries carry the richer fields above;
the **post-merge** fields (`merged_at` / `commit` / `merge_strategy` / `stats`,
and `version` on releasing entries) are left blank at authoring time and filled
later. The hand-authored backfill entries (up to `v1.0.0`) use the original
slimmer schema and stay valid under the lenient required set.

> **Note on timestamps:** wrap ISO 8601 timestamps in quotes
> (`"2026-06-30T10:49:15Z"`). Unquoted timestamps are auto-parsed by YAML into
> Date objects; quoting keeps them as exact strings.

### Categories

| Category   | When to use                                     |
| ---------- | ----------------------------------------------- |
| `feature`  | New / expanded consumer-facing capability       |
| `fix`      | Bug fix in a shipped workflow or action         |
| `chore`    | Tooling, build, dependency bumps                |
| `docs`     | Documentation-only change                       |
| `refactor` | Internal restructuring with no behaviour change |
| `perf`     | Performance improvement                         |

The **product surface** is what determines the category and whether a change is
release-triggering: the `reusable-*.yml` workflows and `.github/actions/*`
composite actions consumers use, plus their reference docs. Changes confined to
this repo's own infrastructure (the self-host `ci.yml`/`claude*.yml` dogfooding,
`.claude`/`.agents` skill bundles, lint configs, hooks) are not release-worthy
and are folded into the next release's body.

If `breaking: true`, the body MUST lead with a `## Breaking` section describing
the change and the migration path.

## Body structure

```markdown
## Breaking <!-- only when breaking: true -->

- Description and migration steps

## Added

- Description

## Changed

- ...

## Fixed

- ...
```

Only include the `Added` / `Changed` / `Fixed` headings that have content.

## Versioning rules

Versions are derived per-release from the **true semantic impact** of the change
(read from the diff, not the PR title — Conventional-Commit discipline was loose
early on). Pre-1.0:

- `feature` → minor bump (`0.1.0 → 0.2.0`)
- `fix` → patch bump (`0.4.0 → 0.4.1`)
- breaking change pre-1.0 → minor bump (stays in `0.x`)
- `1.0.0` was a deliberate "graduate to stable public surface" milestone, not an
  auto-derived bump.
