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

Put this concurrency block on the workflow that hosts the aggregator (not inside a
`reusable-*` callee — `concurrency` is caller-level and a `workflow_call` cannot set
it for you):

```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  # Never cancel a RUNNING run — see Footguns. Cancelling only pending runs is safe:
  # a pending run materialises no jobs and therefore no check-runs.
  cancel-in-progress: false
```

Then add one job to your `ci.yml`, with `needs:` listing every real job (the shared
callers _and_ any local extras):

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
- **Concurrency must never cancel a running gate run (A-1100).** With
  `cancel-in-progress: true`, a superseded run's `needs` all report `cancelled`; the
  `always()` aggregator still executes; and the verdict — which allows only `success` or
  `skipped` — mints a **failure** check-run for a run that never failed. Set
  `cancel-in-progress: false` on the workflow that hosts `GO/NO GO`. Concurrency can then
  only cancel a _pending_ run, which has materialised no jobs and therefore mints no
  check-runs at all. Cost is bounded — GitHub keeps at most one pending run per group and
  collapses the rest, so a burst of four events costs two full runs, not four. This also
  subsumes the narrower release-branch carve-out
  (`github.head_ref != 'release-please--branches--main'`, A-961): `false` covers every
  branch, not just `release-please--*`.
- **`!cancelled()` on the aggregator is a gate bypass, not a fix.** A `skipped` check-run
  is [treated as success](https://docs.github.com/en/pull-requests/reference/status-checks)
  for required status checks, and cancelling a run needs only `actions: write` — which
  every write-access collaborator holds. Skipping the aggregator on cancellation would
  therefore let anyone merge failing code by clicking "Cancel workflow", forging the very
  check the A-425 integration pin exists to make unforgeable. Keep `cancelled` out of the
  verdict allowlist for the same reason: once concurrency cannot produce it, it can only
  mean a genuine manual cancel or a job timeout, both of which must stay red.
  (`cancelled()` is also
  [documented-unreliable on supersession](https://github.com/actions/runner/issues/3041).)
- **The gate must never go green off a run that skipped its real CI.** That principle
  rules out other tempting shortcuts — for example scoping the `edited` trigger so only
  some jobs re-run, which would leave the aggregator seeing all-`skipped` and going green
  with no CI at all. Tightening the blanket `skipped` allowlist is tracked separately
  (A-1103).
- **Treat `skipped` against an allowlist.** A path-skipped job (e.g. on `release-please--*`
  PRs that touch only changelog paths) is a legitimate non-failure, so `skipped` passes
  the verdict. If a repo has jobs that should _never_ skip, tighten the `jq` to allow
  `skipped` only for the expected job names rather than blanket-accepting it.
- **The check name is load-bearing — keep it exactly `GO/NO GO`.** The job `name:` becomes
  `check_run.name`; the ruleset and the orchestrator match that literal. Spaces and `/`
  survive verbatim into the check-run name (`GO/NO GO` keeps both). If a future GitHub
  change ever mangles the `/`, fall
  back to **minting the check-run explicitly** with
  `POST /repos/{owner}/{repo}/check-runs` (`head_sha` = the PR head), which also needs
  `checks: write` and lets you attach custom annotations.

## Canonical shape vs consumer drift

The snippet above is the reference shape: pass `needs` through `env: NEEDS_JSON` (never
interpolate `${{ toJSON(needs) }}` into the shell body — job names would otherwise be a
template-injection surface), use `set -euo pipefail`, and restate
`permissions: contents: read` on the aggregator job.

Some template-lineage consumers still diverge (inline `toJSON` interpolation, missing
`set -euo pipefail` and/or job-level `permissions:`) even while their comments claim to
follow this reference verbatim. Closing that drift is a per-repo fan-out — it cannot live
in a reusable workflow, for the same reason the gate itself cannot.

## Rollout

The migration is complete: the orchestrator and every served repo's ruleset require the
`GO/NO GO` check-run only. A-419 opened a dual-accept window (`🔬 Build & Lint` **or**
`GO/NO GO`) so repos could adopt the aggregator without a flag day; A-596 collapsed it to
`GO/NO GO`-only once every served repo emitted it, and A-437 removed the old name from the
rulesets. The `🔬 Build & Lint` job, where it still exists, survives as ordinary CI feeding
the aggregator's `needs:` — it is no longer a gate.
