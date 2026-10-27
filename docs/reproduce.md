# Reproduce this homelab

What to change and what order to build in. The other docs explain each part.

## What you get

k3s across a few machines on a tailnet, managed by Flux from git:

- Traefik with Gateway API, HTTPS on your domain (wildcard Let's Encrypt cert)
- Secrets in Infisical Cloud, synced by the Infisical Operator
- Postgres via CloudNativePG, backed up continuously to Cloudflare R2
- Self-hosted Infisical as the first app; Renovate PRs for updates

Nothing is exposed to the internet; apps are tailnet-only.

## Prerequisites

| Need | Notes |
|---|---|
| 3+ machines | Here: three Mac minis (Lima VMs) and a Pi. Linux machines can skip Lima. Three servers for HA; one works for testing. |
| Tailscale | All nodes and your laptop on one tailnet. |
| Domain on Cloudflare DNS | For the DNS-01 wildcard cert and `*.home.<domain>` records. |
| Cloudflare R2 | One bucket for backups. Free tier is enough. |
| Infisical Cloud | One project. Free plan works (Universal Auth). |
| GitHub | Repo Flux syncs from. |
| Laptop tools | `kubectl`, `flux`, `helm`, `sops`, `age`, `infisical`, `gh` |

## 1. Take the repo

Fork or copy it, then remove what's specific to this cluster:

```sh
rm -rf apps/life-dashboard clusters/homelab/flux-system .infisical.json
rm infrastructure/configs/infisical-universal-auth.sops.yaml
```

Remove `- life-dashboard` from `apps/kustomization.yaml` and the
`infisical-universal-auth.sops.yaml` line from `infrastructure/configs/kustomization.yaml` (re-added
in step 5). Keep `podinfo` for testing.

## 2. Replace personal values

| Value here | Yours | Files |
|---|---|---|
| `jakehulberg.com` | your domain | `infrastructure/configs/certificate.yaml`, `apps/*/httproute.yaml` |
| `e8dc2af5d31bf4e016b6fab1e2515a27.r2.cloudflarestorage.com` | your R2 S3 endpoint | `apps/infisical/objectstore.yaml` |
| `homelab-postgres-backups` | your bucket | `apps/infisical/objectstore.yaml` |
| `projectSlug: homelab` | your Infisical project slug | every `*-sync.yaml` |
| `https://app.infisical.com` | `https://eu.infisical.com` for EU orgs | `infrastructure/configs/infisical-cloud.yaml` |
| `k3s-mini1/2/3` | your server node names | `k3s/server.yaml` (`tls-san`), `lima/` |
| `age1mn6c...` | your age public key (step 5) | `.sops.yaml` |

Size the VMs in `lima/` for your hardware ([nodes](nodes.md)).

## 3. Cloudflare

1. **DNS:** `A` records `*.home`, one per node Tailscale IP, **DNS only** (grey cloud). IPs come
   from step 4.
2. **API token:** "Edit zone DNS" template, your zone only (cert-manager).
3. **R2:** create the bucket (Standard class) and an Account API token with Object Read & Write on
   it only. Note the key pair and S3 endpoint.

Details: [certificates](certificates.md), [storage](storage.md#r2).

## 4. Nodes and k3s

[Bootstrap](bootstrap.md) steps 1 to 7: token, VMs (with `vzNAT`,
[networking](networking.md#lima-vm-networking-vznat)), Tailscale, k3s config including
`resolv.conf`, k3s, agents, kubeconfig.

## 5. Secrets

**Infisical Cloud**, new project, env `dev`:

| Folder | Secrets |
|---|---|
| `/` | `SOPS_AGE_KEY`, `K3S_CLUSTER_SECRET` |
| `/cert-manager` | `CLOUDFLARE_API_TOKEN` |
| `/r2` | `R2_ACCESS_KEY_ID`, `R2_SECRET_ACCESS_KEY` |
| `/infisical` | `ENCRYPTION_KEY`, `AUTH_SECRET`, `REDIS_PASSWORD` ([generate](secrets.md#infisicals-own-keys)), `REDIS_URL` = `redis://:${REDIS_PASSWORD}@redis:6379`, `SITE_URL` = `https://infisical.home.<domain>` |

**SOPS key:** `age-keygen`; public key into `.sops.yaml`, private key into Infisical as above
([bootstrap step 9](bootstrap.md#9-sops-key)).

**Machine identity:** Universal Auth, no org access, Viewer on this project only. Encrypt its
credentials into `infrastructure/configs/infisical-universal-auth.sops.yaml`
([secrets](secrets.md#rotate-the-operators-credentials)) and re-add it to that folder's
`kustomization.yaml`.

Commit and push.

## 6. Flux

[Bootstrap step 8](bootstrap.md#8-flux) with your owner and repo, then create `sops-age`
([step 9](bootstrap.md#9-sops-key)). Flux installs the rest in order: `infrastructure` →
`infra-configs` → `apps`.

## 7. Check

```sh
flux get kustomizations                 # all READY True
kubectl get certificate -n traefik      # home-wildcard READY True
kubectl get cluster -n infisical        # Cluster in healthy state
```

Open `https://infisical.home.<domain>` and create the admin account right away (first boot takes
~10 minutes, [infisical](infisical.md#first-boot)).

## Optional

- **Renovate:** install the GitHub App on the repo; `renovate.json` is ready ([updates](updates.md)).
- **Your own apps:** follow life-dashboard's pattern in git history: private ghcr.io image built by
  GitHub Actions, Flux image automation (needs bootstrap `--components-extra` and
  `--read-write-key`), secrets via `InfisicalStaticSecret`.

## Known rough edges

Read [incidents](incidents/). In short: on Wi-Fi, use `vzNAT` (or bridged on ethernet) so Tailscale
connects nodes directly, and give k3s its own `resolv.conf` so pod DNS skips your router's search
domains.
