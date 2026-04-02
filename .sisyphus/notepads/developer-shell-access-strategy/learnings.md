# Learnings

- Plan is recommendation-first: Tailscale-first for prod; Cloudflare Access + Tunnel + OIDC as staging alternative.
- `auth.sneakysquid.xyz` exists and is the intended OIDC provider for Cloudflare.
- Verification must be command-based and agent-executable; evidence stored under `.sisyphus/evidence/`.
- Current `devenv` baseline is browser-first: `ttyd` wraps `tmux new -A -s main` on port 7681.
- Staging currently exposes `ttyd` via `Service type: LoadBalancer`; prod exposes `ttyd` via Traefik `IngressRoute`.
- Flux wiring already exists for staging apps and prod apps; the app boundary is reusable, but the browser-facing access layer conflicts with the target.
- Terminal-only means the shell runtime must own `tmux` persistence and `opencode`, while access stays external.
- For this wave, the safest repo decision is reuse-first now and replacement-later for the access overlay.
- Render checks remain documentation-only in this task; no manifests were changed.
- Audit note: `devenv` is reusable for shell access, but blockers remain: `ttyd` hardcodes port `7681`, invokes `tmux new -A -s main`, staging exposes it via `Service type: LoadBalancer`, and prod exposes it via Traefik `IngressRoute`.
- Repo-safe boundary for this wave: no existing manifests in `apps/` or `clusters/` were edited; verification will create evidence files under `.sisyphus/evidence/` only.
- New repo-owned overlay path chosen: `apps/devenv/overlays/ssh-native` with local namespace, deployment, and service files.
- Render evidence captured successfully for `apps/devenv/overlays/ssh-native` and `clusters/prod`; `clusters/staging` render remains large but is preserved in evidence.
- Traefik already exposes `websecure`, so the SSH-native overlay can use `IngressRouteTCP` without a new infra entrypoint for now.
