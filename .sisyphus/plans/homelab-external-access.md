# Homelab External Access Plan

## TL;DR

Update the repo so staging and prod have cleanly separated external-access behavior while keeping shared ingress patterns as close as possible. Remove prod’s coupling to staging vars, keep hcloud-ccm out of repo ownership, and make the app/ingress layout explicit for staging public access vs prod home-network access.
For staging, keep Traefik and expose it via an externally provisioned LoadBalancer; do not move the devenv service to direct LoadBalancer mode.

## Context

### Original Request
Implement the repo-specific recommendations for staging/prod external access in this Flux GitOps homelab.

### Key Findings
- `infrastructure/kustomization.yaml` currently includes `./traefik` only; `./hcloud-ccm` is commented out.
- `infrastructure/traefik/helmrelease.yaml` uses `service.type: NodePort`, `externalTrafficPolicy: Cluster`, and trusted IPs `10.0.0.0/8`.
- `clusters/staging/infrastructure.yaml` and `clusters/staging/apps.yaml` use `staging-vars`.
- `clusters/prod/infrastructure.yaml` incorrectly references `staging-vars`.
- `clusters/staging/apps.yaml` points to `./apps/devenv/overlays/staging` and uses `dev.${STAGING_LB_IP}.nip.io`.
- `clusters/prod/apps.yaml` points to `./apps`.
- Staging prune is `true`; prod prune is `false`.
- hcloud-ccm is managed outside this repo and installed during cluster provisioning.

## Work Objectives

### Core Objective
Make repo-owned manifests reflect the real environment split: staging is ephemeral/public on hcloud, prod is private/home-network on Proxmox.

### Concrete Deliverables
- Prod no longer references `staging-vars`.
- Repo keeps `hcloud-ccm` out of manifest ownership.
- Staging/prod app wiring becomes explicit and environment-specific.
- Staging services remain routed through Traefik, which is reachable via an LB provisioned by the externally managed CCM.
- Prod remains behind Traefik for home-network access.

### Definition of Done
- [x] `kustomize build infrastructure` succeeds. (2026-03-21T17:39:03Z)
- [x] `kustomize build clusters/staging` succeeds. (2026-03-21T17:46:11Z)
- [x] `kustomize build clusters/prod` succeeds. (2026-03-21T13:51:15-04:00)
 - [x] Prod manifests do not reference `staging-vars`. (2026-03-21T17:57:00Z)
  - [x] No repo file attempts to install or enable hcloud-ccm. (2026-03-21T00:00:00Z)

### Must Have
- Keep the plan repo-owned only.
- Preserve staging/prod separation.
- Preserve staging’s public, ephemeral testing role.
- Use env-specific configmaps (`staging-vars`, `prod-vars`).
- Use Traefik for both envs; staging’s Traefik is LB-backed by CCM, prod’s Traefik stays internal/home-network only.

### Must NOT Have
- No repo-managed hcloud-ccm installation.
- No generic “cloud LB everywhere” assumptions.
- No secret material.

## Verification Strategy

> All verification must be executable by the agent; no human checks.

### Test Decision
- **Infrastructure exists**: YES
- **Automated tests**: None
- **Framework**: none

### Agent-Executed QA Scenarios

#### Scenario: Shared infra renders cleanly
  Tool: Bash
  Preconditions: working tree contains the planned YAML edits
  Steps:
    1. Run `kustomize build infrastructure`
    2. Assert command exits 0
    3. Confirm Traefik resources render and no CCM resources appear
  Expected Result: shared infra renders without errors
  Evidence: terminal output

#### Scenario: Staging cluster renders cleanly
  Tool: Bash
  Preconditions: staging manifests updated
  Steps:
    1. Run `kustomize build clusters/staging`
    2. Assert command exits 0
    3. Confirm staging still references `staging-vars` and `nip.io`
  Expected Result: staging remains public/test-oriented
  Evidence: terminal output

