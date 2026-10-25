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
   - Storage: Longhorn or local-path
   - ✅ SOPS (encrypts bootstrap secrets in git, [secrets](docs/secrets.md))
4. **Infisical:** a fresh, stock instance in the cluster.
   - Bootstrap secrets (Postgres, Redis, encryption key) come from SOPS, not Infisical.
   - The demo instance (dummy data for videos) stays in Docker on mini3 and is not managed by
     this repo.
5. **Infisical Operator:** all apps from here on pull secrets from Infisical.
   - Point it at Infisical Cloud (a machine identity with Universal Auth, available on the free
     plan) so Cloud is the single source of truth. That removes the duplicate SOPS copies of
     secrets that also live in Cloud (Cloudflare token, Infisical's own keys), leaving SOPS with
     only the operator's identity credentials. Goal: no secret sprawl.
6. **Authentik:** SSO for everything, including Infisical's login.
7. **Apps:** see below.

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
