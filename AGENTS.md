# AGENTS.md

Repository guidance for agentic coding assistants in `talos-homelab-apps`.

## 1) Repository purpose and layout

- Flux GitOps manifests repo for Kubernetes clusters (staging + prod).
- Main directories:
  - `clusters/` → Flux `Kustomization` objects and cluster wiring
  - `infrastructure/` → shared infrastructure modules (e.g., Traefik)
  - `apps/` → application manifests and environment overlays
- Primary format: YAML manifests.
- No app runtime code package manager (`package.json`, `go.mod`, `pyproject.toml` absent).

## 2) Toolchain baseline

Defined in `mise.toml`:

- `flux2 = latest`
- `kubectl = latest`
- `kustomize = latest`

Assume these CLIs are the canonical workflow tools.

## 3) Build / lint / test commands

This repo has no dedicated linter or unit-test framework; validation is manifest-focused.

### 3.1 Build (render) checks

Run `kustomize build` for each changed scope:

```bash
kustomize build clusters/staging
kustomize build infrastructure
kustomize build apps/devenv/overlays/staging
```

Optional alternative:

```bash
kubectl kustomize clusters/staging
kubectl kustomize infrastructure
kubectl kustomize apps/devenv/overlays/staging
```

### 3.2 Flux reconcile checks

Use after Flux/Kustomization/Helm resource changes:

```bash
flux reconcile source git flux-system -n flux-system
flux reconcile kustomization flux-system -n flux-system
flux reconcile kustomization infrastructure -n flux-system
flux reconcile kustomization apps -n flux-system --with-source
```

### 3.3 Single-test equivalent

No unit tests exist yet; treat a single-target render as per-change testing:

```bash
# App scope
kustomize build apps/devenv/overlays/staging

# Infra scope
kustomize build infrastructure/traefik
```

### 3.4 Operational status checks

```bash
kubectl get kustomizations -n flux-system
kubectl get helmreleases -A
kubectl get pods -n traefik
kubectl get crds | grep -i traefik
```

## 4) Required change workflow

For every manifest update:

1. Edit the smallest correct scope.
2. Build directly modified kustomization path(s).
3. Build at least one parent aggregation path (`clusters/staging` or `infrastructure`).
4. Reconcile in Flux when live behavior is affected.
5. Leave no unresolved placeholders unless intentionally handled by Flux substitution.

## 5) Code style guidelines (YAML-first)

### 5.1 Formatting

- 2-space indentation.
- Stable, scannable key ordering.
- Quote strings when they contain special characters/template syntax.
- Avoid commented-out dead config.

### 5.2 Naming conventions

- `metadata.name`: lowercase kebab-case, short and stable.
- `metadata.namespace`: explicit on namespaced resources.
- Filenames: lowercase kebab-case by resource intent.
  - `kustomization.yaml`
  - `helmrelease.yaml`
  - `helmrepository.yaml`
  - `ingressroute.yaml`

### 5.3 Kustomize conventions

- Every deployable directory should contain `kustomization.yaml`.
- Keep `resources:` paths relative, explicit, and logically ordered.
- Prefer composition via small modules aggregated by parent kustomizations.

### 5.4 Flux conventions

- Flux `Kustomization` objects live under `clusters/*`.
- Use `dependsOn` for ordering (infrastructure before apps).
- Keep `sourceRef` consistent (`GitRepository/flux-system`) unless intentionally different.
- Use `postBuild.substituteFrom` for runtime values (e.g., `${STAGING_LB_IP}`).

### 5.5 Imports, typing, and scripts

- Imports: N/A in this YAML-only repo.
- Static typing: N/A; correctness comes from valid API schemas/fields.
- If scripts are added later, require strict shell safety (`set -euo pipefail`).

### 5.6 Error handling and safety

- Never commit secrets, kubeconfigs, credentials, or tokens.
- Prefer additive, reversible manifest changes.
- Avoid destructive behavior without explicit operator intent.
- Validate uncertain CRD fields against installed API versions.

## 6) Repository-specific patterns

- Staging apps path: `./apps/devenv/overlays/staging`.
- Staging apps depend on staging infrastructure.
- Staging IngressRoute host uses `${STAGING_LB_IP}` substitution.
- Substitution source: `ConfigMap/staging-vars` in `flux-system`.

## 7) Cursor / Copilot rules status

Checked paths and currently absent:

- `.cursor/rules/`
- `.cursorrules`
- `.github/copilot-instructions.md`

Until these files exist, treat this `AGENTS.md` as authoritative.

## 8) Git and PR hygiene

- Keep commits small and deployable.
- Use concise, plain-English commit messages.
- Avoid mixing unrelated changes.
- Validate all affected kustomization roots before PR.

Pre-PR checks:

```bash
git status
kustomize build clusters/staging
kustomize build infrastructure
kustomize build apps/devenv/overlays/staging
```

## 9) Quick agent decision rules

- Ordering change? Verify `dependsOn` and cluster root kustomization.
- Endpoint/IP change? Prefer Flux substitution over hardcoding.
- Editing generated `gotk-*` files? Avoid unless explicitly requested.
- Missing local binaries? Report exact failing command and required install.

End of file.
