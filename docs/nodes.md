# Nodes

| Node | Host | Role | CPUs | RAM | Disk |
|---|---|---|---|---|---|
| k3s-mini1 | mini1 (Lima VM) | server | 4 | 6GiB | 15GiB |
| k3s-mini2 | mini2 (Lima VM) | server | 6 | 10GiB | 50GiB |
| k3s-mini3 | mini3 (Lima VM) | server | 6 | 6GiB | 35GiB |
| pi | Raspberry Pi 5 (bare metal) | agent | 4 | all | SD/SSD |

Servers run the control plane (API server, scheduler, embedded etcd) plus workloads. Three servers
survive losing any one mini.

k3s `v1.37.1+k3s1` on every node (`INSTALL_K3S_VERSION=v1.37.1+k3s1`).

## Hosts

All three minis: Apple M4 (4 performance + 6 efficiency cores), 16GB RAM, 228GB disk.

| Host | Use outside the cluster | RAM left for macOS | Free disk (Oct 2026) |
|---|---|---|---|
| mini1 | Daily software development | 10GB | 38GB |
| mini2 | Mostly idle | 6GB | 74GB |
| mini3 | Demo Infisical in Docker (stays outside the cluster) | 10GB | 54GB |

Other VMs on the hosts, counted in "RAM left":

- mini1: Docker Desktop (~1.7GB)
- mini2: two Virtualization.framework VMs (~1.6GB)
- mini3: Docker Desktop capped at 6GB (Settings → Resources), running the demo Infisical and other
  demo containers (~5GB used). With k3s-mini3 at 6GiB, macOS gets ~4GB. Uncapped, Docker could take
  11.7GB, which with an 8GiB k3s VM overcommitted the host.

## Taking a node down

For a resize, network change or reboot. **One node at a time:** etcd needs two of the three minis up.

**1. Laptop: move any database primary off the node.** CloudNativePG blocks draining a primary.

```sh
kubectl get cluster -A                                        # PRIMARY column
brew install kubectl-cnpg                                     # once
kubectl cnpg promote infisical-db infisical-db-2 -n infisical # promote the copy on another node
kubectl get cluster infisical-db -n infisical                 # wait for "Cluster in healthy state"
```

**2. Laptop: drain.** Pods move, except database instances with data on this node, which stay
Pending until it's back.

```sh
kubectl drain k3s-mini3 --ignore-daemonsets --delete-emptydir-data
```

**3. On the mini, as the VM's owner** (`theodorecomputeiii` on mini1, `mini-2`, `mini3`):

```sh
limactl stop k3s-mini3
limactl edit k3s-mini3 --memory 10   # GiB; also --cpus, --disk; answer yes to start it
```

**4. Laptop: bring it back** once `kubectl get nodes` shows `Ready`:

```sh
kubectl uncordon k3s-mini3
kubectl get cluster -A               # all database instances ready
```

Update the matching `lima/` file so the repo stays the source of truth.

## Upgrading k3s

Rerun each node's install command from [bootstrap](bootstrap.md#5-install-k3s) with the new
`INSTALL_K3S_VERSION`: mini1, mini2, mini3, then the Pi. One node at a time; wait for `Ready` on the
new version in `kubectl get nodes` before the next. No drain: containers keep running while k3s
restarts. The containerd template for gVisor and `/etc/rancher/k3s/config.yaml.d/` are kept.

- **In the VM**, not on the Mac: `limactl shell k3s-miniN` first (the prompt shows `lima-k3s-miniN`).
- **Token first, on its own.** The installer rewrites the service's env file, and the Pi's agent
  can't rejoin without `K3S_TOKEN`. Check `echo ${#K3S_TOKEN}` prints about 140 before installing.
  The token is in any server VM at `/var/lib/rancher/k3s/server/token`: copy the whole line.
- **While mini1 restarts**, kubectl (pointed at `k3s-mini1`) fails; use
  `kubectl --server https://k3s-mini2:6443 get nodes`.
- Then update the version here and in [bootstrap](bootstrap.md).
