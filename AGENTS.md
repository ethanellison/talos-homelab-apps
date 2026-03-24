# AGENTS.md

Repository guidance for agentic coding assistants operating in `talos-homelab-apps`.

## 1) Repository purpose

- Flux GitOps manifest repository for homelab Kubernetes clusters.
- Environments currently in use: `staging` and `prod`.
- Main artifact type: YAML manifests.
- No application runtime project (`package.json`, `go.mod`, `pyproject.toml`) exists here.

## 2) Repository layout

- `clusters/`
  - Cluster-level Flux `Kustomization` resources and wiring.
  - Contains `flux-system` bootstrap artifacts and environment aggregation.
- `infrastructure/`
  - Shared infrastructure modules (example: Traefik) and overlays.
- `apps/`
  - App manifests and environment-specific overlays.

## 3) Canonical toolchain

From `mise.toml`:

- `flux2 = latest`
- `kubectl = latest`
- `kustomize = latest`

Use these tools as source of truth for validation and operations.

## 4) Build / lint / test commands

This repo has no unit-test framework and no dedicated lint command.
Validation is render-based and reconciliation-based.

### 4.1 Primary render checks

Run for relevant changed scopes:

```bash
kustomize build clusters/staging
kustomize build infrastructure
kustomize build apps/devenv/overlays/staging
```

Fallback equivalents:

```bash
kubectl kustomize clusters/staging
kubectl kustomize infrastructure
kubectl kustomize apps/devenv/overlays/staging
```

### 4.2 Single-test equivalent (fast per-change check)

Use one targeted render as closest equivalent of a single test:

```bash
# App scope
kustomize build apps/devenv/overlays/staging

# Infra scope
kustomize build infrastructure/traefik
```

Rule: run single-scope first, then run at least one parent aggregation render.

### 4.3 Flux reconcile checks

Run after Flux/Kustomization/Helm changes affecting live state:

```bash
flux reconcile source git flux-system -n flux-system
flux reconcile kustomization flux-system -n flux-system
flux reconcile kustomization infrastructure -n flux-system
flux reconcile kustomization apps -n flux-system --with-source
```

### 4.4 Operational status checks

```bash
kubectl get kustomizations -n flux-system
kubectl get helmreleases -A
kubectl get pods -n traefik
kubectl get crds | grep -i traefik
```

## 5) Required change workflow

For every manifest change:

1. Edit the smallest correct scope.
2. Build directly modified kustomization path(s).
3. Build at least one parent aggregation path (`clusters/staging` or `infrastructure`).
4. Reconcile in Flux when runtime behavior changes.
5. Leave no unresolved placeholders unless intentionally resolved by Flux substitution.

### 5.1 Decision quality and non-blind-agreement protocol

- Do not agree blindly with user requests when a safer, simpler, or more correct approach exists.
- Before executing non-trivial plans, run a critique-first pass:
  1. State assumptions.
  2. Identify risks and likely failure modes.
  3. Propose 1-3 viable alternatives with tradeoffs.
  4. Recommend the best option and explain why others are weaker.
- If the user-provided plan is suboptimal, explicitly say so and propose a minimal-change better path.
- Prefer constructive pushback with evidence (commands, repo references, or rendered output), not opinion-only disagreement.

### 5.2 Mandatory low-confidence escalation

- If confidence is `low` for plan correctness, safety, or feasibility, escalation is required before implementation.
- Required escalation sequence:
  1. Run `plan-thinking-critic` to evaluate plan quality and reasoning gaps.
  2. Consult `Oracle` for independent review of architecture/logic/risk.
  3. Synthesize both outputs into a revised execution plan.
- Do not proceed to implementation until the revised plan has clear verification steps and resolved critical findings.
- Treat any unresolved `critical` findings from `plan-thinking-critic` or `Oracle` as a blocker.

## 6) Code style guidelines

### 6.1 Formatting

- Use 2-space indentation.
- Keep key order stable and readable (`apiVersion`, `kind`, `metadata`, `spec`).
- Quote strings containing template syntax, special characters, or CLI flags.
- Do not commit commented-out dead configuration.

### 6.2 Naming conventions

- `metadata.name`: lowercase kebab-case and stable.
- `metadata.namespace`: explicit on namespaced resources.
- Filenames: lowercase kebab-case by resource intent.
- Canonical filenames include:
  - `kustomization.yaml`
  - `helmrelease.yaml`
  - `helmrepository.yaml`
  - `ingressroute.yaml`
  - `namespace.yaml`

### 6.3 Imports, typing, naming, and scripts

- Imports: N/A (YAML-first repository).
- Static typing: N/A; correctness is Kubernetes schema/API-version driven.
- If scripts are added, use strict shell mode: `set -euo pipefail`.
- Script/function names should be descriptive and consistent.

### 6.4 Kustomize and Flux conventions

- Every deployable directory should contain `kustomization.yaml`.
- Keep `resources:` paths relative, explicit, and logically ordered.
- Prefer composable modules aggregated by parent kustomizations.
- Use overlays for environment-specific differences.
- Keep Flux `Kustomization` resources under `clusters/*`.
- Keep `sourceRef` consistent with `GitRepository/flux-system` unless intentionally changed.
- Use `dependsOn` for ordering (infrastructure before apps).
- Use `postBuild.substituteFrom` for runtime substitutions.

### 6.5 Error handling and safety

- Treat render failure as hard failure; fix root cause before proceeding.
- Never commit secrets, kubeconfigs, credentials, private keys, or tokens.
- Prefer additive and reversible manifest changes.
- Avoid destructive operations without explicit operator intent.
- Treat generated `gotk-*` manifests as generated artifacts; avoid manual edits unless explicitly requested.

## 7) Repository-specific patterns

- Canonical staging app overlay path: `./apps/devenv/overlays/staging`.
- Staging apps depend on staging infrastructure via `dependsOn`.
- Staging ingress uses `${STAGING_LB_IP}` substitution.
- Substitution source: `ConfigMap/staging-vars` in namespace `flux-system`.

## 8) Cursor and Copilot rules

Checked paths and currently absent:

- `.cursor/rules/`
- `.cursorrules`
- `.github/copilot-instructions.md`

Until those files exist, this `AGENTS.md` is authoritative for agent behavior.

## 9) Git and PR hygiene

- Keep commits small, focused, and deployable.
- Use concise plain-English commit messages.
- Avoid mixing unrelated changes in one commit.
- Validate all affected kustomization roots before PR.

Pre-PR minimum:

```bash
git status
kustomize build clusters/staging
kustomize build infrastructure
kustomize build apps/devenv/overlays/staging
```

## 10) Quick agent heuristics

- Ordering changed? Verify `dependsOn` and cluster root wiring.
- Endpoint/IP changed? Prefer substitution over hardcoding.
- Editing generated `gotk-*` files? Stop unless explicitly requested.
- Missing local binaries? Report exact failing command and required install.

End of file.
