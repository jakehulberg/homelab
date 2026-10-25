# Nodes

| Node | Host | Role | CPUs | RAM | Disk |
|---|---|---|---|---|---|
| k3s-mini1 | mini1 (Lima VM) | server | 4 | 6GiB | 15GiB |
| k3s-mini2 | mini2 (Lima VM) | server | 6 | 10GiB | 50GiB |
| k3s-mini3 | mini3 (Lima VM) | server | 6 | 6GiB | 35GiB |
| pi | Raspberry Pi 5 (bare metal) | agent | 4 | all | SD/SSD |

Servers run the control plane (API server, scheduler, embedded etcd) and also run workloads.
Three servers keep the cluster up if any one mini goes down.

k3s version: `v1.36.5+k3s1`. Install the same version on every node with
`INSTALL_K3S_VERSION=v1.36.5+k3s1`.

## Hosts

All three minis are Apple M4 (4 performance + 6 efficiency cores), 16GB RAM, 228GB disk.

| Host | Use outside the cluster | RAM left for macOS | Free disk (Oct 2026) |
|---|---|---|---|
| mini1 | Daily software development | 10GB | 38GB |
| mini2 | Mostly idle | 6GB | 74GB |
| mini3 | Runs the demo Infisical instance in Docker (stays outside the cluster) | 10GB | 54GB |

Other VMs already on the hosts, counted against the "RAM left" column:

- mini1: Docker Desktop VM (~1.7GB)
- mini2: two Virtualization.framework VMs (~1.6GB)
- mini3: Docker Desktop, capped at 6GB (Settings → Resources), running the demo Infisical and
  other demo containers (~5GB in use). With k3s-mini3 at 6GiB, that leaves macOS about 4GB.
  Before the cap, Docker was allowed 11.7GB, which with an 8GiB k3s VM overcommitted the 16GB host.

## Taking a node down

For a VM resize, a network change, or a reboot. **One node at a time:** the three minis each hold
a copy of etcd, and the cluster needs two of three up.

**1. Laptop: move a database primary off the node, if it has one.** CloudNativePG won't let a
drain evict a primary.

```sh
kubectl get cluster -A                                        # PRIMARY column
brew install kubectl-cnpg                                     # once
kubectl cnpg promote infisical-db infisical-db-2 -n infisical # promote the copy on another node
kubectl get cluster infisical-db -n infisical                 # wait for "Cluster in healthy state"
```

**2. Laptop: drain.** Pods move elsewhere, except database instances whose data lives on this
node; they wait as Pending until it's back.

```sh
kubectl drain k3s-mini3 --ignore-daemonsets --delete-emptydir-data
```

**3. On the mini, as the user that owns the VM** (`theodorecomputeiii` on mini1, `mini-2`,
`mini3`):

```sh
limactl stop k3s-mini3
limactl edit k3s-mini3 --memory 10   # GiB; also --cpus, --disk; answer yes to start it
```

**4. Laptop: bring it back** once `kubectl get nodes` shows it `Ready`:

```sh
kubectl uncordon k3s-mini3
kubectl get cluster -A               # databases back to all instances ready
```

Update the matching file in `lima/` so the repo stays the source of truth.
