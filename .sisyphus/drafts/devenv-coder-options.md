# Draft: Coder Workspace Access Options

## Requirements (confirmed)
- Consider deploying **Coder workspaces** for developer environments.
- Access should be possible behind **Tailscale VPN** and/or **Cloudflare**.
- Existing access preference remains terminal-oriented and single-developer focused.
- Cloudflare path should continue to use `auth.sneakysquid.xyz` as OIDC if included.
- Multiple repos should be supported as separate environments with repo-specific `.devcontainer` configs.

## Technical Decisions
- Coder is now a candidate for the shell/workspace runtime, not just a raw shell pod.
- Coder should be treated as **one option among others**.
- Coder should be evaluated with **both Tailscale and Cloudflare** access models.
- Primary deployment path remains **raw shell + tmux behind Tailscale**.
- Coder is not currently the deployment target; it remains a comparison option.
- Provisioning should be on-demand per repo/environment, not always-on for every repo.
- The provisioning lifecycle should be **backend-agnostic**: Coder may implement it later, but does not own it now.
- Use **skills as the primary repeated agent guidance** in the plan.
- Keep **explicit commands** in acceptance criteria and QA scenarios so verification remains executable.
- The plan should read like a **spec**: desired state, constraints, interfaces, and measurable outcomes — not just a task list.

## Spec-Driven Fit (assessment)
- **Mostly yes**: it already has objectives, guardrails, and command-based verification.
- **Not fully yet**: some sections still read like implementation notes instead of stable requirements.
- For agentic workloads, each task should explicitly state:
  - the capability being specified
  - inputs/outputs and boundaries
  - non-goals
  - acceptance criteria
  - exact verification commands
  - agent skill/profile guidance

## Research Findings
- Current `devenv` is browser-first with `ttyd` + `tmux`.
- Prior direction favored terminal-only access and private access gates.
- Momus flagged one blocking issue: a referenced `apps/devenv/overlays/prod/service.yaml` path did not exist and needed correction.
- Momus recommended splitting the design into three layers: shell runtime, access gate, and workspace orchestration.
- `auth.sneakysquid.xyz` exists, so Cloudflare OIDC integration can be treated as available.

## Open Questions
- Should Coder be the **primary recommended architecture** or just a **competing option**?
- Should the plan compare:
  - Coder + Tailscale
  - Coder + Cloudflare OIDC
  - or Coder as the workspace layer with Tailscale/Cloudflare as interchangeable access gates?

## Scope Boundaries
- INCLUDE: Coder workspace model, access-gate options, `tmux`/session persistence, `opencode` support.
- EXCLUDE: browser terminal/ttyd fallback unless explicitly re-added.

## Recommendation Matrix (working)

| Option | Best For | Pros | Cons | Recommendation |
|---|---|---|---|---|
| Raw shell + tmux | Minimal, direct SSH workflow | Simple, low overhead, easy to reason about | You own all access/auth hardening | Good baseline |
| Coder workspaces | Browser-based dev env with isolation | Workspace persistence, easier onboarding, strong UX | More platform weight, less terminal-native | Considered option |
| Tailscale access | Private prod access | Strongest fit for terminal-only, simple trust model | Requires Tailscale client/device auth | **Recommended for prod** |
| Cloudflare + OIDC | Edge-access / staging convenience | Public zero-trust, good identity controls via `auth.sneakysquid.xyz` | More moving parts, less direct than Tailscale | **Recommended for staging / alt** |

## Provisioning Flow (primary option)
- Flux applies a dedicated `devenv` workload in Kubernetes.
- The workload runs a prebuilt dev image with shell tools, `tmux`, and `opencode` available.
- A persistent volume backs the home/workspace directory so files survive pod restarts.
- Tailscale provides private network access to the workload.
- Users connect with SSH over the Tailscale network, then attach to `tmux`.
- No browser terminal is deployed in the primary path.

## Provisioning Lifecycle
- **Create**: spin up a repo-specific environment only when needed.
- **Start**: attach the workspace volume and launch the shell runtime.
- **Stop**: pause the environment while preserving state.
- **Resume**: restart and reattach to the same session/state.
- **Destroy**: remove the workspace and its resources when no longer needed.
