# CONSTRAINTS

- No real ms-pms-app source, config, or secrets in this repository. It is public.
- The train must never merge, force-push, or bypass a rule.
- Every sweep exit path must leave every open PR holding a `success` status.
- Verification is by observed API state, never by assertion.
