# Developer Shell Access Strategy

## TL;DR

> **Quick Summary**: Rework `devenv` into a **terminal-only** shell/workspace strategy with `tmux` persistence and `opencode` inside the runtime. This plan compares **Coder workspaces** against raw shell access, recommends **Tailscale SSH-first for prod**, and documents **Cloudflare Access + Tunnel + OIDC** (via `auth.sneakysquid.xyz`) as the staging / alternate path using terminal-native SSH. For the current repo-safe scope, keep manifests unchanged and treat the access gate as external to GitOps until a dedicated SSH-native overlay is defined.
>
> **Deliverables**:
> - Reworked `devenv` shell strategy, replacing browser-terminal assumptions
> - Coder workspace option evaluated alongside raw shell access
> - Side-by-side Tailscale and Cloudflare access options, with explicit primary/alternate recommendation
> - Cloudflare Access + Tunnel + OIDC plan using `auth.sneakysquid.xyz` and terminal-native SSH wording
> - `tmux` session persistence for reconnects and multiple sessions
> - Validation checklist using repo-native render, reconcile, and cluster checks
> - Skills-led execution guidance with commands reserved for verification
>
> **Estimated Effort**: Medium
> **Parallel Execution**: YES - 2 waves
> **Critical Path**: confirm terminal-only model → design access paths → validate renders, reconcile, and exposure

---

## Context

### Original Request
The user wants to rework the existing `devenv` direction toward direct terminal access. They want options for a Tailscale-style access model and a Cloudflare-style access model, with Cloudflare using `auth.sneakysquid.xyz` as its OIDC identity provider. They want `tmux`, multiple shell sessions, and `opencode` inside the environment, and they do **not** want browser SSH or a browser IDE.
They also want Coder workspaces considered as an alternative workspace layer behind Tailscale and/or Cloudflare.
This plan is recommendation-first: it documents both access models, but keeps Tailscale SSH as the primary production recommendation and Cloudflare Access + Tunnel + native SSH as the staging/alternate path.

### Interview Summary
**Key Discussions**:
- Existing repo has a `devenv` app.
- Current `devenv` is browser-first (`ttyd` + `tmux` over HTTP), not direct SSH.
- User wants **both** Tailscale-style and Cloudflare-style access options.
- User wants **Coder workspaces** considered as one option among others.
- User wants **Tailscale-first for prod** and **Cloudflare as a simpler fallback/alternate for staging**.
- Cloudflare should use `auth.sneakysquid.xyz` as the existing OIDC identity provider.
- User scope is **single developer**.
- Browser SSH / browser terminal is explicitly out of scope.

**Research Findings**:
- Private SSH behind VPN/zero-trust is the safest baseline for a single developer.
- `devenv` currently exposes a browser terminal, with staging using a LoadBalancer and prod using Traefik ingress.
- Direct SSH into app pods is the least recommended path; a dedicated shell runtime behind an access gate is the cleaner model.
- Cloudflare-style access should be modeled as Access + Tunnel in front of native SSH using OIDC.
- Tailscale-style access should be modeled as Tailscale SSH with device/user identity controls.
- `auth.sneakysquid.xyz` exists, so the Cloudflare OIDC dependency is available rather than hypothetical.

### Metis Review
**Identified Gaps** (addressed):
- Need to decide the primary access primitive → plan assumes terminal-only SSH access.
- Need to position both models clearly → plan recommends Tailscale for prod and Cloudflare for staging/alternate.
- Need to lock down scope creep → plan excludes browser SSH and browser IDE paths.
- Need executable validation criteria → plan uses render, exposure, and connectivity checks.
- Need explicit operational checks → plan now includes Flux reconcile and rollback/observability validation.

---

## Work Objectives

### Core Objective
Rework `devenv` into a terminal-only developer shell/workspace path on Kubernetes that supports reconnects via `tmux`, runs `opencode` in the runtime, and is organized into three explicit layers: shell runtime, access gate, and workspace orchestration. Compare direct shell and Coder workspaces, with Tailscale SSH-first for production and Cloudflare Access + Tunnel + native SSH as the alternate/staging path.

