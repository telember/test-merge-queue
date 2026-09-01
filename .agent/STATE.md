# STATE

**Task** — verify the ms-pms-app merge train mechanism in an isolated lab.
**Risk** — MEDIUM. Throwaway public repo, no secrets, no production surface. The *logic*
being validated is destined for ms-pms-app, so correctness of the sweep matters.

## Decisions

- Repo made **public**: rulesets and required status checks are unavailable on a private
  repo under a free personal account (verified: HTTP 403 on ruleset POST).
- Gates are mocked at ~10x speed, keeping the fast/slow spread that causes starvation.
- `GITHUB_TOKEN` instead of a GitHub App: this lab measures whether `update-branch` under
  `GITHUB_TOKEN` re-triggers gates. PMS uses an App token specifically because it does not.

## Status

Scaffolded. Ruleset and case verification pending.
