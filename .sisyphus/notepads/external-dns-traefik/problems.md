plan: external-dns-traefik
created: 2026-03-24T13:07:05.371Z
summary: |
  Blockers preventing immediate implementation are recorded here for tracking and triage.

---

# Problems (append-only)

## 2026-03-24T13:07:05.371Z
- Missing `external-dns` module in repository — must be added before rendering or reconcile tasks.
- Provider credentials not yet provided for Cloudflare (staging) and Pi-hole (prod) — must confirm delivery mechanism into Flux and ensure secrets are not committed.
- Need confirmation on Pi-hole external-dns mechanism (provider/adapter) and whether Pi-hole supports TXT ownership mode used by external-dns.

appended-by-session: ses_2e0266693ffepKMOaaTVMp9YDK at 2026-03-24T13:07:05.371Z
