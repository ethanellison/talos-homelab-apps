# DevPod + TTYD Workload Plan

## TL;DR

Bake `devpod` into the `ttyd` image and include the Kubernetes provider so users can provision devcontainer workspaces as pods from inside ttyd. Keep the workflow internal-only to the cluster, reuse the existing `devenv` app wiring, and add the minimum RBAC / ServiceAccount auth needed for DevPod to create and exec into workspace pods.

**Deliverables**
- Updated ttyd image/runtime to include `devpod` and `kubectl`.
- Kubernetes provider packaged or installed into the ttyd runtime (provider.yaml + provider binary or release download access).
- Dedicated ServiceAccount + RBAC for DevPod so it can create pods.
- Verification that the provider is present and the rendered manifests remain clean.

**Estimated Effort**: Medium
**Parallel Execution**: YES - 2 waves
**Critical Path**: image/runtime → provider install → kube-access/RBAC → verification

---

## Context

### Original Request
Update the plan to use DevPod in the ttyd image and ensure the Kubernetes provider is included.

### Interview Summary
**Key Discussions**
- DevPod should be available from inside ttyd so users can provision devcontainer pods in-cluster.
- The Kubernetes provider is required because DevPod uses Kubernetes pods for workspaces.
- The workflow remains internal-only; no public exposure is needed for DevPod itself.

**Research Findings**
- DevPod is a CLI binary that can be installed directly into an image.
- The Kubernetes provider uses `kubectl` to create/exec workspace pods and needs in-cluster ServiceAccount credentials plus RBAC.
- The provider is distributed as provider.yaml plus helper binary/assets; DevPod can install it from a local file or release.
- For devcontainer builds, DevPod may use init containers / kaniko / build helper images.
- The repo’s existing `devenv` app wiring is the right pattern to reuse for shell access and internal-only cluster access.

**Metis Review**
**Identified Gaps (addressed)**
- Need to define how DevPod will be made available inside ttyd → resolved as image/runtime inclusion.
- Need to ensure the Kubernetes provider is available → resolved as provider package/install step.
- Need to ensure the cluster API access is sufficient → resolved as explicit RBAC + ServiceAccount tasks.
- Need to ensure the provider binary and kubectl are present in the ttyd image → resolved as explicit image packaging requirements.

---

## Work Objectives

### Core Objective
Provide a ttyd-based developer environment that can run DevPod and use the Kubernetes provider to provision devcontainer pods inside the cluster, using in-cluster ServiceAccount auth.

### Concrete Deliverables
- Updated `apps/devenv/overlays/staging/deployment.yaml` and `apps/devenv/overlays/prod/deployment.yaml` so the ttyd image includes `devpod` and `kubectl`.
- Updated ttyd startup/runtime so DevPod can use the Kubernetes provider.
- Dedicated ServiceAccount / RBAC wiring that allows DevPod to create pods and exec into them.
- Provider bundle/config inside the image or container startup (provider.yaml + provider binary or release access).

### Definition of Done
- [ ] `devpod` binary is present in the ttyd image and runnable from inside the pod.
- [ ] Kubernetes provider is installed/available to DevPod.
- [ ] ttyd pod authenticates to the cluster using a dedicated ServiceAccount.
- [ ] DevPod can provision a workspace pod in the cluster.
- [ ] `kustomize build` succeeds for affected app overlays and cluster roots.

### Must Have
- DevPod must be usable from inside ttyd.
- Kubernetes provider must be included.
- The workflow must remain internal-only to the cluster.
- ttyd must authenticate via in-cluster ServiceAccount, not a baked kubeconfig.
- The ttyd image must include `kubectl` and either a local provider bundle or access to provider release assets.

### Must NOT Have (Guardrails)
- No new public ingress/LB for DevPod.
- No devpod controller/service if a simple ttyd image + provider path is sufficient.
- No unnecessary refactor of unrelated app manifests.

---

## Verification Strategy (MANDATORY)

> **UNIVERSAL RULE: ZERO HUMAN INTERVENTION**
>
> All verification must be executable by the agent with commands or tooling.

### Test Decision
- **Infrastructure exists**: YES
- **Automated tests**: None
- **Framework**: none

### Agent-Executed QA Scenarios

#### Scenario: ttyd image includes DevPod and kubectl
  Tool: Bash
  Preconditions: ttyd image updated
  Steps:
    1. Exec into the ttyd pod or inspect the image build output
    2. Run `devpod --version`
    3. Run `kubectl version --client`
  Expected Result: Both binaries are present and runnable
  Evidence: terminal output

#### Scenario: Kubernetes provider is available to DevPod
  Tool: Bash
  Preconditions: DevPod is installed in the ttyd container
  Steps:
    1. Run the provider install/add command used by the plan
    2. List installed providers or inspect provider config
    3. Confirm Kubernetes provider is present
  Expected Result: DevPod can resolve the Kubernetes provider
  Evidence: terminal output

#### Scenario: DevPod can access the Kubernetes API
  Tool: Bash
  Preconditions: ttyd pod has kubeconfig or serviceaccount credentials
  Steps:
    1. Run a simple `kubectl get pods` from the ttyd pod context
    2. Run a DevPod workspace creation/connect command against the cluster
    3. Confirm the workspace pod is created
  Expected Result: DevPod can create and connect to a Kubernetes workspace pod
  Evidence: terminal output

#### Scenario: No public exposure added for DevPod
  Tool: Bash
  Preconditions: manifests rendered
  Steps:
    1. Search rendered output for DevPod ingress/load balancer resources
    2. Assert none exist
  Expected Result: DevPod remains internal-only
  Evidence: grep output

---

## Execution Strategy

