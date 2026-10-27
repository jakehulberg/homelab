# Updates

## Software in the repo: Renovate

[Renovate](https://github.com/renovatebot/renovate) (Mend-hosted GitHub App, this repo only) opens a
PR per available update, with release notes. Merge and Flux deploys. Its Dependency Dashboard issue
lists everything tracked.

Config: `renovate.json`.

- The `flux` and `kubernetes` managers scan every YAML file, so new HelmReleases, OCIRepositories
  and images are picked up with no per-app setup.
- `clusters/homelab/flux-system/` is excluded from `kubernetes`. Flux's controllers ship as one set
  and the `flux` manager upgrades them together; bumping images individually could mix versions of
  the thing that deploys everything else.

Before merging:

- Read release notes for breaking changes, especially chart value renames.
- Charts: diff `helm show values <chart> --version <new>`.
- Database updates (CloudNativePG, Barman plugin): check `kubectl get cluster -A` afterwards.

### Not tracked by Renovate

| What | Where | How to update |
|---|---|---|
| Gateway API CRDs | `infrastructure/controllers/gateway-api/standard-install.yaml` (downloaded) | Overwrite with the new release's `standard-install.yaml` |
| k3s | Each node; version in `docs/nodes.md` | Reinstall with the new `INSTALL_K3S_VERSION`, one node at a time ([nodes](nodes.md#taking-a-node-down)); later, Rancher's system-upgrade-controller |
| Lima VM image | `lima/*.yaml` | Only affects new VMs; existing VMs update through their OS |

## The machines

Not covered yet. Planned: `unattended-upgrades` in the VMs and on the Pi. macOS updates on its own.

## Private repos

The hosted app reads repo code on Mend's servers. Fine here; for private repos with personal data in
code (life-dashboard), self-host Renovate instead, e.g. as a CronJob in the cluster.
