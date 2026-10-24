# Bootstrap

Building the cluster from nothing. Run the `limactl` steps on each mini, from a checkout of this repo.

## 1. Cluster token

Generate once and store it in your password manager. It can't live in Infisical, since Infisical
runs inside the cluster.

```sh
openssl rand -hex 32
```

## 2. Create the VMs

On each mini (replace `mini1` with `mini2` / `mini3`):

```sh
brew install lima
limactl create --name=k3s-mini1 lima/k3s-mini1.yaml
limactl start k3s-mini1
limactl autostart enable k3s-mini1
```

## 3. Join the VMs to Tailscale

```sh
limactl shell k3s-mini1 sudo tailscale up --hostname=k3s-mini1
```

In the Tailscale admin console, disable key expiry for each `k3s-*` machine.

## 4. Copy the k3s config in

```sh
limactl copy k3s/config.yaml k3s/server.yaml k3s-mini1:/tmp/
limactl shell k3s-mini1 sudo sh -c '
  mkdir -p /etc/rancher/k3s/config.yaml.d &&
  cp /tmp/config.yaml /etc/rancher/k3s/config.yaml &&
  cp /tmp/server.yaml /etc/rancher/k3s/config.yaml.d/server.yaml'
```

## 5. Install k3s

First server (mini1), which creates the cluster:

```sh
limactl shell k3s-mini1
curl -sfL https://get.k3s.io | K3S_TOKEN=<token> sh -s - server \
  --cluster-init \
  --node-name k3s-mini1 \
  --node-ip "$(tailscale ip -4)"
```

mini2 and mini3 join it:

```sh
limactl shell k3s-mini2
curl -sfL https://get.k3s.io | K3S_TOKEN=<token> sh -s - server \
  --server https://k3s-mini1:6443 \
  --node-name k3s-mini2 \
  --node-ip "$(tailscale ip -4)"
```

Record the installed version (`k3s --version`) in `docs/nodes.md` and pin it with
`INSTALL_K3S_VERSION` on later installs.

## 6. Add the Pi

On the Pi (already on Tailscale), enable memory cgroups: append
`cgroup_memory=1 cgroup_enable=memory` to the single line in `/boot/firmware/cmdline.txt`, then reboot.

Copy `k3s/config.yaml` to `/etc/rancher/k3s/config.yaml` (not `server.yaml`), then:

```sh
curl -sfL https://get.k3s.io | K3S_URL=https://k3s-mini1:6443 K3S_TOKEN=<token> sh -s - agent \
  --node-name pi \
  --node-ip "$(tailscale ip -4)"
```

## 7. kubeconfig on your laptop

```sh
ssh mini1 'limactl shell k3s-mini1 sudo cat /etc/rancher/k3s/k3s.yaml' \
  | sed 's/127.0.0.1/k3s-mini1/' > ~/.kube/homelab.yaml
export KUBECONFIG=~/.kube/homelab.yaml
kubectl get nodes -o wide
```

## 8. Flux

```sh
flux bootstrap github \
  --owner=jakehulberg \
  --repository=homelab-private \
  --private --personal \
  --path=clusters/homelab
```

From here on, the cluster is changed by committing to this repo.
