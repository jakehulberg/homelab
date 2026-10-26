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

- **Power-loss test:** reboot one mini without logging in, and confirm its node comes back `Ready`
  (Lima autostart may need the VM's user logged in).
- **Restore drill:** restore `infisical-db` from R2 into a new Cluster and point a test Infisical at
  it, to prove backups actually work ([storage](docs/storage.md#restore-not-yet-tested)).
- **Ethernet for the minis:** direct paths now (vzNAT), but Wi-Fi latency between nodes is
  60 to 280ms with spikes. Wiring them would cut that and allow bridged networking.

## Apps

- draw.io
- life-dashboard
- Plant watering app
- KubeCon demo
- Agents: run my own agents on the cluster, to learn and to dogfood the idea

## Candidates

- **Monitoring:** Grafana, Prometheus, Loki
- **Code:** Gitea or Forgejo, Renovate
- **Dashboards:** Homepage, Uptime Kuma
- **Backups:** Velero
- **AI:** Ollama, Open WebUI, agents as Deployments/CronJobs, Agent Vault for agent credentials