#### Scenario: Prod cluster decouples from staging vars
  Tool: Bash
  Preconditions: prod manifests updated
  Steps:
    1. Run `kustomize build clusters/prod`
    2. Assert command exits 0
    3. Confirm rendered output does not contain `staging-vars`
  Expected Result: prod no longer depends on staging config
  Evidence: terminal output

## Phased Plan

### Phase 1: Remove repo ownership ambiguity and fix prod coupling
Dependency: none

What to do
- Clean up `infrastructure/kustomization.yaml` so it only lists repo-managed infra and contains no CCM-ownership ambiguity.
- Update `clusters/prod/infrastructure.yaml` so it no longer references `ConfigMap/staging-vars`.
- Decide whether prod uses its own substitution source or no `postBuild.substituteFrom` at all.
- Keep prune policy as an explicit environment choice (`staging: true`, `prod: false`) unless the user decides to align them.

Files affected
- `infrastructure/kustomization.yaml`
- `clusters/prod/infrastructure.yaml`

Verification
- `kustomize build infrastructure`
- `kustomize build clusters/prod`

### Phase 2: Keep Traefik shared, but make staging/prod exposure intent explicit
Dependency: Phase 1

What to do
- Leave Traefik as the shared ingress layer in `infrastructure/traefik/helmrelease.yaml`.
- Keep staging’s `nip.io` host pattern and `${STAGING_LB_IP}`-driven ingress route.
- Ensure prod does not reuse staging-only hostnames or substitutions.
- Model staging access as LB-backed service exposure for the devenv service (the LB comes from the externally provisioned CCM).
- Keep prod on Traefik for internal/home-network access.
- If needed, adjust Traefik values only where they support the shared contract; do not add CCM lifecycle management.

Files affected
- `infrastructure/traefik/helmrelease.yaml`
- `apps/devenv/overlays/staging/ingressroute.yaml`
- `apps/devenv/overlays/staging/kustomization.yaml`

Verification
- `kustomize build infrastructure`
- `kustomize build apps/devenv/overlays/staging`

### Phase 3: Make prod app wiring explicit
Dependency: Phase 1

What to do
- Replace prod’s bare `./apps` path with an explicit prod app overlay path.
- Create a prod overlay that mirrors staging’s behavior as closely as possible while using private/home-network exposure.
- Keep the prod overlay free of `nip.io` and staging-specific substitution names.

Files affected
- `clusters/prod/apps.yaml`
- `apps/devenv/overlays/prod/kustomization.yaml` (new, if approved)
- `apps/devenv/overlays/prod/ingressroute.yaml` (new, if approved)

Verification
- `kustomize build clusters/prod`
- `kustomize build apps/devenv/overlays/prod` (if created)

### Phase 4: Roll out safely and reconcile in order
Dependency: Phases 1-3

What to do
- Validate shared infra first.
- Validate staging next.
- Validate prod last.
- Reconcile Flux only after manifests render cleanly.

Files affected
- none

Verification
- `kustomize build infrastructure`
- `kustomize build clusters/staging`
- `kustomize build clusters/prod`
- If applicable: `flux reconcile kustomization infrastructure -n flux-system`
- If applicable: `flux reconcile kustomization apps -n flux-system --with-source`

## Files Affected Per Phase

### Shared
- `infrastructure/kustomization.yaml`
- `infrastructure/traefik/helmrelease.yaml`

### Staging-only
- `apps/devenv/overlays/staging/kustomization.yaml`
- `apps/devenv/overlays/staging/ingressroute.yaml`
- `clusters/staging/infrastructure.yaml`
- `clusters/staging/apps.yaml`

### Prod-only
- `clusters/prod/infrastructure.yaml`
- `clusters/prod/apps.yaml`
- `apps/devenv/overlays/prod/kustomization.yaml` (if approved)
- `apps/devenv/overlays/prod/ingressroute.yaml` (if approved)

## Decisions Resolved
- Prod gets its own overlay path.
- Prod uses a prod-specific substitution source (`prod-vars`); staging uses `staging-vars`.
- Staging is LB-backed via the externally managed CCM; prod uses Traefik for internal/home-network access.
