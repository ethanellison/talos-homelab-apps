---
name: yaml-policy-lint
description: Use this skill to enforce repository YAML conventions for naming, formatting, and namespace safety.
---

# yaml-policy-lint

## When this skill applies

- Any YAML manifest creation or modification.

## Policy checks

- 2-space indentation.
- Stable, scannable key ordering.
- `metadata.name` in lowercase kebab-case.
- Explicit `metadata.namespace` for namespaced resources.
- Filenames in lowercase kebab-case by resource intent (`kustomization.yaml`, `helmrelease.yaml`, `helmrepository.yaml`, `ingressroute.yaml`).
- No commented-out dead config.

## Guardrails

- Never suppress schema mistakes with placeholders not handled by Flux substitution.
- Never commit secrets, kubeconfigs, tokens, or credentials.

## Output contract

Return violations as:

- file path
- rule name
- recommended fix
