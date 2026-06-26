# Composite actions (Layer 1)

The pick-and-mix primitives of the Acme Skunkworks shared-CI model. Each
directory here is a composite action (`action.yml`) consumed cross-repo and
SHA-pinned, e.g.:

```yaml
- uses: acme-skunkworks/shared-workflows/.github/actions/setup-project@<sha>
```

## Why a separate action layer (ADR 0001 §5.7)

GitHub Actions has a cost asymmetry that decides _where_ to split CI:

- A **reusable workflow** (`workflow_call`) is a **job** — its own runner and its
  own `pnpm install`. Splitting one workflow into N tiny ones costs N× setup.
- A **composite action** runs as **steps inside the calling job** — no extra
  runner, no extra install. Splitting here is essentially free, and a failing
  step still shows a red ✗ at step granularity.

At single-package scale setup dominates runtime, so **pick-and-mix lives at the
composite-action layer**; the reusable workflows (Layer 2) stay coarse and pay
setup once. A job calls `setup-project` once, then composes exactly the checks it
needs.

## The catalogue

| Action               | What it does                                                                     |
| -------------------- | -------------------------------------------------------------------------------- |
| `setup-project`      | pnpm + Node (from `.nvmrc`) + lockfile-keyed pnpm store cache, then install      |
| `eslint`             | ESLint over the repo (consumer's flat config + `@acme-skunkworks/eslint-config`) |
| `lint-markdown`      | markdownlint-cli2                                                                |
| `lint-yaml`          | yamllint (config injected from this repo, SK-438) + actionlint                   |
| `build`              | `pnpm run build` (verification build; published artefact rebuilt in release)     |
| `typecheck`          | `tsc --noEmit`                                                                   |
| `test-vitest`        | `vitest run` (unit tests)                                                        |
| `test-bats`          | bats (infrastructure tests)                                                      |
| `shellcheck`         | ShellCheck over tracked shell scripts                                            |
| `changelog-validate` | dated-changelog format + completeness                                            |

All except `setup-project` assume `setup-project` has run earlier in the same job
(they invoke locally installed tooling via `pnpm exec`, or `pnpm run` for the
package's own scripts as `build` does). `lint-yaml` and `shellcheck` are the
exceptions that fetch/use non-Node tools: yamllint (pip), actionlint (pinned
installer), and the runner's preinstalled ShellCheck.

## Conventions

- **Inputs** are sourced (in the reusable workflows) from each repo's
  `repo-config.yaml`; here they are plain typed `with:` inputs with sensible
  defaults.
- **Pinning** — every third-party `uses:`/installer is SHA-pinned with a
  `# vX.Y.Z` comment and bumped weekly by Dependabot. Tool versions (yamllint,
  actionlint, pnpm, Node) mirror this repo's own `ci.yml` so they never drift.
- **Permissions** — composite actions inherit the calling job's token scopes;
  they declare none. These actions sit inside the publish trust boundary (SK-422).

## Why this repo does not dogfood them via `uses:`

The org enforces `sha_pinning_required`. As with the inline workflows (see the
root `CLAUDE.md`), a local `uses: ./…` self-reference is not SHA-pinnable and a
cross-repo self-reference is circular before the first tagged SHA exists. So
`shared-workflows` authors these actions but does **not** consume them in its own
`ci.yml`; consumers reference them cross-repo by `@<sha>`, which is compliant.
Layer 2's `reusable-lint.yml` (SK-415) and `reusable-build-test.yml` (SK-416)
compose them.