### Concrete Deliverables
- A documented target architecture for the terminal-only shell environment.
- A documented comparison that includes Coder workspaces as an option.
- A clear recommendation: Tailscale for prod, Cloudflare for staging/alternate.
- A Cloudflare integration plan using `auth.sneakysquid.xyz` as OIDC.
- Updated manifests/wiring for the chosen access path(s).
- Validation output proving the chosen exposure and runtime behavior.
- A concrete per-repo `.devcontainer` provisioning path.
- A rollback and observability checklist for access changes.

### Coder Ownership Decision
- Coder is **not** the default owner of the current provisioning lifecycle in this recommendation.
- The lifecycle belongs to the workspace/orchestration layer in this plan.
- Coder may be adopted later as an implementation detail, but is not a required platform assumption.

### Architecture Layers
1. **Shell runtime**: `tmux`, `opencode`, repo-specific `.devcontainer` image/config.
2. **Access gate**: Tailscale SSH for prod; Cloudflare Access + Tunnel + OIDC for staging/alternate.
3. **Workspace orchestration**: on-demand create/start/stop/resume/destroy lifecycle; Coder may implement this later but does not own it now.

### Definition of Done
- [ ] The chosen shell access path is terminal-only.
- [ ] Coder is evaluated as one option among others.
- [ ] `tmux` is the persistence mechanism for reconnects and multiple sessions.
- [ ] `opencode` runs inside the same isolated developer runtime.
- [ ] Repo render checks succeed for all touched kustomization roots.

### Must Have
- Single-developer workflow.
- Terminal-only access path.
- No browser SSH or `ttyd` UI.
- Clear separation between access gate and shell runtime.
- Coder remains an option, but does not own the lifecycle in the current recommendation.
- Explicit `.devcontainer` support for multiple repos.
- Terminal-native Cloudflare SSH terminology (Access + Tunnel + `cloudflared`), not browser SSH.

### Scope Clarification
- This plan documents both Tailscale and Cloudflare access models.
- The recommended production path is Tailscale-first.
- The Cloudflare path is a documented staging / alternate implementation using Access + Tunnel + OIDC, with `auth.sneakysquid.xyz` as a confirmed identity provider.
- Coder is evaluated as a competitor/alternative, not as a required owner.

### Execution Style
- **Skills are the repeatable execution primitive** for agents working this plan.
- **Commands are the verification contract** and belong in acceptance criteria / QA only.
- Keep tasks spec-like: objective, boundaries, required skills, and measurable checks.

### Terminology Guidance
- Prefer **Tailscale SSH** over generic Tailscale access.
- Prefer **Cloudflare Access + Tunnel + native SSH** over generic “SSH-only”.
- Use **cloudflared** when referring to terminal-native Cloudflare transport.

### Provisioning Lifecycle
- **Create**: provision a new workspace only when a repo/environment is requested.
- **Start**: attach storage, start the runtime, and expose the chosen access path.
- **Stop**: pause the runtime while preserving workspace state.
- **Resume**: restart the runtime and reattach to the same `tmux` session if present.
- **Destroy**: remove the workspace and clean up its associated resources when no longer needed.
- **Ownership**: lifecycle belongs to the workspace/orchestration layer; Coder can implement it later, but is not the owner in the current plan.

### Must NOT Have (Guardrails)
- No browser IDE scope creep.
- No assumption that Coder must be chosen.
- No multi-user platform design.
- No shared shell accounts.
- No secrets baked into images or manifests.
- No direct SSH into application pods as the primary UX.

---

## Verification Strategy (MANDATORY)

> **UNIVERSAL RULE: ZERO HUMAN INTERVENTION**
>
> All verification must be executable by the agent.

### Test Decision
 - **Infrastructure exists**: NO
 - **Automated tests**: None
 - **Framework**: none

### Agent-Executed QA Scenarios (MANDATORY — ALL tasks)

