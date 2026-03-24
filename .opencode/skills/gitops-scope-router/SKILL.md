---
name: gitops-scope-router
description: Use this skill to map changed files to GitOps validation scopes and required parent aggregation builds in this repository.
---

# gitops-scope-router

## When this skill applies

- Any change under `apps/`, `infrastructure/`, or `clusters/`.
- Any task that asks what to build, validate, or reconcile after manifest edits.

## Repository scope map

- `apps/<app>/overlays/<env>/...` → app overlay scope
  - Validation target: `kustomize build apps/<app>/overlays/<env>`
- `infrastructure/<module>/...` → infrastructure module scope
  - Validation target: `kustomize build infrastructure/<module>`
- `clusters/<env>/...` → cluster wiring scope
  - Validation target: `kustomize build clusters/<env>`

## Required parent aggregation rule

After building directly changed scopes, always build at least one parent aggregation path:

- `kustomize build infrastructure` and/or
- `kustomize build clusters/staging`

## Output contract

Return a concise plan with:

1. Detected change scopes.
2. Exact render commands in run order.
3. Whether Flux reconcile commands are required.

## Guardrails

- Do not edit generated `gotk-*` files unless explicitly requested.
- Keep changes in smallest correct scope.
- Never introduce hardcoded secrets.
