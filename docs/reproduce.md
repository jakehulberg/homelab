# Reproduce this homelab

How to run this setup on your own machines: what to change in the repo, and the order to build
in. The other docs explain each part.

## What you get

k3s across a few machines on a tailnet, managed by Flux from git:

- Traefik with Gateway API, and HTTPS on your domain from a wildcard Let's Encrypt certificate
- Secrets in Infisical Cloud, synced into the cluster by the Infisical Operator
- Postgres through CloudNativePG, backed up continuously to Cloudflare R2
- Self-hosted Infisical as the first app, and Renovate PRs for updates

Nothing is exposed to the internet: apps are reachable only from the tailnet.

## Prerequisites

| Need | Notes |
|---|---|
| 3+ machines | Here: three Mac minis (Lima VMs) and a Pi. Linux machines can skip Lima. Three servers give HA; one works for testing |
| Tailscale | All nodes and your laptop on one tailnet |
| Domain on Cloudflare DNS | For the DNS-01 wildcard certificate and the `*.home.<domain>` records |
| Cloudflare R2 | One bucket for backups. The free tier is enough |
| Infisical Cloud | One project. The free plan works (Universal Auth) |
| GitHub | The repo Flux syncs from |
| Laptop tools | `kubectl`, `flux`, `helm`, `sops`, `age`, `infisical`, `gh` |

## 1. Take the repo

Fork or copy the repo, then remove what's specific to this cluster:

```sh
rm -rf apps/life-dashboard clusters/homelab/flux-system .infisical.json
rm infrastructure/configs/infisical-universal-auth.sops.yaml
```

Then edit two kustomizations:

