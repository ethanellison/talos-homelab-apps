plan: external-dns-traefik
created: 2026-03-24T13:07:05.371Z
summary: |
  Outstanding decisions were extracted from the plan: confirm prod Pi-hole external-dns backend/mechanism and how provider credentials enter Flux.

---

# Decisions (append-only)

## 2026-03-24T13:07:05.371Z
- Decision required: Confirm the supported external-dns backend/mechanism for prod Pi-hole authoritative DNS.
- Decision required: Confirm how Cloudflare and prod credentials should enter Flux (SOPS, ExternalSecrets, or an existing Secret path).

appended-by-session: ses_2e0266693ffepKMOaaTVMp9YDK at 2026-03-24T13:07:05.371Z

## 2026-03-24T13:11:34.708Z
- Created files: `infrastructure/external-dns/{kustomization.yaml,namespace.yaml,helmrepository.yaml,helmrelease.yaml}`, `infrastructure/overlays/staging/external-dns-values-patch.yaml`, `infrastructure/overlays/prod/external-dns-values-patch.yaml`.
- `txt-owner-id` values chosen: `staging-external-dns` and `prod-external-dns`.
- Credential Secret names referenced: `cloudflare-api-token` and `pihole-password`.

## 2026-03-24T13:17:40.000Z
- Hostname alignment applied for devenv:
  - staging: `devenv.staging.sneakysquid.xyz`
  - prod: `devenv.staging.lan`
  - Files modified: `apps/devenv/overlays/staging/ingressroute.yaml`, `apps/devenv/overlays/prod/ingressroute.yaml`.

## 2026-03-24T13:31:00.000Z
- cert-manager issuers moved out of `infrastructure/overlays/*` into dedicated Flux Kustomizations so the HelmRelease can reconcile and install CRDs before `ClusterIssuer` dry-runs.
- Files changed: `clusters/staging/cert-manager.yaml`, `clusters/prod/cert-manager.yaml`, `clusters/staging/kustomization.yaml`, `clusters/prod/kustomization.yaml`, `infrastructure/overlays/staging/kustomization.yaml`, `infrastructure/overlays/prod/kustomization.yaml`.

## 2026-03-24T13:44:00.000Z
- cert-manager HelmRelease updated to set `values.crds.enabled: true` alongside Flux CRD policies so the chart actually renders/registers CRDs.

## 2026-03-24T13:50:00.000Z
- Traefik dashboard exposed safely via route-based `IngressRoute` instead of `api.insecure`.
- Files changed: `infrastructure/traefik/helmrelease.yaml`, `infrastructure/overlays/staging/traefik-dashboard-ingressroute.yaml`, `infrastructure/overlays/prod/traefik-dashboard-ingressroute.yaml`, `infrastructure/overlays/staging/kustomization.yaml`, `infrastructure/overlays/prod/kustomization.yaml`.

## 2026-03-24T19:40:00.000Z
- Cluster ordering relies on Flux Kustomization `dependsOn`, not YAML resource order inside a Kustomize file.
- Traefik stays in infrastructure; IngressRoutes stay in apps so CRDs exist before route resources are reconciled.

## 2026-03-24T13:20:00.000Z
- Created cert-manager files: `infrastructure/cert-manager/{kustomization.yaml,namespace.yaml,helmrepository.yaml,helmrelease.yaml}` plus overlay issuers for staging and prod.
- Chosen issuer types: staging `ClusterIssuer` with ACME DNS-01 via Cloudflare; prod `ClusterIssuer` with CA issuer backed by `prod-internal-ca`.
- Credential Secret names referenced by convention: `cloudflare-api-token` and `prod-internal-ca`.

## 2026-03-24T13:24:00.000Z
- Modified files: `apps/devenv/overlays/staging/ingressroute.yaml`, `apps/devenv/overlays/staging/kustomization.yaml`, `apps/devenv/overlays/prod/ingressroute.yaml`.
- Explicit hostnames used: `devenv.staging.sneakysquid.xyz` (staging) and `devenv.staging.lan` (prod).

## 2026-03-24T14:05:00.000Z
- Source-of-truth changed to the Traefik LoadBalancer Service for the stable `traefik.staging.sneakysquid.xyz` record.
- external-dns staging now watches both `service` and `traefik-proxy`; app routes target `traefik.staging.sneakysquid.xyz` and prod keeps `traefik.staging.lan`.