### Wave 1 (Start Immediately)
├── Task 1: Update ttyd image/runtime for DevPod and kubectl
└── Task 2: Include Kubernetes provider in the ttyd runtime

### Wave 2 (After Wave 1)
├── Task 3: Add cluster credentials / RBAC for DevPod
└── Task 4: Verify DevPod can create workspace pods and no public exposure exists

Critical Path: Task 1 → Task 2 → Task 3 → Task 4

---

## TODOs

- [ ] 1. Update ttyd image/runtime for DevPod and kubectl

  **What to do**:
  - Add `devpod` to the ttyd image path used by `apps/devenv/overlays/staging/deployment.yaml` and `apps/devenv/overlays/prod/deployment.yaml`.
  - Ensure `kubectl` is also present in the image.
  - Keep the existing ttyd/tmux startup behavior unless the DevPod entrypoint requires a minimal adjustment.

  **Must NOT do**:
  - Do not change the app’s public exposure model.
  - Do not add a separate DevPod service/ingress.

  **Recommended Agent Profile**:
  - **Category**: `quick`
    - Reason: Mostly image/runtime wiring.
  - **Skills**: `[]`
  - **Skills Evaluated but Omitted**:
    - `playwright`: not browser automation.
    - `git-master`: no history task.

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 1
  - **Blocks**: Task 2, Task 3, Task 4
  - **Blocked By**: None

  **References**:
  - `apps/devenv/overlays/staging/deployment.yaml` - current ttyd command/image pattern.
  - `apps/devenv/overlays/prod/deployment.yaml` - prod ttyd runtime.
  - DevPod install docs - binary packaging pattern.

  **Acceptance Criteria**:
  - [ ] `devpod --version` works in ttyd.
  - [ ] `kubectl version --client` works in ttyd.

- [ ] 2. Include Kubernetes provider in the ttyd runtime

  **What to do**:
  - Package the DevPod Kubernetes provider into the ttyd image or install it at startup.
  - Ensure the provider is added/available before the ttyd shell is used.
  - Use the provider configuration supported by DevPod (`provider.yaml` / provider add flow).
  - Include the provider binary for the image architecture or allow DevPod to download it from the provider release.

  **Must NOT do**:
  - Do not invent a separate controller if provider installation is enough.

  **Recommended Agent Profile**:
  - **Category**: `unspecified-high`
    - Reason: Needs careful runtime/provider configuration, provider bundling, and in-cluster auth.
  - **Skills**: `[]`
  - **Skills Evaluated but Omitted**:
    - `playwright`, `frontend-ui-ux`, `dev-browser`

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 1
  - **Blocks**: Task 3, Task 4
  - **Blocked By**: Task 1

  **References**:
  - DevPod add-provider docs - provider installation flow.
  - DevPod Kubernetes provider docs - provider requirements.
  - DevPod Kubernetes provider docs - provider bundle format and install flow.
  - Kubernetes RBAC patterns for namespace-scoped ServiceAccounts.

  **Acceptance Criteria**:
  - [ ] Kubernetes provider is present and selectable in DevPod.
  - [ ] Kubernetes provider YAML is bundled or retrievable.

- [ ] 3. Add cluster credentials / RBAC for DevPod

  **What to do**:
  - Ensure the ttyd pod has in-cluster ServiceAccount credentials.
  - Add RBAC for the minimum DevPod actions: create/get/list/delete pods, exec into pods, and any PVC operations needed.
  - Keep permissions namespace-scoped if possible.

  **Must NOT do**:
  - Do not grant unnecessary cluster-admin privileges.

  **Recommended Agent Profile**:
  - **Category**: `unspecified-high`
    - Reason: RBAC and in-cluster auth need careful permission scoping.
  - **Skills**: `[]`

  **Parallelization**:
  - **Can Run In Parallel**: NO
  - **Parallel Group**: Sequential
  - **Blocks**: Task 4
  - **Blocked By**: Tasks 1 and 2

  **References**:
  - DevPod Kubernetes provider docs - kubectl wrapper / API access.
  - Existing `devenv` Deployment/Service wiring - where to attach the creds.

  **Acceptance Criteria**:
  - [ ] DevPod can authenticate to the cluster from ttyd via ServiceAccount token.
  - [ ] DevPod can create and exec into a test workspace pod.

- [ ] 4. Verify DevPod provisioning and no public exposure

  **What to do**:
  - Run `kustomize build` for affected app overlays and cluster roots.
  - Confirm DevPod provider is available and workspace pods can be created.
  - Confirm no public exposure is added for DevPod.

  **Must NOT do**:
  - Do not add public ingress/LB as part of verification.

  **Recommended Agent Profile**:
  - **Category**: `quick`
    - Reason: Straightforward render/grep validation.

  **Parallelization**:
  - **Can Run In Parallel**: NO
  - **Parallel Group**: Final verification
  - **Blocks**: None
  - **Blocked By**: Tasks 1, 2, 3

  **References**:
  - `apps/devenv/overlays/staging/*` - render target.
  - `apps/devenv/overlays/prod/*` - render target.
  - Cluster roots in `clusters/staging` and `clusters/prod`.

  **Acceptance Criteria**:
  - [ ] `kustomize build` passes for affected overlays and cluster roots.
  - [ ] DevPod workspace provisioning succeeds in-cluster.
  - [ ] No public DevPod exposure exists.

---

## Success Criteria

### Verification Commands
```bash
kustomize build apps/devenv/overlays/staging
kustomize build apps/devenv/overlays/prod
kustomize build clusters/staging
kustomize build clusters/prod
```

### Final Checklist
- [ ] DevPod is available inside ttyd.
- [ ] Kubernetes provider is included.
- [ ] DevPod can provision a workspace pod.
- [ ] No public DevPod exposure exists.
