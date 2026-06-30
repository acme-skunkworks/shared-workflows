# The `GO/NO GO` gate

The estate gates merges and releases on a single check-run named **`GO/NO GO`**: an
`if: always()` aggregator job that `needs:` every real CI job in a repo and concludes
`success` only when all of them passed (or legitimately skipped). Its **intrinsic
check-run is the gate** — the estate ruleset requires it (A-425, pinned to the GitHub
Actions integration so it cannot be forged) and the release orchestrator polls it before
squash-merging a release PR (A-419).

This is the per-repo pattern chosen in [ADR 0001 §5.4](adr/0001-shared-ci-architecture-for-npm-packages.md):
the gate lives in each consumer because only the repo itself can enumerate _all_ of its
jobs — the shared `reusable-*` workflows deliberately do **not** name themselves the gate.

The estate ruleset that requires this check is versioned in this repo under A-425.

`shared-workflows` dogfoods the pattern inline in its own
[`ci.yml`](../.github/workflows/ci.yml) (see `CLAUDE.md` for why this repo cannot consume
its own reusable workflows).

## Canonical aggregator

Add one job to your `ci.yml`, with `needs:` listing every real job (the shared callers
_and_ any local extras):

```yaml
go-no-go:
  name: GO/NO GO # job name = check_run.name = the required check
  needs: [lint, build-test, pr-title] # + any local jobs
  if: ${{ always() }} # MANDATORY — else it skips and the gate never reports
  runs-on: ubuntu-latest
  permissions:
    contents: read
  steps:
    - name: ⚖️ Verdict
      env:
        NEEDS_JSON: ${{ toJSON(needs) }}
      run: |
        set -euo pipefail
        echo "$NEEDS_JSON" \
          | jq -e 'to_entries | all(.value.result == "success" or .value.result == "skipped")' >/dev/null \
          || { echo "::error::a required check failed — refusing to go"; exit 1; }
        echo "All required checks passed — GO."
```

The aggregator is **granularity-agnostic**: it `needs:` whatever jobs the repo wired —
one coarse bundle or six fine ones — so it never forces the split (ADR 0001 §5.7).

## Footguns

- **Never path-filter the gate workflow.** A required workflow that is path-filtered sits
  `Pending` forever on a PR that touches none of its paths, and the merge blocks
  indefinitely. Keep the workflow hosting the aggregator unconditional.
- **`if: always()` is mandatory.** Without it the aggregator inherits the default
  "skip if any `needs:` failed" behaviour, so on a real failure it skips, the check-run
  is never minted, and the ruleset's required check never reports.
- **Treat `skipped` against an allowlist.** A path-skipped job (e.g. on `release-please--*`
  PRs that touch only changelog paths) is a legitimate non-failure, so `skipped` passes
  the verdict. If a repo has jobs that should _never_ skip, tighten the `jq` to allow
  `skipped` only for the expected job names rather than blanket-accepting it.
- **The check name is load-bearing — keep it exactly `GO/NO GO`.** The job `name:` becomes
  `check_run.name`; the ruleset and the orchestrator match that literal. Emoji, spaces,
  `&` and `/` all survive into the check-run name (proven by the legacy `🔬 Build & Lint`
  and verified here for `GO/NO GO`). If a future GitHub change ever mangles the `/`, fall
  back to **Option A**: mint the check-run explicitly with
  `POST /repos/{owner}/{repo}/check-runs` (`head_sha` = the PR head), which also needs
  `checks: write` and lets you attach custom annotations.

## Rollout

During migration the orchestrator **dual-accepts** both `🔬 Build & Lint` and `GO/NO GO`
(A-419), so a repo can adopt the aggregator without a flag day. Once every served repo
emits `GO/NO GO`, the orchestrator and the rulesets drop the old name (A-437).
