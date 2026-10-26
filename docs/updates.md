# Updates

## Software in the repo: Renovate

[Renovate](https://github.com/renovatebot/renovate) (the Mend-hosted GitHub App, access to this
repo only) scans the repo and opens a pull request for each available update, with release notes
attached. Merge it and Flux deploys it. It also keeps a Dependency Dashboard issue listing
everything it tracks.

Config: `renovate.json`.

- `flux` and `kubernetes` managers read every YAML file, so new HelmReleases, OCIRepositories and
  container images are picked up automatically, with no per-app setup.
- `clusters/homelab/flux-system/` is excluded from the `kubernetes` manager. Flux's controllers are
  released as one set, and the `flux` manager upgrades them together; bumping them image by image
  could mix versions of the thing that deploys everything else.

Before merging an update:

- Read the release notes for breaking changes, especially chart value renames.
- For a chart, compare values: `helm show values <chart> --version <new>`.
- Database-related updates (CloudNativePG, the Barman plugin): check `kubectl get cluster -A`
  afterwards.

### Not tracked by Renovate

| What | Where | How to update |
|---|---|---|
| Gateway API CRDs | `infrastructure/controllers/gateway-api/standard-install.yaml` (downloaded file) | Download the new release's `standard-install.yaml` over it |
| k3s | Installed on each node; version recorded in `docs/nodes.md` | Reinstall with the new `INSTALL_K3S_VERSION`, one node at a time ([nodes](nodes.md#taking-a-node-down)); later, Rancher's system-upgrade-controller |
| Lima VM image | `lima/*.yaml` | Only matters for new VMs; existing VMs update through their OS |

## The machines

Not covered yet. Planned: Ubuntu's automatic security updates (`unattended-upgrades`) in the VMs and
on the Pi. macOS on the minis updates on its own schedule.

## Private repos

The hosted app reads repo code on Mend's servers. Fine for this repo and public sites; for private
repos with personal data in the code (life-dashboard), run Renovate self-hosted instead, for
example as a CronJob in the cluster.
