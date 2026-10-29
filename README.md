# homelab

k3s on three Mac minis and a Raspberry Pi, managed by Flux. Change the cluster by committing here.

## Layout

| Path | What |
|---|---|
| `lima/` | VM definitions for the minis |
| `k3s/` | k3s config copied onto each node |
| `clusters/homelab/` | Flux entrypoint: Flux itself plus pointers to the folders below |
| `infrastructure/controllers/` | Shared software, applied first (Gateway API, Traefik, cert-manager, CloudNativePG, Infisical Operator, Agent Sandbox) |
| `infrastructure/configs/` | Their configuration, applied second (Let's Encrypt issuer, certificate, Infisical Cloud connection) |
| `apps/` | Apps, applied last |
| `.sops.yaml` | Which files SOPS encrypts, and with which public key |
| `docs/` | Documentation |

## Docs

- [Reproduce](docs/reproduce.md): run this setup on your own machines
- [Plan](PLAN.md): build order and what runs on the cluster
- [Nodes](docs/nodes.md): hardware, VM sizes, resizing
- [Networking](docs/networking.md): Tailscale, reaching services
- [Bootstrap](docs/bootstrap.md): building the cluster from nothing
- [GitOps](docs/gitops.md): how Flux reads this repo, adding and removing apps
- [Secrets](docs/secrets.md): Infisical Cloud as source of truth, synced by the Infisical Operator
- [Storage](docs/storage.md): local-path volumes, CloudNativePG, backups
- [Infisical](docs/infisical.md): the cluster's Infisical, wiring, first boot
- [Authentik](docs/authentik.md): OIDC sign-in for the cluster's apps
- [Airlock](docs/airlock.md): agent sandboxes and the UI for claiming them
- [Certificates](docs/certificates.md): HTTPS for `*.home.jakehulberg.com`
- [Updates](docs/updates.md): Renovate, and what it doesn't cover
- [Incidents](docs/incidents/): what broke, why, what changed
- [Cheatsheet](docs/cheatsheet.md): everyday Lima, kubectl and Flux commands
- [Learning](learning/README.md): reading list for the concepts behind this repo
