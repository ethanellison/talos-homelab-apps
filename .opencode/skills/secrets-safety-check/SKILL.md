---
name: secrets-safety-check
description: Use this skill to detect and block accidental introduction of secrets or credentials in GitOps manifests.
---

# secrets-safety-check

## When this skill applies

- Any manifest or documentation change in this repo.

## Checks

- Reject committed kubeconfigs, tokens, credentials, private keys, or secret literals.
- Flag suspicious keys/values and require secret-management alternatives.
- Ensure examples and docs do not include real sensitive values.

## Repository policy

- This repository must not store secrets.

## Output contract

Return findings with:

- file path
- matched sensitive pattern
- safe remediation guidance
