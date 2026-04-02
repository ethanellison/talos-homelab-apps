# external-dns + Traefik exposure plan

## TL;DR

> Add `external-dns` and `cert-manager` as Flux-managed infrastructure, keep Traefik CRD host rules as the source of truth, and split staging vs prod DNS ownership cleanly.
>
> **Deliverables**:
> - `external-dns` module with environment-specific provider config
> - `cert-manager` module with environment-appropriate issuers
> - Traefik/workload hostname alignment for DNS automation
> - Render + reconcile validation for staging and prod

**Estimated Effort**: Medium
**Parallel Execution**: YES - 2 waves
**Critical Path**: external-dns module → cert-manager module → workload hostname alignment → full validation

---

## Context

### Original Request
Use `external-dns` in conjunction with `traefik` to expose workloads to the network.

### Interview Summary
**Key Discussions**:
- DNS provider split: Cloudflare for staging, Pi-hole for prod.
- Exposure split: public records for staging, internal-only for prod.
- Hostname pattern: same naming scheme across environments.
- Prod DNS authority: Pi-hole is authoritative for internal records.
- TLS: cert-manager should be added to the cluster plan.

**Research Findings**:
- Traefik is already installed as `infrastructure/traefik` and patched per environment in `infrastructure/overlays/staging/traefik-helmrelease-patch.yaml`.
- Flux infra entrypoints already point at `infrastructure/overlays/staging` and `infrastructure/overlays/prod`.
- Traefik CRDs are already used by workloads (`IngressRoute`, `IngressRouteTCP`).
- `external-dns` does not exist in the repo yet.
- Best-practice direction: use `traefik-proxy` for Traefik CRDs, keep `txt-owner-id` unique per environment, and separate public/private ownership.

### Decisions Needed
- Confirm the supported external-dns backend/mechanism for prod Pi-hole authoritative DNS.
- Confirm how Cloudflare and prod credentials should enter Flux (for example SOPS, ExternalSecrets, or an existing Secret path).

### Metis Review
**Identified Gaps** (addressed):
- Environment separation: handled by distinct staging/prod configs.
- DNS ownership collisions: handled by unique TXT owner IDs and strict domain filters.
- TLS ownership: handled by planning cert-manager separately from DNS.
- Scope creep: explicitly excludes Traefik exposure rewrites and wildcard overreach.

---

## Work Objectives

### Core Objective
Make Traefik-hosted workloads automatically publish DNS records through external-dns, with staging going to Cloudflare and prod going to Pi-hole, while keeping TLS management in cert-manager.

### Concrete Deliverables
- New `infrastructure/external-dns/` module
- New `infrastructure/cert-manager/` module
- Environment overlays for staging/prod external-dns and cert-manager config
- Hostname updates for representative workloads so external-dns has canonical Traefik CRD host rules to watch

### Definition of Done
- [ ] `kustomize build infrastructure` succeeds
- [ ] `kustomize build infrastructure/overlays/staging` succeeds
- [ ] `kustomize build infrastructure/overlays/prod` succeeds
- [ ] `kustomize build clusters/staging` succeeds
- [ ] `kustomize build clusters/prod` succeeds
- [ ] `kustomize build apps/devenv/overlays/prod` succeeds if workload host rules are updated there
- [ ] `kustomize build apps/devenv/overlays/ssh-native` succeeds if that route is touched
- [ ] Staging renders `external-dns` with Cloudflare + `staging.sneakysquid.xyz`
- [ ] Prod renders `external-dns` with Pi-hole + `staging.lan`
- [ ] Traefik host rules match the selected DNS zones

### Must Have
- Separate staging and prod DNS ownership
- Traefik CRD host rules remain the source of truth
- cert-manager is planned alongside, not hidden inside DNS automation

### Must NOT Have (Guardrails)
- No edits to generated Flux bootstrap artifacts (`gotk-*`)
- No shared external-dns instance across staging/prod
- No broad DNS ownership outside the chosen domains
- No secret material committed into manifests
- No migration away from Traefik CRDs unless explicitly requested

---

## Verification Strategy (MANDATORY)

> **UNIVERSAL RULE: ZERO HUMAN INTERVENTION**
>
> All verification must be executable by the agent with tools. No manual confirmation.

### Test Decision
- **Infrastructure exists**: NO dedicated unit test framework
- **Automated tests**: None
- **Framework**: none

### Agent-Executed QA Scenarios (MANDATORY — ALL tasks)

