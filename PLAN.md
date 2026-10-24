# Homelab Plan

Kubernetes homelab managed with GitOps (Flux). This repo is the source of truth.

## Build order

1. **Cluster:** k3s in Lima VMs on the 3 minis (servers) plus the Pi (agent), networked over
   Tailscale. See [nodes](docs/nodes.md), [networking](docs/networking.md), [bootstrap](docs/bootstrap.md).
2. **Flux:** `flux bootstrap github` pointed at this repo.
3. **Core infra:**
   - Traefik (ingress: routes hostnames to services, handles HTTPS)
   - cert-manager (TLS certificates)
   - Storage: Longhorn or local-path
   - SOPS (encrypts bootstrap secrets in git)
4. **Infisical:** migrate from mini3.
   - `pg_dump` the existing database, restore into the cluster.
   - Bootstrap secrets (Postgres, Redis, encryption key) come from SOPS, not Infisical.
   - Cut DNS over once verified. Keep mini3 running until then.
5. **Infisical Operator:** all apps from here on pull secrets from Infisical.
6. **Authentik:** SSO for everything, including Infisical's login.
7. **Apps:** see below.

## Apps

- draw.io
- life-dashboard

## Candidates

- **Monitoring:** Grafana, Prometheus, Loki
- **Code:** Gitea or Forgejo, Renovate
- **Dashboards:** Homepage, Uptime Kuma
- **Backups:** Velero
- **AI:** Ollama, Open WebUI, agents as Deployments/CronJobs, Agent Vault for agent credentials
