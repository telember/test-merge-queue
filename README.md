# test-merge-queue

Lab for the **test-app merge train**. Nothing here is production code.

## What is being tested

`test-app` merges are already strictly serialised by `strict_required_status_checks_policy`,
but nothing decides *whose turn is next*. A gate run takes ~8 minutes and merges arrive every
~17, so a pull request can go green and already be stale. One pull request lost that race nine
times in 4h20m on a five-file change.

The fix under test: a `Queue position` commit status that reserves one turn at a time.

## Layout

| File | Purpose |
|---|---|
| `.github/workflows/mock-gates.yml` | Five fake gates named after the real `test-app` checks, on `ubuntu-latest`, with a deliberate fast/slow spread (10s–45s) so the starvation race is reproducible. Put `MOCK_FAIL=frontend` in a PR body to fail one on purpose. |
| `.github/workflows/pr-merge-train.yml` | The mechanism under test. Elects one head, posts `Queue position` on every open PR, arms native auto-merge, and updates only the head's branch. Never merges directly. |

## Knobs

- Repo variable `MERGE_TRAIN_EJECT_MINUTES` — shortens the 30-minute eject so it can be tested.
- `workflow_dispatch` input `simulate_crash` — forces the sweep to throw, to prove it fails open.

## Differences from test-app

- `ubuntu-latest` instead of the self-hosted runner.
- `GITHUB_TOKEN` instead of a GitHub App token. `test-app` needs the App token because a branch
  push made with `GITHUB_TOKEN` does not re-trigger CI — this lab measures what happens without it.
- Public repo, because rulesets are unavailable on a private repo under a free personal account.