#### Scenario: Render staging infrastructure cleanly
  Tool: Bash
  Preconditions: Plan implementation completed locally
  Steps:
    1. Run `kustomize build infrastructure/overlays/staging`
    2. Capture stdout to `.sisyphus/evidence/task-1-staging-render.txt`
    3. Assert output contains the `external-dns` and `cert-manager` resources
    4. Assert output contains Cloudflare-specific staging config and the `staging.sneakysquid.xyz` domain filter
  Expected Result: Staging overlay renders without errors
  Failure Indicators: missing resource kinds, unresolved substitutions, or build failure
  Evidence: `.sisyphus/evidence/task-1-staging-render.txt`

#### Scenario: Render prod infrastructure cleanly
  Tool: Bash
  Preconditions: Plan implementation completed locally
  Steps:
    1. Run `kustomize build infrastructure/overlays/prod`
    2. Capture stdout to `.sisyphus/evidence/task-1-prod-render.txt`
    3. Assert output contains the `external-dns` and `cert-manager` resources
    4. Assert output contains Pi-hole/internal-zone config and the `staging.lan` domain filter
  Expected Result: Prod overlay renders without errors
  Failure Indicators: missing resources, wrong provider config, or build failure
  Evidence: `.sisyphus/evidence/task-1-prod-render.txt`

#### Scenario: Reconcile cluster and confirm controllers settle
  Tool: Bash
  Preconditions: Flux access available in the target cluster context
  Steps:
    1. Run `flux reconcile kustomization infrastructure -n flux-system --with-source`
    2. Run `kubectl get pods -A | grep -E 'external-dns|cert-manager|traefik'`
    3. Run `kubectl get clusterissuer,issuer -A`
    4. Capture output to `.sisyphus/evidence/task-4-reconcile.txt`
  Expected Result: resources reconcile and controllers are present
  Failure Indicators: not-ready pods, missing issuers, reconcile errors
  Evidence: `.sisyphus/evidence/task-4-reconcile.txt`

---

## Execution Strategy

### Parallel Execution Waves

Wave 1:
- Task 1 and Task 2 can start independently.

Wave 2:
- Task 3 starts after infrastructure foundations are in place.

Wave 3:
- Task 4 validates the full stack after all manifests are updated.

### Dependency Matrix

| Task | Depends On | Blocks | Can Parallelize With |
|------|------------|--------|---------------------|
| 1 | None | 3, 4 | 2 |
| 2 | None | 3, 4 | 1 |
| 3 | 1, 2 | 4 | None |
| 4 | 1, 2, 3 | None | None |

---

## TODOs

> Implementation + validation stay together in each task.

- [ ] 1. Add the external-dns infrastructure module and environment-specific overlays

  **What to do**:
  - Create `infrastructure/external-dns/` with namespace, Helm repository, and Helm release manifests.
  - Wire the module into `infrastructure/kustomization.yaml` and both environment overlays.
  - Configure Traefik CRD sourcing as the default watch path.
  - Stage Cloudflare settings for `staging.sneakysquid.xyz`; prod Pi-hole settings for `staging.lan`.
  - Use unique `txt-owner-id` values and strict filters per environment.
  - Define the secret bootstrap path for provider credentials before rendering.

  **Must NOT do**:
  - Do not use one shared external-dns instance for both environments.
  - Do not expand zone scope beyond the approved domains.
  - Do not edit `gotk-*` artifacts.

  **Recommended Agent Profile**:
  - **Category**: `unspecified-high`
    - Reason: multi-manifest GitOps work with environment-specific constraints.
  - **Skills**: `gitops-scope-router`, `kustomize-render-guard`, `yaml-policy-lint`, `secrets-safety-check`
    - `gitops-scope-router`: maps the changed manifests to the right validation scope.
    - `kustomize-render-guard`: required for render-based validation.
    - `yaml-policy-lint`: keeps names, ordering, and namespaces consistent.
    - `secrets-safety-check`: prevents accidental credential leakage.
  - **Skills Evaluated but Omitted**:
    - `flux-reconcile-playbook`: useful later in validation, but not necessary for the manifest authoring step.

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 1
  - **Blocks**: Task 3, Task 4
  - **Blocked By**: None

  **References**:
  - `infrastructure/traefik/kustomization.yaml:3-6` - existing infrastructure module pattern to mirror.
  - `infrastructure/traefik/helmrelease.yaml:19-34` - Traefik HelmRelease structure and values style.
  - `infrastructure/overlays/staging/kustomization.yaml:3-6` - staging overlay patch pattern.
  - `infrastructure/overlays/staging/traefik-helmrelease-patch.yaml:7-12` - environment-specific service patch style.
  - `infrastructure/overlays/prod/kustomization.yaml:3-4` - prod overlay aggregation pattern.
  - `clusters/staging/infrastructure.yaml:6-14` - Flux path wiring for staging infra.
  - `clusters/prod/infrastructure.yaml:6-18` - Flux path wiring for prod infra.
  - external-dns guidance: traefik-proxy source for Traefik CRDs, unique TXT ownership, and strict filters.

  **WHY Each Reference Matters**:
  - The Traefik module shows how new shared infra modules should be packaged.
  - The overlays show how to keep environment-specific service/provider differences isolated.
  - The Flux kustomizations show the actual reconciliation entrypoints that must remain valid.

  **Acceptance Criteria**:
  - [ ] `kustomize build infrastructure/overlays/staging` succeeds.
  - [ ] `kustomize build infrastructure/overlays/prod` succeeds.
  - [ ] Rendered staging config targets Cloudflare and `staging.sneakysquid.xyz`.
  - [ ] Rendered prod config targets Pi-hole and `staging.lan`.
  - [ ] Rendered manifests show Traefik CRD source selection and unique TXT ownership.

