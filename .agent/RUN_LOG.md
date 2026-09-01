# RUN LOG

## Setup findings (lab infrastructure, not the mechanism under test)

| # | Finding | Detail |
|---|---|---|
| S1 | Rulesets unavailable on private repo, free personal account | `POST /rulesets` → 403 *"Upgrade to GitHub Pro or make this repository public"*. Repo made public with consent. |
| S2 | Git credential helper pushed as the wrong identity | HTTPS remote authenticated as `iyc-central`, not `telember` → 403. Fixed by using the SSH remote. |
| S3 | Ruleset blocks creation of its own target branch | `do_not_enforce_on_create: false` (same as PMS) rejects the initial `main` push. Seed with enforcement disabled, then re-enable. |
| S4 | Unquoted `PMS: 431s` in a step name broke the whole workflow | YAML reads colon-space as a mapping. Startup failure, **zero jobs, no logs** — the run shows the file *path* instead of the workflow name. Caught only by `actionlint`, not by `yaml.safe_load` of the other file. |
| S5 | `inputs` context is invalid outside `workflow_dispatch` | `${{ inputs.x }}` in a job `env:` causes a startup failure on `push`/`pull_request`. Use `${{ github.event.inputs.x }}`. |

## Process correction

S4 existed because a YAML parse was run on `pr-merge-train.yml` and the result was reported as
"YAML OK" without `mock-gates.yml` having been checked. `actionlint` is now the gate for every
workflow file before any push.

`actionlint` 1.7.12 reports `pull_request_review_thread` as an unknown webhook event. GitHub does
support it; treat this specific finding as a stale-linter false positive, pending empirical proof
that the workflow starts.

## Review round 1 — findings acted on

Independent reviewer (fresh context) returned 2xP0 and 7xP1. Fixed in `7d25ad7`:

| Sev | Finding | Fix |
|---|---|---|
| P0 | Fork PRs can never receive the status; a fork run also burns the single-flight slot with a read-only token | Job-level `if` restricts to same-repo events; sweep filters to same-repo heads |
| P0 | `base: 'main'` filter meant PRs on any other base never got a status at all | Sweep lists **every** open PR; non-main bases get `success` / "not queued — base is X" |
| P1 | **`checks: read` missing.** Public lab masks it; on private ms-pms-app every sweep 403s, fails open, and silently orders nothing | Added to `permissions` |
| P1 | Eject measured `committer.date` — the commit's age, not time at the front. Would mass-eject the whole backlog, each eject firing the next sweep | Eject on the `created_at` of the PR's own `ready to merge next` status, and only when `behind_by === 0` |
| P1 | Eject label was never removed, so "push to rejoin" did not work and an ejected PR was excluded forever | Eject comment carries `<!-- merge-train-eject sha=... -->`; label is dropped once the head SHA differs |
| P1 | **"gates running -> success" let a PR merge out of turn** the moment its gates went green | Split durable vs transitional blocks. Durable (draft/conflict/ejected/other base) -> `success`; transitional (gates running, threads open, mergeability unknown) -> `pending` |
| P1 | `listCommitStatusesForRef` unpaginated: >100 Queue position writes on one SHA push real gates off page 1, so a red gate reads as green | `getCombinedStatusForRef` (one entry per context) + identical writes skipped |
| P2 | Every 422 from `updateBranch` treated as a conflict -> irreversible eject on a transient error | Eject only when the message matches `/conflict/i` |
| P2 | `updateBranch` without `expected_head_sha` could act on an unevaluated SHA | `expected_head_sha` passed |
| P2 | A cancelled job leaves PRs on a stale `pending`, and the catch never runs | `if: always()` backstop step releases any PR left pending |

## Design correction forced by the review

The grill concluded "a not-ready PR gets `success`, never `pending`". That is only sound for
**durable** blocks. For a PR whose gates are still running, `success` is a live hazard: when the
gates finish there is no cheap trigger to re-sweep, so it holds a stale `success` alongside newly
green gates and can merge out of turn. Transitional states must be `pending`.

`pull_request_review_thread` cannot be used to close that gap: GitHub rejects the workflow file
outright (verified — startup failure, zero jobs, run displays the file path instead of the name).
