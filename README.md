# CoverOps GitHub Action

One step for the three CI questions CoverOps answers:

| `command` | Question | Exit |
|---|---|---|
| `select` | Which tests can this change not reach? | always 0; the answer is in the outputs |
| `verdict` | Did anything fail that is not in quarantine? | 0 green, 1 a real failure, 2 no reports |
| `gate` | Did the tests cover the code this change touched? | 0 pass, 1 fail, 2 misconfigured |

The action installs the CLI from your own CoverOps address
(`/v1/agents/artifact/cli`), so the CLI always matches the server it talks
to, in the cloud or on your own servers. It fetches the base branch itself;
the change is the commits between the pull request's base and HEAD.

## Use it

Reference it by its major version; `v1` moves with fixes, a full tag
(`v1.0.0`) pins one release:

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 0 }

      # pytest, JUnit 5 and TestNG select inside the runner; nothing else needed:
      - run: pytest --coverops-select --junitxml=reports/pytest.xml || true
        env:
          COVERAGEOPS_ENDPOINT: ${{ vars.COVEROPS_ENDPOINT }}
          COVERAGEOPS_TOKEN: ${{ secrets.COVEROPS_TOKEN }}
          COVERAGEOPS_BASE: origin/${{ github.base_ref }}

      # Fail the build on real failures only; quarantined tests still ran.
      - uses: melihsakarya/coverops-action@v1
        with:
          command: verdict
          endpoint: ${{ vars.COVEROPS_ENDPOINT }}
          token: ${{ secrets.COVEROPS_TOKEN }}
          junit: reports

      - uses: melihsakarya/coverops-action@v1
        with:
          command: gate
          endpoint: ${{ vars.COVEROPS_ENDPOINT }}
          token: ${{ secrets.COVEROPS_TOKEN }}
          comment: true   # one pull request comment, rewritten on every push
          args: --min-changed 80
```

With `comment: true` the gate writes one comment on the pull request and
rewrites it on every push: changed-code coverage, the changed lines no test
ran, and for each failed test what kind of red it is (code change, test
changed, environment, flaky…) with the commit and lines behind a code change.
The job needs `permissions: { pull-requests: write, statuses: write }`.
The commit status is posted as `coverops/changed-code-coverage`, and for now
also under the old name `coverlens/changed-code-coverage`, so branch
protection rules that require the old name keep working. Move them to the
new name, then set `COVERAGEOPS_STATUS_LEGACY=off` to stop the old one.

For any other runner, ask for the skip list and hand it to the runner:

```yaml
      - id: select
        uses: melihsakarya/coverops-action@v1
        with:
          command: select
          endpoint: ${{ vars.COVEROPS_ENDPOINT }}
          token: ${{ secrets.COVEROPS_TOKEN }}
          ignore: "docs/**,deploy/**"

      # outputs.mode is "subset" or "all"; outputs.skip is one suite::name per
      # line and empty when everything must run.
      - run: ./run-tests.sh --exclude-file <(echo "$SKIP")
        env:
          SKIP: ${{ steps.select.outputs.skip }}
```

Jest, Vitest, Playwright and Cypress skip inside the runner: write the list
to a file and point `COVERAGEOPS_SKIP_FILE` at it, with `@coverageops/jest`'s
environment, `@coverageops/vitest/setup`, `@coverageops/playwright`'s
`selectionSkip` or `@coverageops/cypress/support`'s `selectionSkip()` in place
(their READMEs):

```yaml
      - run: |
          printf '%s\n' "$SKIP" > "$RUNNER_TEMP/skip.txt"
          COVERAGEOPS_SKIP_FILE="$RUNNER_TEMP/skip.txt" npx playwright test
        env:
          SKIP: ${{ steps.select.outputs.skip }}
```

GitHub Enterprise Server without access to github.com: copy `action.yml` to
`.github/actions/coverops/action.yml` in your repository and use
`uses: ./.github/actions/coverops` instead; it is the same file.

On a `pull_request` run, `actions/checkout` checks out the merge commit
GitHub makes (`refs/pull/N/merge`). That commit is in no branch, so the CLI
and the runner plugins measure the change to the pull request's own head
instead: its commits since the merge base with the base branch.

## Observe before you skip

`observe: true` asks the same question, records the answer under the run
key (the workflow run id) and skips nothing: `outputs.skip` stays empty and
`outputs.mode` is `observe`. The pytest plugin (`--coverops-observe`) and the
JUnit/TestNG agent (`COVERAGEOPS_SELECT=observe`) do the same inside the
runner. When the run's results arrive, the dashboard's **Test selection**
page shows, measured on that run, whether any test selection would have
skipped failed and how much test time skipping would have saved. Run it for
a while, until the window holds runs that failed for real reasons, before
acting on the skip list.

## A time budget

When even the safe selection is too slow for the pipeline, give it a budget:

```yaml
      - uses: melihsakarya/coverops-action@v1
        with:
          command: select
          endpoint: ${{ vars.COVEROPS_ENDPOINT }}
          token: ${{ secrets.COVEROPS_TOKEN }}
          time: 15m        # or target: 60%  (of the safe selection's own time)
```

What is never cut: tests that failed recently, whose own file changed, that
have no footprint, or that are new. Then the floor: the cheapest set of
impacted tests that still reaches every changed line any footprint reaches.
The rest of the impacted tests fill the budget by risk (failure history 50%,
share of the change reached 35%, deferrals since the test last ran 15%), and
what does not fit is skipped with reason `over_budget`. A test deferred five
times since it last ran joins the floor, so the same tests cannot be put off
forever. If the floor alone is over the budget it runs anyway, and the step
summary says so. With `observe: true` as well, the Test selection page shows
which failures only the budget would have missed.

## What it saved

Every select call that carries the run key (the action sends the workflow
run id) is recorded. The dashboard's **Test selection** page then shows, per
calendar month, the test time the runs that acted on the answer saved: each
skipped test priced at its own average measured duration in that month and
the 30 days before, against the measured time of what the runs actually ran
(share = saved / (ran + saved)). Time saved by the safe selection and time
saved by a budget are shown apart, next to the same months' observation
recall. Runs whose results never arrived and tests never measured are left
out rather than estimated. It is test time, not wall-clock: tests that run in
parallel still count one by one.

## What may be skipped

Only tests whose recorded footprint misses every changed line. A test written
since the last run, a test with no footprint, a test whose own file changed
and a test that failed recently always run. When the change cannot be read in
full, or touches code no agent measures, `mode` is `all` and the step summary
says why.

A periodic full run keeps selection honest: set it on the dashboard's Test
selection page (every N selections, and/or at least every H hours). When one
is due, `mode` is `all` with reason `scheduled_full_run`; footprints are
refreshed and anything selection would have skipped is seen to pass or fail.
It counts the answers runners record with their run key, and leaves
observation mode alone.

Code no test has executed yet is the one place footprints are silent. When
the project has a code map (built from the source the agents upload) and it
places every such file, the tests that run the files depending on it run
instead of everything (reason `graph_reach`). The map only ever adds tests;
a file it cannot place, or whose dependents no agent measures, still means a
full run.

The answer and the pull request comment also name the user flows the change
reaches: the entry routes (`POST /api/orders`) of the recorded requests that
ran the changed code, with how many tests went through each.

Inputs reach the scripts as environment variables, never as script text, so a
branch name or an input value cannot run as a command.

## License

This action (`action.yml` and this README) is licensed under the
[Apache License 2.0](LICENSE). The CoverOps CLI it installs and the CoverOps
service are covered by their own terms. "CoverOps" is a trademark; the license
does not grant use of it (Apache-2.0, section 6).
