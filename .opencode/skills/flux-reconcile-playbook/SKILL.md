---
name: flux-reconcile-playbook
description: Use this skill when Flux source, Kustomization, or Helm resources change and explicit reconcile guidance is needed.
---

# flux-reconcile-playbook

## When this skill applies

- Changes to `clusters/*` Flux objects.
- Changes to HelmRelease or HelmRepository resources.
- Any task requiring live reconciliation steps after manifest updates.

## Canonical reconcile sequence

```bash
flux reconcile source git flux-system -n flux-system
flux reconcile kustomization flux-system -n flux-system
flux reconcile kustomization infrastructure -n flux-system
flux reconcile kustomization apps -n flux-system --with-source
```

## Operational checks

```bash
kubectl get kustomizations -n flux-system
kubectl get helmreleases -A
kubectl get pods -n traefik
kubectl get crds | grep -i traefik
```

## Guardrails

- Use namespace `flux-system` for the listed reconcile commands.
- Do not claim reconciliation succeeded without command evidence.
