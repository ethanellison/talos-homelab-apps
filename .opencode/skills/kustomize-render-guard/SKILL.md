---
name: kustomize-render-guard
description: Use this skill to run mandatory Kustomize render validation for changed GitOps manifests before concluding work.
---

# kustomize-render-guard

## When this skill applies

- Any manifest change in this repository.
- Any pre-PR validation pass.

## Required validation workflow

Run builds in this order whenever relevant:

1. Directly modified kustomization path(s).
2. At least one parent aggregation path.

Canonical repository commands:

```bash
kustomize build clusters/staging
kustomize build infrastructure
kustomize build apps/devenv/overlays/staging
```

Single-scope equivalents:

```bash
kustomize build apps/devenv/overlays/staging
kustomize build infrastructure/traefik
```

## Failure handling

- On render failure, stop and fix root cause before continuing.
- Do not bypass failures.

## Output contract

Report:

- Commands executed.
- Exit status for each command.
- First failing command if any.
