# Homelab Plan

Kubernetes homelab managed with GitOps (Flux). This repo is the source of truth.

## Build order

1. ✅ **Cluster:** k3s in Lima VMs on the 3 minis (servers) plus the Pi (agent), networked over
   Tailscale. See [nodes](docs/nodes.md), [networking](docs/networking.md), [bootstrap](docs/bootstrap.md).
2. ✅ **Flux:** `flux bootstrap github` pointed at this repo. Repo split into `infrastructure/` and
   `apps/` ([gitops](docs/gitops.md)); podinfo deployed as a smoke test.
3. **Core infra:**
   - ✅ Traefik with Gateway API (routes hostnames to services; HTTPS once cert-manager is in),
     [networking](docs/networking.md)
   - ✅ cert-manager: wildcard Let's Encrypt certificate for `*.home.jakehulberg.com` via
     Cloudflare DNS, [certificates](docs/certificates.md). Gateway serves HTTPS with it.
   - ✅ Storage: local-path volumes plus CloudNativePG for Postgres, backed up continuously to
     Cloudflare R2, [storage](docs/storage.md).
   - ✅ SOPS (encrypts bootstrap secrets in git, [secrets](docs/secrets.md))
4. ✅ **Infisical:** a fresh, stock instance at `https://infisical.home.jakehulberg.com`,
   [infisical](docs/infisical.md). Database backed up to R2.
   - Bootstrap secrets (Postgres, Redis, encryption key) come from SOPS, not Infisical.
   - The demo instance (dummy data for videos) stays in Docker on mini3 and is not managed by
     this repo.
5. ✅ **Infisical Operator:** syncs every secret from Infisical Cloud (dedicated `homelab` project,
   machine identity `homelab-cluster`, Universal Auth). SOPS now holds only the operator's own
   credentials. No secret sprawl. [secrets](docs/secrets.md)
6. **Authentik:** SSO for everything, including Infisical's login.
7. **Apps:** see below.

## To do

- **life-dashboard backups (do first):** its SQLite data lives only in one volume, with no off-site
  copy; mini1's `~/Documents/life-dashboard/data/` is a frozen copy from the 2026-10-26 migration.
  Plan: Litestream sidecar streaming `finance.db` and `tasks.db` to R2, encrypted with SSE-C (a key
  of ours in Infisical, never stored by Cloudflare). Litestream 0.5 doesn't support age; optional
  extra: nightly age-encrypted snapshot via a CronJob. Use a key separate from the SOPS key.
- **life-dashboard on Postgres:** port from SQLite to a CloudNativePG database, for replicas and the
  same backups as Infisical. About 205 synchronous `better-sqlite3` calls across 25 files become
  async, plus SQL dialect fixes and a data migration.
- **Power-loss test:** reboot one mini without logging in, and confirm its node comes back `Ready`
  (Lima autostart may need the VM's user logged in).
- **Restore drill:** restore `infisical-db` from R2 into a new Cluster and point a test Infisical at
  it, to prove backups actually work ([storage](docs/storage.md#restore-not-yet-tested)).
- **Ethernet for the minis:** direct paths now (vzNAT), but Wi-Fi latency between nodes is
  60 to 280ms with spikes. Wiring them would cut that and allow bridged networking.

## Apps

- draw.io
- **life-dashboard** (next). Next.js app, private repo `jakehulberg/life-dashboard`, today a dev
  server in tmux on mini1 (`~/Documents/life-dashboard`).
  - **Pipeline:** Dockerfile with a production build; a GitHub Action builds on every push to `main`
    and pushes a private image to `ghcr.io/jakehulberg/life-dashboard`; Flux image automation
    detects new tags and commits the bump here, so every deploy is a revertible commit.
  - **Storage:** one PersistentVolume (local-path, ~2GB) mounted as `data/` (`finance.db`,
    `tasks.db`, uploaded investment statements). One replica, `Recreate` strategy (SQLite has one
    writer). Litestream sidecar streams the databases to R2, age-encrypted.
  - **Secrets:** `/life-dashboard` folder in Infisical Cloud (weather coordinates, YouTube, GitHub
    PAT, Plaid), synced by the operator. ghcr.io pull token also via Infisical.
  - **Route:** `life.home.jakehulberg.com`.
  - **To check:** `.dockerignore` keeps `data/`, `statements/`, `mortgage/` out of the image; the
    system-stats widget reads host CPU/disk/network (`en1`), which becomes container stats; mortgage
    seed data (account number) is hardcoded in `lib/finance-db.ts`; confirm `statements/`,
    `mortgage/`, `cars/` aren't read at runtime.
- Plant watering app
- KubeCon demo
- **Agent orchestration** (after life-dashboard): run my own agents on the cluster, to learn and
  to dogfood the idea.
  - Cilium for networking and policy (replaces k3s's flannel, so it means reinstalling the cluster
    network).
  - Agent Sandbox for isolated agent pods.
  - No ServiceAccount token mounted in agent pods.
  - Infisical Agent Vault as the agents' egress proxy, so credentials never sit in the pod.
  - Resource limits on every agent pod.

## Candidates

- **Monitoring:** Grafana, Prometheus, Loki
- **Code:** Gitea or Forgejo, Renovate
- **Dashboards:** Homepage, Uptime Kuma
- **Backups:** Velero
- **AI:** Ollama, Open WebUI, agents as Deployments/CronJobs, Agent Vault for agent credentials