**Scenario: Render the touched Kubernetes scopes**
  Tool: Bash
  Preconditions: Repo dependencies available (`kustomize` installed)
  Steps:
    1. Run `kustomize build apps/devenv/overlays/staging`
    2. Run `kustomize build apps/devenv/overlays/prod`
    3. Run `kustomize build infrastructure`
    3. Run `kustomize build clusters/staging`
    4. Run `kustomize build clusters/prod`
  Expected Result: All renders succeed without schema or reference errors
  Failure Indicators: Missing resources, invalid fields, or render failures
  Evidence: `.sisyphus/evidence/render-touched-scopes.txt`

**Scenario: Reconcile Flux for touched scopes**
  Tool: Bash
  Preconditions: Cluster access and Flux CLI available
  Steps:
    1. Run `flux reconcile source git flux-system -n flux-system`
    2. Run `flux reconcile kustomization flux-system -n flux-system`
    3. Run `flux reconcile kustomization infrastructure -n flux-system`
    4. Run `flux reconcile kustomization apps -n flux-system --with-source`
  Expected Result: Flux reports successful reconciliation without kustomization errors
  Failure Indicators: Reconcile failures, dependency errors, or source fetch issues
  Evidence: `.sisyphus/evidence/flux-reconcile-touched-scopes.txt`

**Scenario: Confirm runtime exposure matches the terminal-only design**
  Tool: Bash
  Preconditions: Cluster resources applied
  Steps:
    1. Run `kubectl get svc,ingress,ingressroute -A`
    2. Confirm no browser-terminal service is exposed
    3. Confirm only the intended SSH/access-gate path is present
  Expected Result: Exposure matches the terminal-only strategy
  Failure Indicators: Unexpected browser terminal, conflicting ingress, or duplicate access paths
  Evidence: `.sisyphus/evidence/exposure-terminal-only.txt`

**Scenario: Verify the shell runtime supports tmux persistence**
  Tool: Bash
  Preconditions: Developer shell workload running
  Steps:
    1. Connect using the chosen access method
    2. Start or attach to a `tmux` session
    3. Disconnect and reconnect
    4. Reattach to the same session
  Expected Result: Session persists across reconnects
  Failure Indicators: Lost session state, new session created unexpectedly, or tmux unavailable
  Evidence: `.sisyphus/evidence/tmux-persistence.txt`

**Scenario: Verify `opencode` launches in the developer runtime**
  Tool: Bash
  Preconditions: Shell workload running with `opencode` installed
  Steps:
    1. Open a shell in the runtime
    2. Run `opencode --help` or the agreed startup command
    3. Confirm the process exits cleanly or starts as expected
  Expected Result: `opencode` is usable inside the environment
  Failure Indicators: Missing binary, permission errors, or runtime crashes
  Evidence: `.sisyphus/evidence/opencode-runtime.txt`

---

## Execution Strategy

### Parallel Execution Waves

```
Wave 1 (Start Immediately):
├── Task 1: Confirm terminal-only architecture and guardrails
└── Task 2: Audit current devenv exposure and reuse/replacement options

Wave 2 (After Wave 1):
├── Task 3: Update manifests for the chosen access path(s)
├── Task 4: Validate render, exposure, and shell persistence behavior
└── Task 5: Document rollback and observability for access-gate changes

Critical Path: Task 1 → Task 3 → Task 4 → Task 5
```

### Dependency Matrix

| Task | Depends On | Blocks | Can Parallelize With |
|------|------------|--------|----------------------|
| 1 | None | 3 | 2 |
| 2 | None | 3 | 1 |
| 3 | 1, 2 | 4, 5 | None |
| 4 | 3 | 5 | None |
| 5 | 3 | None | None |

---

## TODOs

> Implementation + Test = ONE Task. Never separate.

