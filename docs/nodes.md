# Nodes

The cluster has four nodes: a k3s server VM on each of three Mac minis, and a Raspberry Pi 5 as an
agent. Every node runs k3s `v1.37.1+k3s1` (`INSTALL_K3S_VERSION=v1.37.1+k3s1`).

| Node | Host | Role | CPUs | RAM | Disk |
|---|---|---|---|---|---|
| k3s-mini1 | mini1 (Lima VM) | server | 4 | 6GiB | 30GiB |
| k3s-mini2 | mini2 (Lima VM) | server | 6 | 10GiB | 50GiB |
| k3s-mini3 | mini3 (Lima VM) | server | 6 | 6GiB | 35GiB |
| pi | Raspberry Pi 5 (bare metal) | agent | 4 | all | SD/SSD |

The three servers run the control plane (API server, scheduler, embedded etcd) and workloads.
etcd needs two of the three up, so the cluster survives losing any one mini. VM sizes come from
`lima/k3s-mini*.yaml`.

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
  demo containers (~5GB used). With k3s-mini3 at 6GiB, macOS gets ~4GB. Keep the cap: uncapped,
  Docker can take 11.7GB and overcommit the host

## Taking a node down

Use this for a resize, a network change or a reboot. **One node at a time:** etcd needs two of
the three minis up.

### 1. Move database primaries off the node

On the laptop. CloudNativePG blocks draining a node that runs a primary.

```sh
kubectl get cluster -A                                        # PRIMARY column
brew install kubectl-cnpg                                     # once
kubectl cnpg promote infisical-db infisical-db-2 -n infisical # promote the copy on another node
kubectl get cluster infisical-db -n infisical                 # wait for "Cluster in healthy state"
```

### 2. Drain the node

On the laptop. Pods move to other nodes, except database instances with data on this node, which
stay Pending until it's back.

```sh
kubectl drain k3s-mini3 --ignore-daemonsets --delete-emptydir-data
```

### 3. Stop and change the VM

On the mini, as the VM's owner (`theodorecomputeiii` on mini1, `mini-2`, `mini3`):

```sh
limactl shell k3s-mini3 sudo poweroff
limactl stop -f k3s-mini3
limactl edit k3s-mini3 --memory 10   # GiB; also --cpus, --disk; answer yes to start it
```

`limactl stop` on its own hangs at `Shutting down VZ` on these VMs. Powering off from inside the
VM shuts k3s down cleanly, and `stop -f` then clears Lima's leftover files.

If you changed the VM's size, update its `lima/` file to match, so the repo stays the source of
truth.

### 4. Bring the node back

On the laptop, once `kubectl get nodes` shows the node `Ready`:

```sh
kubectl uncordon k3s-mini3
kubectl get cluster -A               # all database instances ready
```

## Upgrading k3s

Upgrade one node at a time, in this order: mini1, mini2, mini3, then the Pi. No drain is needed:
containers keep running while k3s restarts. The upgrade keeps the containerd template for gVisor
and `/etc/rancher/k3s/config.yaml.d/`.

For each node:

1. On a mini, open a shell in the VM, not on the Mac: `limactl shell k3s-miniN` (the prompt shows
   `lima-k3s-miniN`)
2. Set `K3S_TOKEN`, in its own command, and check that `echo ${#K3S_TOKEN}` prints about 140. The
   token is in any server VM at `/var/lib/rancher/k3s/server/token`; copy the whole line. The
   installer rewrites the k3s service's env file, and without `K3S_TOKEN` the Pi's agent can't
   rejoin
3. Rerun the node's install command from [bootstrap](bootstrap.md#5-install-k3s) with the new
   `INSTALL_K3S_VERSION`
4. Wait until `kubectl get nodes` shows the node `Ready` on the new version

While mini1 restarts, kubectl fails, because the kubeconfig points at `k3s-mini1`. Use
`kubectl --server https://k3s-mini2:6443 get nodes` instead.

After the last node, update the version on this page and in [bootstrap](bootstrap.md).
