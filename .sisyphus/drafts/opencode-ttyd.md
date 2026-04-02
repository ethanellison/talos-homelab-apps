# Draft: DevPod for ttyd

## Requirements (confirmed)
- Add DevPod support in the ttyd image.
- The Kubernetes provider must be included.
- DevPod should be usable to provision devcontainer pods from ttyd.
- ttyd should authenticate using a dedicated ServiceAccount, not a baked kubeconfig.

## Technical Decisions
- Package DevPod into the ttyd runtime and ensure the Kubernetes provider is available.
- Provide in-cluster ServiceAccount auth and RBAC so DevPod can create/exec into workspace pods.

## Open Questions
- Should access be read-only/tooling-only, or allowed to modify the repo from within ttyd?

## Scope Boundaries
- INCLUDE: repo changes needed to make opencode available from ttyd.
- EXCLUDE: unrelated app changes.