- [x] 1. Confirm the terminal-only shell architecture and guardrails

  **What to do**:
  - Finalize the shell-first model: terminal-only access, `tmux` persistence, `opencode` inside the runtime, and repo-specific `.devcontainer` usage.
  - Decide whether the existing `devenv` app is repurposed as the shell entrypoint or replaced.
  - Define what is explicitly out of scope: browser IDE, browser SSH, multi-user platform, public SSH.

  **Must NOT do**:
  - Do not introduce a browser IDE path.
  - Do not introduce a browser SSH path.
  - Do not design for multiple users yet.
  - Do not expose app pods directly over public SSH.

  **Recommended Agent Profile**:
  > Select category + skills based on task domain. Justify each choice.
  - **Category**: `unspecified-high`
    - Reason: Architecture and platform decision-making with security trade-offs.
  - **Skills**: [`gitops-scope-router`, `yaml-policy-lint`]
    - `gitops-scope-router`: maps the work to the correct repo scope before touching manifests.
    - `yaml-policy-lint`: enforces repo-specific manifest conventions and naming safety.
  - **Skills Evaluated but Omitted**:
    - `playwright`: not a browser task.
    - `frontend-ui-ux`: not a UI design task.

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 1 (with Task 2)
  - **Blocks**: Task 3
  - **Blocked By**: None

  **References** (CRITICAL - Be Exhaustive):
  - `apps/devenv/overlays/staging/kustomization.yaml` - current staging entrypoint and resource composition.
  - `apps/devenv/overlays/prod/kustomization.yaml` - current prod entrypoint and replacement options.
  - `apps/devenv/overlays/prod/ingressroute.yaml` - current browser-access exposure model.
  - `clusters/staging/apps.yaml` - Flux wiring for the app scope.

  **Acceptance Criteria**:
  - The target architecture is written down in the plan and reflected in the repo strategy.
  - The chosen access model is explicitly terminal-only.
  - The plan clearly states whether the current `devenv` app is reused or replaced.
  - The plan explicitly splits shell runtime, access gate, and orchestration responsibilities.

  **Agent-Executed QA Scenarios (MANDATORY — per-scenario, ultra-detailed):**

  ```
  Scenario: Architecture decision is internally consistent
    Tool: Bash
    Preconditions: None
    Steps:
      1. Read the plan and confirm the target model is terminal-only.
      2. Confirm the plan excludes browser IDE scope creep.
      3. Confirm the plan chooses one shell runtime model.
    Expected Result: One coherent architecture direction is stated without conflicting access models
    Failure Indicators: Mixed browser/terminal requirements or unresolved primary UX
    Evidence: Plan file content

  Scenario: Existing devenv exposure is recognized
    Tool: Bash
    Preconditions: Repo available
    Steps:
      1. Inspect the current devenv overlays.
      2. Confirm staging uses LoadBalancer exposure and prod uses Traefik ingress.
    Expected Result: The current browser-terminal path is understood and documented
    Failure Indicators: Missing or incorrect exposure model assumptions
    Evidence: Manifest paths and render output
  ```

- [x] 2. Audit the existing `devenv` app as a reuse or replacement candidate

  **What to do**:
  - Review the current `ttyd`/`tmux` setup as the existing baseline.
  - Decide whether the current app can be adapted into a terminal-only shell entrypoint or should be repurposed/replaced.
  - Identify any clean reuse opportunities in namespace, deployment, or Flux wiring.

  **Must NOT do**:
  - Do not keep the browser-terminal path as the default.
  - Do not widen scope to team collaboration or shared workspaces.

  **Recommended Agent Profile**:
  > Select category + skills based on task domain. Justify each choice.
  - **Category**: `deep`
    - Reason: Requires reading existing manifests and inferring architectural fit.
  - **Skills**: [`git-master`, `gitops-scope-router`]
    - `git-master`: helpful for tracing history if needed.
    - `gitops-scope-router`: helps locate the correct cluster/app scope and parent aggregation path.
  - **Skills Evaluated but Omitted**:
    - `playwright`: no browser interaction required.

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 1 (with Task 1)
  - **Blocks**: Task 3
  - **Blocked By**: None

  **References** (CRITICAL - Be Exhaustive):
  - `apps/devenv/overlays/staging/deployment.yaml` - current `ttyd` command and tmux usage.
  - `apps/devenv/overlays/staging/service.yaml` - current service exposure for shell access.
  - `apps/devenv/namespace.yaml` - namespace reuse constraints.
  - `clusters/prod/apps.yaml` - app-level Flux wiring in production.

  **Acceptance Criteria**:
  - The current browser-first architecture is documented as baseline.
  - Reuse vs replacement decision is explicit.
  - No ambiguity remains about the existing `ttyd` flow.
  - The `.devcontainer` angle is captured as a first-class provisioning input.

  **Agent-Executed QA Scenarios (MANDATORY — per-scenario, ultra-detailed):**

  ```
  Scenario: Baseline exposure matches repo manifests
    Tool: Bash
    Preconditions: Repo available
    Steps:
      1. Inspect the devenv deployment and service manifests.
      2. Confirm the shell is served by `ttyd` on port 7681.
      3. Confirm tmux is invoked in the container command.
    Expected Result: Existing browser shell baseline is accurately captured
    Failure Indicators: Missing ttyd/tmux linkage or incorrect port assumptions
    Evidence: Manifest content and build output
  ```

