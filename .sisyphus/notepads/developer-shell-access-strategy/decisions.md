# Decisions

- Execution style: skills-led execution, commands reserved for verification.
- Coder is an evaluated option, not the owner of lifecycle by default.
- Cloudflare OIDC provider `auth.sneakysquid.xyz` confirmed.
- Reuse the existing `devenv` scaffold/namespace/Flux wiring; replace the browser-facing access layer.
- Terminal-only shell is the target model.
- Tailscale SSH is the primary production path.
- Cloudflare Access + Tunnel + native SSH is the alternate/staging path.
- Reuse `devenv` now; replace the current browser-facing overlay later with an SSH-native overlay if needed.
- Omitted skills review: `playwright` is irrelevant (no browser work), `frontend-ui-ux` is irrelevant (no UI design), `git-master` is unnecessary (no git operation requested), and `kustomize-render-guard` is deferred because this wave makes no manifest edits.
- This wave is documentation-only: do not mutate repo manifests; keep access-gate ownership external until a dedicated SSH-native overlay is introduced.
- GitOps-owned access was adopted: use a new `apps/devenv/overlays/ssh-native` overlay instead of mutating ttyd in place.
- The SSH-native overlay should own its namespace locally, keep `Service type: LoadBalancer` initially, and use `sshd -D -e` as the runtime entrypoint.
- Staging and prod Flux app wiring now point at the SSH-native overlay path.
- SSH-native overlay now uses Traefik TCP routing (`IngressRouteTCP`) on the existing `websecure` entrypoint instead of a direct `LoadBalancer` service.
