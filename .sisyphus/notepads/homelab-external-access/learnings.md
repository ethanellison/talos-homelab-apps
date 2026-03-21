Updated prod cluster kustomization to reference prod-vars instead of staging-vars.

Rationale: The prod Kustomization was incorrectly substituting values from the staging-vars ConfigMap. For environment-specific substitutions we must reference prod-vars in the prod cluster so Flux injects production values. This change keeps all other kustomization fields unchanged and only swaps the ConfigMap name.

Note: Added a minimal root kustomization at clusters/prod/kustomization.yaml so `kustomize build clusters/prod` can be run locally for render verification. It lists infrastructure.yaml first and apps.yaml second to preserve dependency ordering (infrastructure -> apps).

Added environment-specific overlays for Traefik under infrastructure/overlays/staging and infrastructure/overlays/prod. The staging overlay patches the Traefik HelmRelease to request service.type: LoadBalancer so the externally-managed hcloud CCM can provision a LoadBalancer; the prod overlay leaves the default NodePort and continues to use postBuild substitution from prod-vars. This preserves existing Traefik defaults except for the single staging patch that requests a LoadBalancer.

Created apps/devenv/overlays/prod to mirror the staging devenv overlay but use an internal hostname (Host(`devenv.local`)) instead of the staging nip.io host. This overlay keeps the same resources order and namespace `devenv`.

2026-03-20T18:00:51-04:00 - Phase 4 verification: kustomize renders OK for infrastructure, staging, prod; Traefik staging=LoadBalancer, prod=NodePort; bun and LSP missing; next action: reconcile flux kustomizations in order (infrastructure -> clusters/staging -> clusters/prod) and open PR if infra changes needed.
2026-03-21T17:39:03Z - ran `kustomize build infrastructure` -> exit code 0; rendered HelmRelease "traefik" present; no references to "hcloud-ccm" or "cloud-controller-manager" found.
2026-03-21T17:46:00Z - ran kustomize build clusters/staging -> exit code 0; rendered IngressRoute host uses dev..nip.io and traefik HelmRelease present.

2026-03-21T13:51:15.916716-04:00 - ran kustomize build clusters/prod -> exit code 0; rendered output: prod-vars present, no staging-vars, IngressRoute Host(`devenv.local`) found.
2026-03-21T17:57:00Z - verification: grep across repo for "hcloud-ccm" and "cloud-controller-manager" -> matches found in infrastructure/hcloud-ccm/helmrelease.yaml and a commented reference in infrastructure/kustomization.yaml; this indicates repo contains CCM manifests but infrastructure/kustomization.yaml currently comments it out. Presence of these files means the repo contains references to hcloud-ccm (investigate if they should be removed).
2026-03-21T17:57:00Z - verification: kustomize build clusters/prod -> exit code 0; rendered output saved to /tmp/kustomize_prod_output.yaml; grep for "staging-vars" returned no matches.
2026-03-21T00:00:00Z - Removed infrastructure/hcloud-ccm directory and verified kustomize builds passed; grep found no hcloud-ccm or cloud-controller-manager matches. Verification outputs recorded: infra=/tmp/kustomize_infra_after.yaml staging=/tmp/kustomize_staging_after.yaml prod=/tmp/kustomize_prod_after.yaml; grep found 0 matches for both terms.