- [x] 3. Document the repo-safe access-path boundary (no manifest edits)

  **What to do**:
  - Treat this as a documentation-only scope for the current repo-safe recommendation; do not change manifests in this repo yet.
  - Confirm the repo-safe decision is to keep the current `devenv` manifests unchanged in this scope.
  - Document that Tailscale SSH and Cloudflare Access + Tunnel + native SSH are external access-layer prerequisites, not repo-owned manifests yet.
  - Preserve `tmux` as the session layer and ensure `opencode` is available in the shell runtime.
  - State that any future GitOps-owned SSH-native path should be added as a new overlay/path, not by mutating the current `ttyd` overlays in place.
  - Add rollback and observability notes for the external access gate boundary.

  **Must NOT do**:
  - Do not create direct public SSH exposure.
  - Do not introduce a second, competing browser-based access path.
  - Do not bake secrets into the image.

  **Recommended Agent Profile**:
  > Select category + skills based on task domain. Justify each choice.
  - **Category**: `unspecified-high`
    - Reason: Cross-cutting platform update with security implications.
  - **Skills**: [`gitops-scope-router`, `kustomize-render-guard`, `yaml-policy-lint`, `secrets-safety-check`, `substitution-contract-check`]
    - `gitops-scope-router`: confirms all touched resources are validated at the right scope.
    - `kustomize-render-guard`: ensures the manifest changes render safely.
    - `yaml-policy-lint`: checks naming, formatting, and namespace conventions.
    - `secrets-safety-check`: guards against accidental secret introduction.
    - `substitution-contract-check`: validates any Flux substitutions remain resolvable.
  - **Skills Evaluated but Omitted**:
    - `frontend-ui-ux`: not applicable.

  **Parallelization**:
  - **Can Run In Parallel**: NO
  - **Parallel Group**: Sequential
  - **Blocks**: Task 4
  - **Blocked By**: Tasks 1, 2

  **References** (CRITICAL - Be Exhaustive):
  - `clusters/staging/kustomization.yaml` - cluster wiring constraints and dependency order.
  - `clusters/prod/kustomization.yaml` - prod composition pattern.
  - `apps/devenv/overlays/prod/ingressroute.yaml` - current ingress route pattern if a fallback path is kept.
  - `apps/devenv/overlays/prod/deployment.yaml` - current runtime shape for comparison.

  **Acceptance Criteria**:
  - The plan explicitly says the current repo-safe scope makes no manifest edits.
  - The shell runtime includes `tmux` and `opencode` support.
  - The access-gate primitive is named explicitly for both Tailscale SSH and Cloudflare Access + Tunnel + native SSH.
  - The follow-up path for a future SSH-native overlay is documented.
  - Rollback and audit/observability checks are included.

  **Agent-Executed QA Scenarios (MANDATORY — per-scenario, ultra-detailed):**

  ```
  Scenario: Private shell exposure is the only active path
    Tool: Bash
    Preconditions: Cluster resources applied
    Steps:
      1. Run `kubectl get svc,ingress,ingressroute -A`.
      2. Confirm the shell endpoint is not exposed as a browser terminal UI.
      3. Confirm the intended private gate is the only access route.
    Expected Result: No browser terminal service exists for the shell runtime
    Failure Indicators: Additional public listener or conflicting ingress path
    Evidence: Terminal output captured

  Scenario: tmux and opencode are usable in the runtime
    Tool: Bash
    Preconditions: Shell runtime running
    Steps:
      1. Attach to the shell.
      2. Start or attach a `tmux` session.
      3. Run `opencode --help` or the agreed command.
    Expected Result: tmux session exists and opencode is callable
    Failure Indicators: Missing tmux, missing opencode, or command failure
    Evidence: Terminal transcript captured
  ```

