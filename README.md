# homelab

A k3s cluster on three Mac minis and a Raspberry Pi, managed with Flux. This repo is the source
of truth: the cluster is changed by committing here.

## Layout

| Path | What |
|---|---|
| `lima/` | VM definitions for the Mac minis |
| `k3s/` | k3s config copied onto each node |
| `clusters/homelab/` | Flux entrypoint (created by `flux bootstrap`) |
| `docs/` | Documentation |

## Docs

- [Plan](PLAN.md): build order and what runs on the cluster
- [Nodes](docs/nodes.md): hardware, VM sizes, resizing
- [Networking](docs/networking.md): Tailscale, reaching services
- [Bootstrap](docs/bootstrap.md): building the cluster from nothing
