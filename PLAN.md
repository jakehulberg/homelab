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
- **Airlock with Infisical Agent Vault:** agents call APIs through Agent Vault, which injects
  credentials at the proxy, so none sit in the sandbox. Today Airlock only sources `/session/env`
  and has no Infisical support. Needed:
  - **Who mints session tokens:** the orchestrator, with its own Infisical machine identity, minting
    a scoped, short-lived Agent Vault session per claim and handing it to the sandbox; or a sidecar
    in each sandbox pod. Either way the minting identity must stay out of the `agent` container.
    Airlock code change.
  - **Agent image:** an `agent` container with the `claude` CLI, replacing busybox in the
    `SandboxTemplate`.
  - **Wiring:** `/session/env` with the proxy settings (`HTTPS_PROXY`, its CA) before the first task.
    Claim-time env forces a cold start instead of a warm sandbox, so write the file after the claim.
  - **Egress:** a `networkPolicy` on the template allowing only Agent Vault (and DNS), replacing the
    default internet-only policy.
  - **Secrets:** the identity's credentials in `/airlock` (or the sandbox's folder), synced by the
    operator.

## Apps

- draw.io
- **life-dashboard** (next). Next.js, private repo `jakehulberg/life-dashboard`, currently a dev
  server in tmux on mini1 (`~/Documents/life-dashboard`).
  - **Pipeline:** production Dockerfile; GitHub Action on push to `main` pushes a private image to
    `ghcr.io/jakehulberg/life-dashboard`; Flux image automation commits the tag bump here, so every
    deploy is a revertible commit.
  - **Storage:** one local-path PV (~2GB) at `data/` (`finance.db`, `tasks.db`, uploaded
    statements). One replica, `Recreate` (SQLite has one writer). Litestream sidecar to R2.
  - **Secrets:** `/life-dashboard` in Infisical Cloud (weather coordinates, YouTube, GitHub PAT,
    Plaid) and the ghcr.io pull token, synced by the operator.
  - **Route:** `life.home.jakehulberg.com`.
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
  - Agent image and Infisical Agent Vault (to do, below), so credentials never sit in the pod.
  - Cilium for egress by domain name: deferred. It replaces flannel, so the cluster network is
    reinstalled, and needs care over Tailscale. Prototype in kind first.

## Candidates

- **Code:** Gitea or Forgejo, Renovate
- **Dashboards:** Homepage, Uptime Kuma
- **Backups:** Velero
- **AI:** Ollama, Open WebUI, agents as Deployments/CronJobs, Agent Vault for agent credentials