- [x] 4. Validate current baseline and terminal-only design assumptions

  **What to do**:
  - Validate the current repo baseline as a no-change check; do not expect a manifest diff from this plan scope.
  - Run the required render checks for touched scopes.
  - Verify the network exposure matches the terminal-only design.
  - Confirm reconnect behavior with `tmux` and capture terminal evidence.
  - Treat any successful validation as a baseline check, not as evidence of changed manifests in this scope.

  **Must NOT do**:
  - Do not accept human-only validation.
  - Do not leave ambiguous exposure paths untested.

  **Recommended Agent Profile**:
  > Select category + skills based on task domain. Justify each choice.
  - **Category**: `quick`
    - Reason: Straightforward verification and smoke-testing work.
  - **Skills**: [`kustomize-render-guard`, `flux-reconcile-playbook`]
    - `kustomize-render-guard`: repeatable render validation for touched scopes.
    - `flux-reconcile-playbook`: standardizes Flux reconciliation checks after manifest changes.
  - **Skills Evaluated but Omitted**:
    - `playwright`: no browser UI verification required for the chosen strategy.

  **Parallelization**:
  - **Can Run In Parallel**: NO
  - **Parallel Group**: Sequential
  - **Blocks**: None
  - **Blocked By**: Task 3

  **References** (CRITICAL - Be Exhaustive):
  - `apps/devenv/overlays/staging/kustomization.yaml` - render target for staging.
  - `apps/devenv/overlays/prod/kustomization.yaml` - render target for the prod variant and Cloudflare/Tailscale wiring.
  - `infrastructure/` - parent render target for shared infra.
  - `clusters/staging/` - top-level render and wiring verification.
  - `clusters/prod/` - top-level prod render and wiring verification.

  **Acceptance Criteria**:
  - The baseline render commands succeed.
  - Connectivity smoke checks confirm the chosen private access path.
  - The plan remains docs-only for the current repo-safe scope.

  **Agent-Executed QA Scenarios (MANDATORY — per-scenario, ultra-detailed):**

  ```
  Scenario: Render checks pass for touched scopes
    Tool: Bash
    Preconditions: kustomize installed
    Steps:
      1. Run `kustomize build apps/devenv/overlays/staging`.
      2. Run `kustomize build apps/devenv/overlays/prod`.
      3. Run `kustomize build infrastructure`.
      4. Run `kustomize build clusters/staging`.
      5. Run `kustomize build clusters/prod`.
    Expected Result: All builds succeed with no invalid manifest errors
    Failure Indicators: Render failure or schema violations
    Evidence: `.sisyphus/evidence/render-checks.txt`

  Scenario: Reconnect preserves shell state
    Tool: Bash
    Preconditions: Access path available and tmux running
    Steps:
      1. Open a shell session.
      2. Create or attach a tmux session.
      3. Disconnect and reconnect.
      4. Reattach to the same tmux session.
    Expected Result: The prior session is still available
    Failure Indicators: Session loss or unexpected new session creation
    Evidence: `.sisyphus/evidence/reconnect-preserves-shell-state.txt`
  ```

