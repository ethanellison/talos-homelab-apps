Updated prod cluster kustomization to reference prod-vars instead of staging-vars.

Rationale: The prod Kustomization was incorrectly substituting values from the staging-vars ConfigMap. For environment-specific substitutions we must reference prod-vars in the prod cluster so Flux injects production values. This change keeps all other kustomization fields unchanged and only swaps the ConfigMap name.

Note: Added a minimal root kustomization at clusters/prod/kustomization.yaml so `kustomize build clusters/prod` can be run locally for render verification. It lists infrastructure.yaml first and apps.yaml second to preserve dependency ordering (infrastructure -> apps).

Added environment-specific overlays for Traefik under infrastructure/overlays/staging and infrastructure/overlays/prod. The staging overlay patches the Traefik HelmRelease to request service.type: LoadBalancer so the externally-managed hcloud CCM can provision a LoadBalancer; the prod overlay leaves the default NodePort and continues to use postBuild substitution from prod-vars. This preserves existing Traefik defaults except for the single staging patch that requests a LoadBalancer.

Created apps/devenv/overlays/prod to mirror the staging devenv overlay but use an internal hostname (Host(`devenv.local`)) instead of the staging nip.io host. This overlay keeps the same resources order and namespace `devenv`.
