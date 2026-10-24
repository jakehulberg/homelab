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

The version is pinned so every node matches (see `docs/nodes.md`).

**a. Shell into the VM** (on that mini):

```sh
limactl shell k3s-mini1
```

**b. Set the token.** Paste it and press Enter; nothing shows on screen. Run this on its own line,
not pasted together with the next command, or `read` swallows the next line as the token.

```sh
read -s K3S_TOKEN && export K3S_TOKEN
```

**c. Install.** The first server (mini1) creates the cluster:

```sh
curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION=v1.36.5+k3s1 sh -s - server \
  --cluster-init \
  --node-name k3s-mini1 \
  --node-ip "$(tailscale ip -4)"
```

mini2 and mini3 join it (use `k3s-mini3` on mini3):

```sh
curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION=v1.36.5+k3s1 sh -s - server \
  --server https://k3s-mini1:6443 \
  --node-name k3s-mini2 \
  --node-ip "$(tailscale ip -4)"
```

**d. Check.** `k3s --version` should print the pinned version.

## 6. Add the Pi

The Pi runs k3s directly (no VM) and is already on Tailscale. It joins as an agent (worker only).

**a. Enable memory cgroups and reboot.** On the Pi (`ssh pi`):

```sh
grep -q cgroup_enable=memory /boot/firmware/cmdline.txt \
  || sudo sed -i '1 s/$/ cgroup_memory=1 cgroup_enable=memory/' /boot/firmware/cmdline.txt
sudo reboot
```

**b. Copy the k3s config over.** From your laptop, in this repo (after the Pi is back up):

```sh
scp k3s/config.yaml pi:/tmp/config.yaml
```

Only `config.yaml`. `server.yaml` is for the minis.

**c. Put the config in place.** On the Pi:

```sh
sudo mkdir -p /etc/rancher/k3s
sudo mv /tmp/config.yaml /etc/rancher/k3s/config.yaml
```

**d. Set the token.** On the Pi. Paste it and press Enter; nothing shows on screen.

```sh
read -s K3S_TOKEN && export K3S_TOKEN
```

**e. Install k3s as an agent.** On the Pi:

```sh
curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION=v1.36.5+k3s1 K3S_URL=https://k3s-mini1:6443 sh -s - agent \
  --node-name pi \
  --node-ip "$(tailscale ip -4)"
```

**f. Check it joined.** From your laptop:

```sh
kubectl get nodes
```

`pi` should show `Ready` within a minute.

## 7. kubeconfig on your laptop

k3s writes an admin kubeconfig inside the first server VM. It points at `127.0.0.1`, so copy it
out, point it at `k3s-mini1` (reachable over Tailscale), and merge it into `~/.kube/config` as a
context named `homelab`. Never commit this file: it is full admin access to the cluster.

**a. Copy it out.** From your laptop. SSH in as the macOS user that created the VM, and use the
full `limactl` path, since non-interactive SSH doesn't load Homebrew's PATH:

```sh
ssh mini1 '/opt/homebrew/bin/limactl shell k3s-mini1 sudo cat /etc/rancher/k3s/k3s.yaml' \
  | sed 's/127.0.0.1/k3s-mini1/' > /tmp/homelab.yaml
```

**b. Rename its context.** k3s names it `default`:

```sh
KUBECONFIG=/tmp/homelab.yaml kubectl config rename-context default homelab
```

**c. Merge it into `~/.kube/config`:**

```sh
KUBECONFIG=~/.kube/config:/tmp/homelab.yaml kubectl config view --flatten > /tmp/kubeconfig \
  && mv /tmp/kubeconfig ~/.kube/config && rm /tmp/homelab.yaml
```

**d. Use it:**

```sh
kubectl config use-context homelab
kubectl get nodes -o wide
```

Switch clusters with `kubectl config use-context <name>`; list them with `kubectl config get-contexts`.

## 8. Flux

From your laptop, in this repo, with `kubectl` on the `homelab` context.

**a. Install the CLI and check the cluster:**

```sh
brew install fluxcd/tap/flux
flux check --pre
```

**b. Bootstrap.** This installs Flux into the cluster, commits its manifests to
`clusters/homelab/flux-system/`, and adds a deploy key to the GitHub repo so the cluster can pull it.

```sh
export GITHUB_TOKEN=$(gh auth token)
flux bootstrap github \
  --owner=jakehulberg \
  --repository=homelab-private \
  --private --personal \
  --path=clusters/homelab
git pull
```

**c. Check:**

```sh
flux check                    # ends with "all checks passed"
flux get kustomizations       # flux-system, infrastructure, apps all READY True
```

From here on, the cluster is changed by committing to this repo.

## 9. SOPS key

Flux decrypts the encrypted secrets in this repo with an age private key, which the cluster reads
from the `sops-age` Secret. That Secret is the one thing never stored in git, so recreate it on
every rebuild.

**First time only: make the key.**

```sh
brew install sops age
age-keygen -o /tmp/age.txt
```

- The `# public key: age1...` line goes in `.sops.yaml` (safe to commit).
- The `AGE-SECRET-KEY-1...` line goes in Infisical as `SOPS_AGE_KEY` (env `dev`, folder
  `/homelab`), with a second copy in a password manager. Infisical runs on this cluster, so it
  can't be the only copy.
- Then `rm /tmp/age.txt`.

**Every rebuild: give the key to the cluster.**

First check the key comes back. If this prints nothing, the next command would create an empty
Secret and decryption fails.

```sh
infisical secrets get SOPS_AGE_KEY --plain --env=dev --path=/homelab
```

Then:

```sh
infisical secrets get SOPS_AGE_KEY --plain --env=dev --path=/homelab \
  | kubectl create secret generic sops-age -n flux-system --from-file=age.agekey=/dev/stdin
flux reconcile kustomization apps --with-source
```

See [secrets](secrets.md) for adding encrypted secrets.
