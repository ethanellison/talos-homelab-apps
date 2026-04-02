plan: external-dns-traefik
created: 2026-03-24T13:07:05.371Z
summary: |
  Add `external-dns` and `cert-manager` as Flux-managed infrastructure, keep Traefik CRD host rules as the source of truth, and split staging vs prod DNS ownership cleanly.
  Deliverables: external-dns module per environment, cert-manager with env-specific issuers, Traefik host-rule alignment for DNS automation.
  Staging -> Cloudflare (staging.sneakysquid.xyz); Prod -> Pi-hole (staging.lan); validate via kustomize renders and Flux reconcile.

---

# Learnings (append-only)

## 2026-03-24T13:07:05.371Z
- TL;DR extracted from plan:
  Add `external-dns` and `cert-manager` as Flux-managed infrastructure, keep Traefik CRD host rules as the source of truth, and split staging vs prod DNS ownership cleanly.
  Deliverables: external-dns module per environment, cert-manager with env-specific issuers, Traefik host-rule alignment for DNS automation.
  Staging -> Cloudflare (staging.sneakysquid.xyz); Prod -> Pi-hole (staging.lan). Validate renders + reconcile as specified in plan.

appended-by-session: ses_2e0266693ffepKMOaaTVMp9YDK at 2026-03-24T13:07:05.371Z

## 2026-03-24T13:11:34.708Z
- Added external-dns base module under `infrastructure/external-dns/` plus staging/prod overlay patches.
- Base HelmRelease now watches `traefik-proxy`; staging patches Cloudflare settings for `staging.sneakysquid.xyz`; prod patches Pi-hole settings for `staging.lan`.

## 2026-03-24T13:17:40.000Z
- Aligned workload hostnames for devenv:
  - staging: `devenv.staging.sneakysquid.xyz`
  - prod: `devenv.staging.lan`
  Updated overlays: `apps/devenv/overlays/staging/ingressroute.yaml` (created) and `apps/devenv/overlays/prod/ingressroute.yaml` (updated).

## 2026-03-24T13:20:00.000Z
- Added cert-manager base module under `infrastructure/cert-manager/` with Jetstack HelmRepository and HelmRelease CRD installation enabled.
- Added overlay ClusterIssuers for staging Cloudflare DNS-01 and prod internal CA issuance.

## 2026-03-24T13:24:00.000Z
- Aligned devenv workload host rules with DNS zones: staging `devenv.staging.sneakysquid.xyz`, prod `devenv.staging.lan`.

## 2026-03-24T14:05:00.000Z
- Traefik DNS publication now comes from the LoadBalancer Service annotation for `traefik.staging.sneakysquid.xyz`.
- Workload routes target the stable Traefik hostname instead of self-referencing the app host.

## 2026-03-24T13:31:00.000Z
- Split cert-manager issuers out of infra overlays into separate Flux Kustomizations (`clusters/staging/cert-manager.yaml`, `clusters/prod/cert-manager.yaml`) so cert-manager CRDs install before issuer dry-runs.

## 2026-03-24T13:44:00.000Z
- cert-manager HelmRelease needed chart values `crds.enabled: true` in addition to Flux `install.crds: Create` / `upgrade.crds: Create` so chart CRDs are rendered and registered for dependent issuers.

## 2026-03-24T13:50:00.000Z
- Enabled Traefik dashboard via `api.dashboard: true` and exposed it with route-based `IngressRoute` manifests in staging/prod overlays targeting `api@internal`.

## 2026-03-24T19:40:00.000Z
- Flux ordering is already correct at the cluster layer (`infrastructure` before `apps` via `dependsOn`); Kustomize file ordering is not the install mechanism. Removed stray route manifests from infra overlays and kept Traefik CRDs in infrastructure while app IngressRoutes live under apps.
