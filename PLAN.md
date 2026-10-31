# Homelab Plan

Kubernetes homelab managed with Flux. This repo is the source of truth.

## Build order

1. ✅ **Cluster:** k3s in Lima VMs on the 3 minis (servers) plus the Pi (agent), over Tailscale.
   [nodes](docs/nodes.md), [networking](docs/networking.md), [bootstrap](docs/bootstrap.md)
2. ✅ **Flux:** bootstrapped against this repo, split into `infrastructure/` and `apps/`
   ([gitops](docs/gitops.md)). podinfo as smoke test.
3. **Core infra:**
   - ✅ Traefik with Gateway API ([networking](docs/networking.md))
   - ✅ cert-manager: wildcard Let's Encrypt cert for `*.home.jakehulberg.com` via Cloudflare DNS,
     served by the Gateway ([certificates](docs/certificates.md))
   - ✅ Storage: local-path volumes, CloudNativePG Postgres, continuous backups to R2
     ([storage](docs/storage.md))
   - ✅ SOPS for bootstrap secrets in git ([secrets](docs/secrets.md))
4. ✅ **Infisical:** stock instance at `https://infisical.home.jakehulberg.com`, database backed up to
   R2 ([infisical](docs/infisical.md)).
   - Its own secrets come from Infisical Cloud and CloudNativePG, never from itself.
   - The demo instance (dummy data for videos) stays in Docker on mini3, outside this repo.
5. ✅ **Infisical Operator:** syncs every secret from Infisical Cloud (`homelab` project, machine
   identity `homelab-cluster`, Universal Auth). SOPS holds only the operator's own credentials.
   [secrets](docs/secrets.md)
6. ✅ **Authentik:** OIDC sign-in for the cluster's own apps at `https://auth.home.jakehulberg.com`
   ([authentik](docs/authentik.md)). Infisical keeps its own login.
7. **Apps:** below.

## To do

- **life-dashboard restore drill:** Litestream streams to R2 (SSE-C), restore untested. Optional:
  nightly age-encrypted snapshot CronJob (Litestream 0.5 has no age support).
- **life-dashboard on Postgres:** move from SQLite to CloudNativePG for replicas and shared backups.
  ~205 sync `better-sqlite3` calls across 25 files go async, plus SQL dialect fixes and a data
  migration.
- **Monitoring:** kube-prometheus-stack (Prometheus, Grafana) plus Loki. Needs 1 to 2GB RAM.
- **Power-loss test:** reboot a mini without logging in; confirm the node returns `Ready` (Lima
  autostart may need a logged-in user).
- **Restore drill:** restore `infisical-db` from R2 into a new Cluster and point a test Infisical at
  it ([storage](docs/storage.md#restore-not-yet-tested)).
- **Ethernet for the minis:** Wi-Fi latency between nodes is 60 to 280ms with spikes, even on
  direct paths (vzNAT). Wiring cuts that and allows bridged networking.
- **Session minting outside the sandbox pod:** today the `session` sidecar logs in with Kubernetes
  Auth and opens the Agent Vault session inside the pod. Under gVisor both containers share one
  kernel, so an escape from `agent` into the sidecar reaches its Infisical access token. Have the
  orchestrator (or a broker) open the session per claim and pass only the session in; then drop
  the sandbox's egress to Infisical. Airlock code change.
- **Renovate PR reviewer:** a daily agent run that reviews open Renovate PRs across my repos. It
  reads each diff, the release notes and CI status, checks how the repo uses the dependency, then
  approves, comments with needed changes, or flags majors for me. Start comment-only; later allow
  merging patch updates with green CI. Needed:
  - **GitHub through Agent Vault:** fine-grained token (contents read, pull requests read/write) on
    the repos, `api.github.com` and `github.com` in a bundle. `gh` in the agent image with a
    placeholder `GH_TOKEN`.
  - **Pool:** a `reviewer` SandboxTemplate and pool with its own access bundle (GitHub, Anthropic).
  - **Instructions:** prompt or `CLAUDE.md`: what counts as safe, what never to merge.
  - **Trigger:** a `CronJob` in `airlock` that claims a sandbox through the orchestrator API, sends
    the prompt, waits, and releases. Needs its own orchestrator token and a NetworkPolicy rule on
    the orchestrator. In-cluster so the orchestrator stays private; no webhook, since GitHub can't
    reach the cluster.

## Apps

- draw.io
- ✅ **life-dashboard** at `life.home.jakehulberg.com`: private ghcr.io image, Flux image
  automation, SQLite on a local-path volume with Litestream to R2 ([storage](docs/storage.md)).
  - **To check:** `.dockerignore` excludes `data/`, `statements/`, `mortgage/`; the system-stats
    widget reads host CPU/disk/network (`en1`) and will show container stats instead; mortgage seed
    data (account number) is hardcoded in `lib/finance-db.ts`; confirm `statements/`, `mortgage/`,
    `cars/` aren't read at runtime.
- Plant watering app
- KubeCon demo
- **Agent orchestration:** run my own agents on the cluster ([airlock](docs/airlock.md)).
  - ✅ Agent Sandbox `v1.0.8` for isolated agent pods; Airlock claims one per session.
  - ✅ No ServiceAccount token and resource limits in agent pods.
  - ✅ gVisor runtime for sandbox pods, on the minis only ([airlock](docs/airlock.md#gvisor)).
  - ✅ Claude Code image ([agent-images](https://github.com/jakehulberg/agent-images)) and
    Infisical Agent Vault through a session sidecar, so no credentials sit in the agent container.
  - Cilium for egress by domain name: deferred. It replaces flannel, so the cluster network is
    reinstalled, and needs care over Tailscale. Prototype in kind first.

## Candidates

- **Code:** Gitea or Forgejo
- **Dashboards:** Homepage, Uptime Kuma
- **Backups:** Velero
- **AI:** Ollama, Open WebUI
