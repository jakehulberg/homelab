# homelab

A k3s cluster on three Mac minis and a Raspberry Pi, managed with Flux. This repo is the source
of truth: the cluster is changed by committing here.

## Layout

| Path | What |
|---|---|
| `lima/` | VM definitions for the Mac minis |
| `k3s/` | k3s config copied onto each node |
| `clusters/homelab/` | Flux entrypoint: Flux itself plus pointers to the folders below |
| `infrastructure/controllers/` | Shared software, applied first (Gateway API, Traefik, cert-manager, CloudNativePG, Infisical Operator) |
| `infrastructure/configs/` | Their configuration, applied second (Let's Encrypt issuer, certificate, Infisical Cloud connection) |
| `apps/` | Apps, applied last |
| `.sops.yaml` | Which files SOPS encrypts, and with which public key |
| `docs/` | Documentation |

## Docs

- [Plan](PLAN.md): build order and what runs on the cluster
- [Nodes](docs/nodes.md): hardware, VM sizes, resizing
- [Networking](docs/networking.md): Tailscale, reaching services
- [Bootstrap](docs/bootstrap.md): building the cluster from nothing
- [GitOps](docs/gitops.md): how Flux reads this repo, adding and removing apps
- [Secrets](docs/secrets.md): Infisical Cloud as source of truth, synced by the Infisical Operator
- [Storage](docs/storage.md): local-path volumes, CloudNativePG databases, backups
- [Infisical](docs/infisical.md): the cluster's Infisical, how it's wired, first boot
- [Certificates](docs/certificates.md): HTTPS for `*.home.jakehulberg.com` via cert-manager and Let's Encrypt
- [Incidents](docs/incidents/): what broke, why, and what changed
- [Cheatsheet](docs/cheatsheet.md): everyday Lima, kubectl and Flux commands
- [Learning](learning/README.md): reading list for the concepts behind this repo
