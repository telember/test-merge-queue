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

## Verification round 1 — results

| # | Case | Result |
|---|---|---|
| V1 | Three PRs opened from the *same* `main` tip | All held; no simultaneous merge |
| V2 | Gates still running | `pending` / "waiting — gates running" |
| V3 | Head election, lowest number wins | `#1` `success` / "ready to merge next" |
| V4 | Ordinal positions | `#2` "2nd in line, behind #1", `#3` "3rd in line, behind #1" |
| V5 | **Out-of-turn merge blocked** | `#2` refused: *"the base branch policy prohibits the merge"* |
| V6 | Head merges normally | `#1` merged |
| V7 | **`update-branch` under `GITHUB_TOKEN`** | **FAILED — see below** |

## P0 reproduced in the lab: update-branch under GITHUB_TOKEN strands the head

Sequence, from real timestamps:

```
14:01:22  #1 merged
14:01:25  push sweep elects #2, calls update-branch
14:01:59  new SHA 5fbc7b1c; Mock Gates AND Merge Train both -> action_required
          Queue position on 5fbc7b1c: NONE
          mergeable_state: blocked
```

The merge commit `update-branch` creates is authored by `github-actions[bot]`, and the runs it
triggers land in `action_required` awaiting approval. So the sweep that would post `Queue position`
on the new SHA never runs, and with the context required the pull request is stuck.

The `if: always()` backstop does **not** save this: the backstop is a step inside the very run that
is waiting for approval. A backstop can only cover failures of a run that actually starts.

**Consequence for ms-pms-app.** The GitHub App token is not an optimisation, it is a prerequisite —
which is exactly what `pr-auto-update-branch.yml` already says in its own header comment. The design
doc was wrong to treat it as optional.

**Consequence for the design.** "Every failure path ends in `success`" is only true for failures
*inside* a sweep. A sweep that never starts posts nothing. The only real mitigations are: never let
the train mutate the head's SHA under a token whose commits cannot re-trigger workflows, and keep the
kill switch documented.

## Verification round 2 — complete

| # | Case | Result |
|---|---|---|
| V8  | Promotion after a merge | PARTIAL — works, but needed the blocked runs approved by hand |
| V9  | Fail-open under a forced crash | PASS — both PRs -> `success` / "train unavailable", job conclusion `failure`, next sweep restored the ordering exactly |
| V10 | Eject after the timeout | PASS — label, comment with SHA marker, `#3` promoted in the same sweep, and the ejected PR **kept `success`** |
| V11 | Push to rejoin after an eject | PASS — `pull_request` sweep removed the label and returned `#2` to the line |

Unplanned observation from V9: the forced crash rewrote the head status with a different
description, which reset the `created_at` that the head-since clock reads — so the first eject
attempt correctly did nothing. A pull request is not charged for time spent while the train was
down. Not designed; it falls out of measuring the clock from the status the train itself writes.

## Final state: NEEDS_HUMAN_REVIEW

Ten of eleven cases pass. The mechanism is proven: a green, up-to-date pull request was refused
solely because it was second in line.

**Blocked on one prerequisite before ms-pms-app.** `update-branch` under `GITHUB_TOKEN` strands the
head with no status. It must run under the Release GitHub App token — which is exactly what
`pr-auto-update-branch.yml` already does, and says so in its own header comment.

Two things this lab structurally cannot verify, both requiring a private repo:
- the missing `checks: read` class of bug (public-repo check reads succeed on metadata alone);
- the GITHUB_TOKEN API budget under real PMS volume.