- [x] 5. Document rollback and observability for external access-gate changes

  **What to do**:
  - Define a rollback path for both Tailscale and Cloudflare access changes.
  - Record the operational signals to watch during rollout: Flux health, ingress reachability, and shell session availability.
  - Capture a minimal revert sequence that returns the repo to the last known-good terminal-only configuration.
  - State clearly that the current repo scope makes no manifest changes and that rollout here refers to external access-gate boundary changes only.

  **Must NOT do**:
  - Do not assume manual intervention is the rollback strategy.
  - Do not leave the revert path undocumented.

  **Recommended Agent Profile**:
  > Select category + skills based on task domain. Justify each choice.
  - **Category**: `writing`
    - Reason: Operational documentation and rollback procedure definition.
  - **Skills**: [`flux-reconcile-playbook`, `yaml-policy-lint`]
    - `flux-reconcile-playbook`: gives the rollback section operational command coverage.
    - `yaml-policy-lint`: keeps the rollback docs aligned with repo conventions.

  **Parallelization**:
  - **Can Run In Parallel**: NO
  - **Parallel Group**: Sequential
  - **Blocks**: None
  - **Blocked By**: Task 3

  **References** (CRITICAL - Be Exhaustive):
  - `clusters/staging/kustomization.yaml` - rollback target for staged cluster wiring.
  - `clusters/prod/kustomization.yaml` - rollback target for production cluster wiring.
  - `apps/devenv/overlays/staging/kustomization.yaml` - revert target for staging app exposure.
  - `apps/devenv/overlays/prod/kustomization.yaml` - revert target for prod app exposure.

  **Acceptance Criteria**:
  - Rollback steps are documented for both access variants.
  - Observability signals are explicitly listed with command-based checks.
  - The revert path returns the system to the last known-good terminal-only state.
  - The plan explicitly distinguishes repo-owned manifest changes from external access-layer rollout.

  **Agent-Executed QA Scenarios (MANDATORY — per-scenario, ultra-detailed):**

  ```
  Scenario: Rollback steps are executable from the plan
    Tool: Bash
    Preconditions: Plan file available
    Steps:
      1. Read the rollback section in the plan.
      2. Confirm it names the staging and prod kustomization targets.
      3. Confirm it includes a command-based revert path.
    Expected Result: A concrete, actionable rollback sequence exists
    Failure Indicators: Vague rollback language or missing target paths
    Evidence: `.sisyphus/evidence/rollback-executable.txt`

  Scenario: Observability checks are command-based
    Tool: Bash
    Preconditions: Cluster access available
    Steps:
      1. Run `flux get kustomizations -n flux-system`.
      2. Run `kubectl get pods -n flux-system`.
      3. Run `kubectl get svc,ingress,ingressroute -A`.
    Expected Result: The plan’s observability checks map to real cluster commands
    Failure Indicators: Missing checks or non-command-based verification
    Evidence: `.sisyphus/evidence/observability-command-based.txt`
  ```

---

## Success Criteria

### Verification Commands
```bash
kustomize build apps/devenv/overlays/staging
kustomize build apps/devenv/overlays/prod
kustomize build infrastructure
kustomize build clusters/staging
kustomize build clusters/prod
flux reconcile source git flux-system -n flux-system
flux reconcile kustomization flux-system -n flux-system
flux reconcile kustomization infrastructure -n flux-system
flux reconcile kustomization apps -n flux-system --with-source
```

### Final Checklist
- [ ] The strategy is terminal-only and iPad-friendly.
- [ ] Tailscale is the recommended production path.
- [ ] Cloudflare Access + Tunnel + `auth.sneakysquid.xyz` is documented as the staging / alternate path.
- [ ] Coder workspaces are included as a considered alternative.
- [ ] The current `devenv` browser-terminal baseline is explicitly handled.
- [ ] `tmux` persistence is part of the design.
- [ ] `opencode` is supported in the chosen runtime.
- [ ] All touched render checks pass.
- [ ] Flux reconcile checks pass for touched scopes.
- [ ] Rollback and observability are documented with command-based checks.
