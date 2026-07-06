# A-732 throwaway verification (do not merge)

This is a disposable PR that exercises the canonical `.coderabbit.yaml` merged in
PR #48. It verifies, against a `main` base that now carries the config:

- CodeRabbit still reviews an **unlabelled** PR, applying `en-GB` + the `chill`
  profile.
- `path_filters` excludes vendored/generated trees — the sibling `dist/` file in
  this PR should **not** appear in CodeRabbit's "Files selected for processing".
- Applying the `skip-review` label makes CodeRabbit skip the review (the denylist,
  read from the `main` base branch).

It will be closed without merging.

<!-- trigger re-review after label -->
