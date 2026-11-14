# Homelab plan

A Kubernetes homelab managed with Flux. This repo is the source of truth for the cluster.

## Build order

1. ✅ **Cluster:** k3s in Lima VMs on the 3 minis (servers) plus the Pi (agent), over Tailscale
   ([nodes](docs/nodes.md), [networking](docs/networking.md), [bootstrap](docs/bootstrap.md))
2. ✅ **Flux:** bootstrapped against this repo, split into `infrastructure/` and `apps/`
   ([gitops](docs/gitops.md)), with podinfo as a smoke test
3. **Core infra:**
   - ✅ Traefik with Gateway API ([networking](docs/networking.md))
   - ✅ cert-manager: wildcard Let's Encrypt cert for `*.home.jakehulberg.com` via Cloudflare DNS,
     served by the Gateway ([certificates](docs/certificates.md))
   - ✅ Storage: local-path volumes, CloudNativePG Postgres, continuous backups to R2
     ([storage](docs/storage.md))
   - ✅ SOPS for bootstrap secrets in git ([secrets](docs/secrets.md))
4. ✅ **Infisical:** stock instance at `https://infisical.home.jakehulberg.com`, with its
   database backed up to R2 ([infisical](docs/infisical.md))
   - Its own secrets come from Infisical Cloud and CloudNativePG, never from itself
   - The demo instance (dummy data for videos) stays in Docker on mini3, outside this repo
5. ✅ **Infisical Operator:** syncs every secret from Infisical Cloud (`homelab` project, machine
   identity `homelab-cluster`, Universal Auth). SOPS holds only the operator's own credentials
   ([secrets](docs/secrets.md))
6. ✅ **Authentik:** OIDC sign-in for the cluster's own apps at `https://auth.home.jakehulberg.com`
   ([authentik](docs/authentik.md)). Infisical keeps its own login