- `apps/kustomization.yaml`: remove `- life-dashboard`
- `infrastructure/configs/kustomization.yaml`: remove the `infisical-universal-auth.sops.yaml`
  line (you add it back in [step 5](#5-secrets))

Keep `podinfo` for testing. The other apps each need their own Infisical Cloud folders. Remove
an app's line from `apps/kustomization.yaml` if you don't want it:

| App | Infisical folders (env `dev`) | Also needs |
|---|---|---|
| `authentik` | `/authentik`, `/r2` | [authentik](authentik.md) |
| `agents` | none | gVisor ([bootstrap step 11](bootstrap.md#11-gvisor-on-the-minis)). Uses the `ghcr.io/jakehulberg/agent-claude` image |
| `airlock` | `/airlock`, `/ghcr` | Private `ghcr.io/jakehulberg/airlock-*` images ([airlock](airlock.md)) |
| `agent-vault` | `/agent-vault` | [agent-vault](agent-vault.md) |
| `monitoring` | `/monitoring`, `/monitoring/alertmanager` | |
| `experiments` | `/kagent`, `/agentgateway` | Controllers in `infrastructure/controllers/experiments/` ([experiments](../apps/experiments/README.md)) |

## 2. Replace personal values

| Value here | Yours | Files |
|---|---|---|
| `jakehulberg.com` | your domain | `infrastructure/configs/certificate.yaml`, `apps/*/httproute.yaml`, `apps/airlock/web.yaml` |
| `e8dc2af5d31bf4e016b6fab1e2515a27.r2.cloudflarestorage.com` | your R2 S3 endpoint | `apps/infisical/objectstore.yaml`, `apps/authentik/objectstore.yaml` |
| `homelab-postgres-backups` | your bucket | `apps/infisical/objectstore.yaml`, `apps/authentik/objectstore.yaml` |
| `projectSlug: homelab` | your Infisical project slug | every `*-sync.yaml` |
| `https://app.infisical.com` | `https://eu.infisical.com` for EU orgs | `infrastructure/configs/infisical-cloud.yaml` |
| `k3s-mini1/2/3` | your server node names | `k3s/server.yaml` (`tls-san`), `lima/` |
| `age1mn6c...` | your age public key ([step 5](#5-secrets)) | `.sops.yaml` |

Size the VMs in `lima/` for your hardware ([nodes](nodes.md)).

## 3. Cloudflare

1. **DNS:** add `A` records for `*.home`, one per node Tailscale IP, set to **DNS only** (grey
   cloud). The IPs come from [step 4](#4-nodes-and-k3s)
   ([certificates](certificates.md#dns-records))
2. **API token:** create a token from the "Edit zone DNS" template, scoped to your zone only. It's
   for cert-manager ([certificates](certificates.md#cloudflare-api-token))
3. **R2:** create the bucket (Standard class) and an Account API token with Object Read & Write on
   that bucket only. Note the key pair and the S3 endpoint ([storage](storage.md#r2))

## 4. Nodes and k3s

Follow [bootstrap](bootstrap.md) steps 1 to 7: token, VMs (bridged onto
the LAN, [networking](networking.md#lima-vm-networking-bridged)), Tailscale, k3s config including
`resolv.conf`, k3s servers, the agent, and kubeconfig. If you keep `agents`, also do
[bootstrap step 11](bootstrap.md#11-gvisor-on-the-minis).

## 5. Secrets

**Infisical Cloud:** create a new project with env `dev`, then add these folders and secrets:

| Folder | Secrets |
|---|---|
| `/` | `SOPS_AGE_KEY`, `K3S_CLUSTER_SECRET` |
| `/cert-manager` | `CLOUDFLARE_API_TOKEN` |
| `/r2` | `R2_ACCESS_KEY_ID`, `R2_SECRET_ACCESS_KEY` |
| `/infisical` | `ENCRYPTION_KEY`, `AUTH_SECRET`, `REDIS_PASSWORD` ([generate](secrets.md#infisicals-own-keys)), `REDIS_URL` = `redis://:${REDIS_PASSWORD}@redis:6379`, `SITE_URL` = `https://infisical.home.<domain>` |

Add the folders for any other apps you kept ([step 1](#1-take-the-repo)).

**Infisical CLI:** in the repo, run `infisical login`, then `infisical init` and pick this
project. `infisical init` writes a new `.infisical.json`, which the `infisical secrets get`
commands in [bootstrap step 9](bootstrap.md#9-sops-key) read.

**SOPS key:** run `age-keygen`, put the public key in `.sops.yaml` and the private key in
Infisical Cloud as above ([bootstrap step 9](bootstrap.md#9-sops-key)).

**Machine identity:** create one with Universal Auth, no org access, and Viewer on this project
only. Encrypt its credentials into `infrastructure/configs/infisical-universal-auth.sops.yaml`
([secrets](secrets.md#rotate-the-operators-credentials)) and add the file back to
`infrastructure/configs/kustomization.yaml`.

Commit and push.

## 6. Flux

Run [bootstrap step 8](bootstrap.md#8-flux) with your owner and repo, then create `sops-age`
([bootstrap step 9](bootstrap.md#9-sops-key)). Flux then installs the rest in order:
`infrastructure`, then `infra-configs`, then `apps`.

## 7. Check

```sh
flux get kustomizations                 # all READY True
kubectl get certificate -n traefik      # home-wildcard READY True
kubectl get cluster -n infisical        # infisical-db: Cluster in healthy state
```

Then open `https://infisical.home.<domain>` and create the admin account right away. First boot
takes about 10 minutes ([infisical](infisical.md#first-boot)).

## Optional

- **Renovate:** install the GitHub App on the repo. `renovate.json` is ready ([updates](updates.md))
- **Your own apps:** follow the pattern of `apps/life-dashboard` (in git history after step 1): a
  private ghcr.io image built by GitHub Actions, Flux image automation (needs the bootstrap flags
  `--components-extra` and `--read-write-key`), and secrets through `InfisicalStaticSecret`

## Known rough edges

Read [incidents](incidents/). The two that apply to any rebuild:

- Put the machines on Ethernet and bridge the VMs onto the LAN, so Tailscale connects the nodes
  directly ([networking](networking.md#lima-vm-networking-bridged))
- Give k3s its own `resolv.conf` so pod DNS skips your router's search domains
  ([networking](networking.md#dns-inside-the-cluster))