- [ ] 2. Add cert-manager infrastructure and issuers for staging and prod

  **What to do**:
  - Create `infrastructure/cert-manager/` with namespace and Helm release wiring.
  - Add environment-specific issuer configuration in the overlays.
  - Plan staging issuance with Cloudflare DNS-01 and prod issuance with an internal CA/self-managed trust path.
  - Keep cert-manager concerns separate from external-dns concerns.
  - Specify how issuer credentials/secrets enter Flux safely.

  **Must NOT do**:
  - Do not fold TLS issuance into external-dns manifests.
  - Do not reuse the public staging issuer for prod internal-only records.
  - Do not hardcode secrets or API tokens.

  **Recommended Agent Profile**:
  - **Category**: `unspecified-high`
    - Reason: cluster-level infrastructure with provider-specific constraints.
  - **Skills**: `kustomize-render-guard`, `secrets-safety-check`, `yaml-policy-lint`
    - `kustomize-render-guard`: validates issuers render cleanly.
    - `secrets-safety-check`: avoids credential leaks in issuer configuration.
    - `yaml-policy-lint`: keeps the manifests consistent with repo conventions.
  - **Skills Evaluated but Omitted**:
    - `flux-reconcile-playbook`: defer until the final validation task.

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 1
  - **Blocks**: Task 3, Task 4
  - **Blocked By**: None

  **References**:
  - `infrastructure/traefik/helmrelease.yaml:19-34` - example of how shared infra services are installed.
  - `clusters/prod/infrastructure.yaml:15-18` - prod substitution wiring to respect.
  - `clusters/staging/infrastructure.yaml:15-18` - staging currently has no substitution, so the plan must avoid depending on it.
  - cert-manager best practice: separate cluster issuer strategy from DNS record management.

  **WHY Each Reference Matters**:
  - Existing infra patterns define how new cluster services should be wired.
  - Flux substitution behavior differs between staging and prod; the plan must not assume they are identical.

  **Acceptance Criteria**:
  - [ ] `kustomize build infrastructure` succeeds.
  - [ ] Rendered cert-manager resources exist in both overlays.
  - [ ] Staging issuer config is compatible with Cloudflare DNS-01.
  - [ ] Prod issuer config does not depend on public DNS ownership.

