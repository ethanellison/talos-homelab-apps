---
name: substitution-contract-check
description: Use this skill to validate Flux postBuild substitutions and prevent unresolved runtime placeholders.
---

# substitution-contract-check

## When this skill applies

- Any manifest introducing `${...}` placeholders.
- Any IngressRoute or environment-specific endpoint changes.

## Required checks

- Detect `${...}` placeholders in edited manifests.
- Confirm placeholders are intentionally resolved via Flux substitution.
- For staging patterns, verify `postBuild.substituteFrom` references `ConfigMap/staging-vars` in `flux-system`.
- Ensure placeholders are not left unresolved unless explicitly intended by Flux runtime substitution.

## Repository-specific note

- `${STAGING_LB_IP}` is an expected substitution pattern for staging ingress host behavior.

## Output contract

Report:

- placeholder key
- files where used
- substitution source used (or missing)
- remediation if missing