7. **Apps:** see [Apps](#apps)

## To do

- **life-dashboard restore drill:** Litestream streams the database to R2 (SSE-C), but a restore
  is untested. Optional: a nightly age-encrypted snapshot CronJob (Litestream 0.5 has no age
  support)
- **life-dashboard on Postgres:** move from SQLite to CloudNativePG for replicas and shared backups.
  ~205 sync `better-sqlite3` calls across 25 files become async, plus SQL dialect fixes and a
  data migration
- **Custom alerts:** kube-prometheus-stack runs at `grafana.home.jakehulberg.com`
  (`apps/monitoring/`), and Alertmanager pushes warning and critical alerts to ntfy. The built-in
  rules cover disks, crash loops and nodes, not backups, certificates or Flux. Fill in
  `apps/monitoring/alerts.yaml` (empty, and not yet in its `kustomization.yaml`) with
  `PrometheusRule`s, one at a time. The CRDs are in `infrastructure/`, so a rebuild can apply the
  rules:
  - **Certificates:** `prometheus.servicemonitor.enabled: true` in the cert-manager values, then
    `CertificateExpiringSoon` (under 14 days, from
    `certmanager_certificate_expiration_timestamp_seconds`) and `CertificateNotReady`
  - **Postgres backups:** CNPG metrics through a `PodMonitor`, then `PostgresBackupStale` (no
    backup in 26h) and `PostgresWALArchivingFailing`, for `infisical-db` and `authentik-db`
  - **Flux:** a `PodMonitor` for the controllers (port `http-prom`) plus kube-state-metrics custom
    resource config for `gotk_resource_info`, then `FluxReconcileFailing` (not Ready for 15m)
- **Logs:** Loki, queried from Grafana
- **Power-loss test:** reboot a mini without logging in, and confirm the node returns to `Ready`
  (Lima autostart may need a logged-in user)
- **Restore drill:** restore `infisical-db` from R2 into a new Cluster and point a test Infisical
  at it ([storage](docs/storage.md#restore-not-yet-tested))
- **Session minting outside the sandbox pod:** today the `session` sidecar logs in with Kubernetes
  Auth and opens the Agent Vault session inside the pod. Under gVisor both containers share one
  kernel, so an escape from `agent` into the sidecar reaches its Infisical access token. Have the
  orchestrator (or a broker) open the session per claim and pass only the session in, then drop
  the sandbox's egress to Infisical. This needs a change in Airlock's code
- **kagent through Agent Vault:** the POC reads `ANTHROPIC_API_KEY` from Infisical Cloud into a
  Secret, so the key sits in the agent pod's environment. The Agent CRD allows `env`, `volumes` and
  `extraContainers`, so the session sidecar fits, and the Go runtime honors `HTTPS_PROXY` and
  `SSL_CERT_FILE`. Blocker: the sidecar writes the proxy settings to `/session/env` at runtime, and
  kagent's container reads its environment only at startup. The fix needs a wrapper entrypoint
  or a fixed proxy address, and the ModelConfig key becomes a placeholder
- **Renovate PR reviewer:** a daily agent run that reviews open Renovate PRs across Jake's repos.
  For each PR, the agent reads the diff, the release notes and CI status, checks how the repo uses
  the dependency, then approves, comments with needed changes, or flags major versions for Jake.
  Start comment-only; later allow merging patch updates with green CI. Needed:
  - **GitHub through Agent Vault:** fine-grained token (contents read, pull requests read/write) on
    the repos, `api.github.com` and `github.com` in a bundle. `gh` in the agent image with a
    placeholder `GH_TOKEN`
  - **Pool:** a `reviewer` SandboxTemplate and pool with its own access bundle (GitHub, Anthropic)
  - **Instructions:** a prompt or `CLAUDE.md` that says what counts as safe and what never to merge
  - **Trigger:** a `CronJob` in `airlock` that claims a sandbox through the orchestrator API,
    sends the prompt, waits for the result, and releases the claim. The CronJob needs its own
    orchestrator token and a NetworkPolicy rule on the orchestrator. It runs in-cluster so the
    orchestrator stays private, and there's no webhook, because GitHub can't reach the cluster

## Apps

- draw.io
- ✅ **life-dashboard** at `life.home.jakehulberg.com`: private ghcr.io image, Flux image
  automation, SQLite on a local-path volume with Litestream to R2 ([storage](docs/storage.md))
  - **To check:** `.dockerignore` excludes `data/`, `statements/`, `mortgage/`; the system-stats
    widget reads host CPU/disk/network (`en1`) and will show container stats instead; mortgage seed
    data (account number) is hardcoded in `lib/finance-db.ts`; confirm `statements/`, `mortgage/`,
    `cars/` aren't read at runtime
- Plant watering app
- KubeCon demo
- **Agent orchestration:** run Jake's own agents on the cluster ([airlock](docs/airlock.md))
  - ✅ Agent Sandbox `v1.0.8` for isolated agent pods; Airlock claims one per session
  - ✅ No ServiceAccount token, and resource limits, in agent pods
  - ✅ gVisor runtime for sandbox pods, on the minis only ([airlock](docs/airlock.md#gvisor))
  - ✅ Claude Code image ([agent-images](https://github.com/jakehulberg/agent-images)) and
    [Agent Vault](docs/agent-vault.md) through a session sidecar, so the agent container holds no
    credentials
  - Cilium for egress by domain name: deferred. Cilium replaces flannel, so the cluster network
    has to be reinstalled, and it needs care over Tailscale. Prototype it in kind first

## Experiments

Things Jake is trying out, not recommending. They live in
`infrastructure/controllers/experiments/` and `apps/experiments/`, labeled
`homelab.jakehulberg.com/experiment: "true"` ([apps/experiments](apps/experiments/README.md)).

- ✅ **kagent** `0.10.4`: `k8s-agent` and `promql-agent`, read-only tools, bundled Postgres, UI at
  `kagent.home.jakehulberg.com` with no login (tailnet only)
- **agentgateway** `v1.7.0`: an in-cluster Gateway that adds the Anthropic key from
  `/agentgateway`. Next, point kagent's ModelConfig at the Gateway (`anthropic.baseUrl`, no key)
  and drop `/kagent` from Infisical
- kagent `1.0` and `SandboxAgent`: could it replace Airlock?

## Candidates

- **Code:** Gitea or Forgejo
- **Dashboards:** Homepage, Uptime Kuma
- **Backups:** Velero
- **AI:** Ollama, Open WebUI