- [ ] 3. Align workload hostnames with Traefik CRD source-of-truth and external-dns zones

  **What to do**:
  - Update the representative workload ingress route(s) so host rules match the chosen DNS zones.
  - Keep `IngressRoute`/`IngressRouteTCP` as the routing model unless a plain `Ingress` is explicitly needed.
  - Ensure the hostnames follow the shared naming scheme across staging and prod.
  - Only add DNS annotations if a specific resource needs to override defaults.

  **Must NOT do**:
  - Do not switch the repo wholesale to standard Ingress objects.
  - Do not let TCP routes drive DNS records.
  - Do not create overlapping hostnames across public and internal zones.

  **Recommended Agent Profile**:
  - **Category**: `unspecified-high`
    - Reason: workload-level changes tied to cluster exposure rules.
  - **Skills**: `yaml-policy-lint`, `gitops-scope-router`, `kustomize-render-guard`
    - `yaml-policy-lint`: preserves manifest consistency.
    - `gitops-scope-router`: identifies the exact workload overlays affected.
    - `kustomize-render-guard`: validates the route changes render correctly.
  - **Skills Evaluated but Omitted**:
    - `secrets-safety-check`: no secret-bearing manifests are expected here.

  **Parallelization**:
  - **Can Run In Parallel**: NO
  - **Parallel Group**: Sequential after Tasks 1 and 2
  - **Blocks**: Task 4
  - **Blocked By**: Tasks 1, 2

  **References**:
  - `apps/devenv/overlays/prod/ingressroute.yaml:1-14` - representative Traefik `IngressRoute` host rule.
  - `apps/devenv/overlays/ssh-native/ingressroute-tcp.yaml:1-13` - TCP route that should remain separate from DNS automation.
  - external-dns guidance: Traefik CRD host rules are the source of truth for `traefik-proxy` source mode.

  **WHY Each Reference Matters**:
  - The existing ingress routes show the canonical Traefik CRD patterns already in use.
  - The TCP route is a reminder not to overgeneralize DNS publication to all Traefik resources.

  **Acceptance Criteria**:
  - [ ] Rendered ingress route hostnames match `staging.sneakysquid.xyz` / `staging.lan`-based naming.
  - [ ] DNS record candidates are derived from Traefik host rules.
  - [ ] No new host conflicts exist across overlays.

- [ ] 4. Run full render and Flux reconciliation validation

  **What to do**:
  - Build the modified kustomizations and parent aggregations.
  - Reconcile Flux on the affected cluster paths.
  - Verify external-dns, cert-manager, and Traefik controllers all settle cleanly.
  - Capture evidence for render and reconcile results.
  - If reconcile fails, revert the last manifest set and re-run renders before another sync.

  **Must NOT do**:
  - Do not skip the parent aggregation builds.
  - Do not rely on manual visual inspection.
  - Do not accept unresolved render warnings.

  **Recommended Agent Profile**:
  - **Category**: `unspecified-high`
    - Reason: cross-cutting validation across multiple controller stacks.
  - **Skills**: `kustomize-render-guard`, `flux-reconcile-playbook`, `yaml-policy-lint`
    - `kustomize-render-guard`: catches broken manifests before reconcile.
    - `flux-reconcile-playbook`: required for live-state validation.
    - `yaml-policy-lint`: helps spot structural issues in rendered YAML.
  - **Skills Evaluated but Omitted**:
    - `secrets-safety-check`: already covered in the authoring tasks.

  **Parallelization**:
  - **Can Run In Parallel**: NO
  - **Parallel Group**: Final validation wave
  - **Blocks**: None
  - **Blocked By**: Tasks 1, 2, 3

  **References**:
  - `clusters/staging/kustomization.yaml:3-6` - staging aggregation root.
  - `clusters/staging/infrastructure.yaml:6-14` - staging Flux infra target.
  - `clusters/prod/infrastructure.yaml:6-18` - prod Flux infra target.
  - `infrastructure/kustomization.yaml:1-4` - shared infra aggregation root.

  **WHY Each Reference Matters**:
  - These are the exact reconciliation entry points that must remain valid for the plan to work.

  **Acceptance Criteria**:
  - [ ] `kustomize build infrastructure/overlays/staging` passes.
  - [ ] `kustomize build infrastructure/overlays/prod` passes.
  - [ ] `kustomize build infrastructure` passes.
  - [ ] `flux reconcile kustomization infrastructure -n flux-system --with-source` completes without errors.
  - [ ] Evidence files exist under `.sisyphus/evidence/` for render and reconcile checks.

---

## Success Criteria

### Verification Commands
```bash
kustomize build infrastructure/overlays/staging
kustomize build infrastructure/overlays/prod
kustomize build clusters/staging
kustomize build clusters/prod
kustomize build infrastructure
 kustomize build apps/devenv/overlays/prod
 kustomize build apps/devenv/overlays/ssh-native
flux reconcile kustomization infrastructure -n flux-system --with-source
```

### Final Checklist
- [ ] Staging uses Cloudflare and public DNS only
- [ ] Prod uses Pi-hole and internal DNS only
- [ ] Traefik host rules drive DNS publication
- [ ] cert-manager is present and separated from DNS logic
- [ ] No generated artifacts were edited

---

## PLAN STATUS

- status: COMPLETED
- completed_at: 2026-03-24T13:30:00.000Z
- completed_by_session: ses_2e0266693ffepKMOaaTVMp9YDK

All tasks in this plan have been implemented in the repository and verified with local kustomize renders. See `.sisyphus/evidence/` for rendered outputs and failure notes for cluster-level checks (requires KUBECONFIG).
