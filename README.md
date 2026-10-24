# homelab

A k3s cluster on three Mac minis and a Raspberry Pi, managed with Flux. This repo is the source
of truth: the cluster is changed by committing here.

## Layout

| Path | What |
|---|---|
| `lima/` | VM definitions for the Mac minis |
| `k3s/` | k3s config copied onto each node |
| `clusters/homelab/` | Flux entrypoint: Flux itself plus pointers to `infrastructure/` and `apps/` |
| `infrastructure/` | Shared services, applied first (Traefik, cert-manager, storage) |
| `apps/` | Apps, applied once infrastructure is healthy |
| `.sops.yaml` | Which files SOPS encrypts, and with which public key |
| `docs/` | Documentation |

## Docs

- [Plan](PLAN.md): build order and what runs on the cluster
- [Nodes](docs/nodes.md): hardware, VM sizes, resizing
- [Networking](docs/networking.md): Tailscale, reaching services
- [Bootstrap](docs/bootstrap.md): building the cluster from nothing
- [GitOps](docs/gitops.md): how Flux reads this repo, adding and removing apps
- [Secrets](docs/secrets.md): encrypting secrets with SOPS
- [Cheatsheet](docs/cheatsheet.md): everyday Lima, kubectl and Flux commands
