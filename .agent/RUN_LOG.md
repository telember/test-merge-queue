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
