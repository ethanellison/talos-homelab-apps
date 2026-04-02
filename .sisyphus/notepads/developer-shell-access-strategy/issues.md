# Issues

- Need to confirm exact Cloudflare primitive choice (Access vs Tunnel) before manifest changes.
- No existing evidence folder; tasks must create `.sisyphus/evidence/` files as they run.
- Reuse candidate is blocked by exposure mismatch: staging uses LoadBalancer while prod uses Traefik ingress, so any replacement must preserve two wiring paths or standardize them first.
- Future follow-up: decide whether prod should keep the same `LoadBalancer` model or switch to a TCP route after this initial SSH-native overlay is proven.
- Evidence note: staging cluster render output is very large; the important validation is that the SSH-native overlay renders cleanly and prod wiring points at the new overlay.
