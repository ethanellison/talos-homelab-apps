# Rollbacks

- Repo-safe rollback for this wave: no manifest rollback is needed because no files under `apps/` or `clusters/` are changed.
- If a future SSH-native overlay is added, rollback should prefer reverting the new overlay path first, then the access-gate wiring, while leaving the existing browser-facing baseline untouched until replacement is proven.

## Rollback sequence

1. Disable the external access gate first.
2. Revert any new SSH-native overlay or auth wiring.
3. Restore the prior browser-facing entrypoint only if it was intentionally replaced.
4. Recheck service exposure, then confirm tmux/session access still works.

## Observability commands

- `flux get kustomizations -n flux-system`
- `kubectl get pods -n flux-system`
- `kubectl get svc,ingress,ingressroute -A`
- `kustomize build apps/devenv/overlays/staging`
- `kustomize build apps/devenv/overlays/prod`
- `kustomize build infrastructure`
- `kustomize build clusters/staging`
- `kustomize build clusters/prod`

## Future GitOps-owned SSH-native overlay notes

- Prefer a new overlay path rather than mutating the current browser-first manifests in place.
- Keep secret material out of plain manifests; use Sealed Secrets or SOPS with Flux postBuild substitution contracts for high-level secret handling.
- Do not implement the future overlay in this wave.
