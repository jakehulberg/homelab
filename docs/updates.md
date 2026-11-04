# Updates

Renovate opens a PR for each update to software pinned in this repo, and merging the PR is the
deploy. This page covers what Renovate tracks, what to check before merging, and what needs
updating by hand.

## Software in the repo: Renovate

[Renovate](https://github.com/renovatebot/renovate) runs as the Mend-hosted GitHub App, on this
repo only. It opens a PR per available update, with release notes, and Flux deploys the change
once the PR is merged. Its Dependency Dashboard issue lists everything Renovate tracks.

The config is `renovate.json`:

- **Managers:** `flux`, `kubernetes` and `helm-values` scan every YAML file, so Renovate picks up
  new HelmReleases, OCIRepositories and images with no per-app setup
- **Release age:** with `minimumReleaseAge: "3 days"`, Renovate opens a PR only once a release is
  3 days old
- **Flux:** `clusters/homelab/flux-system/` is excluded from the `kubernetes` manager. Flux's
  controllers ship as one set, and the `flux` manager upgrades them together; bumping images one
  by one could mix versions of the controllers that deploy everything else

Before merging a Renovate PR:

- Read the release notes for breaking changes, especially chart value renames
- For a chart, diff `helm show values <chart> --version <new>` against the current version
- For a database update (CloudNativePG, Barman plugin), check `kubectl get cluster -A` afterwards

Images with a `{"$imagepolicy": ...}` marker (life-dashboard, Airlock) aren't Renovate PRs: Flux
commits their new tags straight to `main` ([gitops](gitops.md)).

### Not tracked by Renovate

| What | Where | How to update |
|---|---|---|
| Gateway API CRDs | `infrastructure/controllers/gateway-api/standard-install.yaml` (downloaded) | Overwrite with the new release's `standard-install.yaml` |
| Agent Sandbox | `infrastructure/controllers/agent-sandbox/sandbox-with-extensions.yaml` (downloaded) | Overwrite with the new release's `sandbox-with-extensions.yaml`. Airlock needs the `v1beta1` API |
| Prometheus Operator CRDs | `infrastructure/controllers/prometheus-operator-crds/crds.yaml` (downloaded) | When Renovate bumps kube-prometheus-stack, overwrite with `stripped-down-crds.yaml` from the operator release that matches the chart's `APP VERSION` (`helm search repo prometheus-community/kube-prometheus-stack`). Push it with or before the chart bump |
| k3s | Each node; version in `docs/nodes.md` | Reinstall with the new `INSTALL_K3S_VERSION`, one node at a time ([nodes](nodes.md#upgrading-k3s)) |
| Lima VM image | `lima/*.yaml` | A new image only affects new VMs; existing VMs update through their OS |

## The machines

Not covered yet. The plan is `unattended-upgrades` in the VMs and on the Pi. macOS updates
itself.

## Private repos

The hosted app reads repo code on Mend's servers. That's fine for this repo. For a private repo
with personal data in its code (life-dashboard), self-host Renovate instead, e.g. as a CronJob in
the cluster.
