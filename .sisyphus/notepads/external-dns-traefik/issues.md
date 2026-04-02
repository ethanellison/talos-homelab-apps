plan: external-dns-traefik
created: 2026-03-24T13:07:05.371Z
summary: |
  Metis/Momus review identified a set of gaps and mitigations; capture concise critical findings here.

---

# Issues (append-only)

## 2026-03-24T13:07:05.371Z
- Environment separation must be explicit to avoid DNS ownership collisions (use unique txt-owner-id per env).
- DNS ownership collisions risk if zone filters are too broad; require strict filters per environment.
- TLS and DNS responsibility separation needed: cert-manager planned but must not be folded into DNS automation.
- external-dns missing from repo: must add module and overlays before renders/reconcile.

appended-by-session: ses_2e0266693ffepKMOaaTVMp9YDK at 2026-03-24T13:07:05.371Z
