# Changelog

One Markdown file per release, capturing what changed and why. This is the
curated, per-release, machine-readable history of `shared-workflows` — there is
no root `CHANGELOG.md`.

Unlike the estate's npm packages (`eslint-config`, `agent-skills`), this repo
**publishes no npm package**: a "release" here is a **git tag (`vX.Y.Z`) + a
GitHub release** over the reusable workflows and composite actions, nothing more.
Each entry carries a `version` tying it to the tag it shipped in, and the GitHub
release notes are sourced from the matching entry's body.

**Move the floating `v1` tag on every release.** Consumers pin their callers to
`@v1` (A-662), so after tagging a new `vX.Y.Z` and cutting its GitHub release,
force-move the annotated `v1` tag onto the same commit and push it, or the release
never reaches `@v1` consumers:

```bash
git tag -f -a v1 <release-commit> -m "Floating major tag: track latest v1.x"
git push -f origin v1
```

A breaking change ships as `v2` (a new floating major) and must **not** move `v1`.
This step is manual until A-597 automates the release flow.

Also **bump the "current release" pointer and full-history range in
[`README.md`](../README.md)'s Versioning section** to the new `vX.Y.Z` — the
routinely-forgotten step that leaves the docs pointing at an old release.

The entries up to and including `v1.0.0` were **hand-authored as a backfill**
(A-585), reconstructing the project's history from its merged PRs — there was no
`/send-it`/changelog tooling in this repo when the work landed. New entries, if
the repo later adopts a forward flow, would follow the same schema.

## File naming

```text
changelog/YYYYMMDD-HHMMSS-<slug>.md
```

- Timestamp is UTC and matches `created_at` in the frontmatter (for the backfill,
  the merge time of the release-triggering PR).
- Slug: lowercase, non-alphanumerics replaced with `-`, repeats collapsed,
  ~60-char cap on a word boundary.

## Frontmatter schema

```yaml
---
title: "Concise summary of the release"
release_note: "One-sentence user-facing summary" # optional; string or null
version: "0.7.0" # semver; the tag this entry shipped in
created_at: "2026-06-30T10:49:15Z" # set once; never overwritten
category: feature # feature | fix | chore | docs | refactor | perf
breaking: false
issues: ["A-418"] # Linear issue IDs
---
```

### Required fields

`title`, `created_at`, `category`, `breaking`. Everything else is optional
(validated by type when present). `version` is present on every released entry.

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
