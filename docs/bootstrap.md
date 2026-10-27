# Bootstrap

Building the cluster from nothing. Run `limactl` steps on each mini, from a checkout of this repo.

## 1. Cluster token

Generate once. Store as `K3S_CLUSTER_SECRET` in Infisical Cloud (homelab project, env `dev`, folder
`/`), plus a password manager copy. Cloud is fine since it isn't the Infisical inside this cluster.

```sh
openssl rand -hex 32
```

## 2. Create the VMs

On each mini (swap in `mini2` / `mini3`):

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

Disable key expiry for each `k3s-*` machine in the Tailscale admin console.

## 4. Copy the k3s config in

```sh
limactl copy k3s/config.yaml k3s/server.yaml k3s-mini1:/tmp/
limactl shell k3s-mini1 sudo sh -c '
  mkdir -p /etc/rancher/k3s/config.yaml.d &&
  cp /tmp/config.yaml /etc/rancher/k3s/config.yaml &&
  cp /tmp/server.yaml /etc/rancher/k3s/config.yaml.d/server.yaml &&
  printf "nameserver 1.1.1.1\nnameserver 8.8.8.8\n" > /etc/rancher/k3s/resolv.conf'
```

The `resolv.conf` keeps Tailscale's and the router's search domains out of pods
([networking](networking.md#dns-inside-the-cluster)).

## 5. Install k3s

Version is pinned so every node matches ([nodes](nodes.md)).

**a. Shell in** (on that mini):

```sh
limactl shell k3s-mini1
```

**b. Set the token.** Paste, press Enter (no echo). Run it alone, not pasted with the next command,
or `read` swallows that line as the token.

```sh
read -s K3S_TOKEN && export K3S_TOKEN
```

**c. Install.** mini1 creates the cluster:

```sh
curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION=v1.36.5+k3s1 sh -s - server \
  --cluster-init \
  --node-name k3s-mini1 \
  --node-ip "$(tailscale ip -4)"
```

mini2 and mini3 join (`k3s-mini3` on mini3):

```sh
curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION=v1.36.5+k3s1 sh -s - server \
  --server https://k3s-mini1:6443 \
  --node-name k3s-mini2 \
  --node-ip "$(tailscale ip -4)"
```

**d. Check:** `k3s --version` prints the pinned version.

## 6. Add the Pi

Bare-metal k3s agent (worker only), already on Tailscale.

**a. Enable memory cgroups and reboot** (on the Pi, `ssh pi`):

```sh
grep -q cgroup_enable=memory /boot/firmware/cmdline.txt \
  || sudo sed -i '1 s/$/ cgroup_memory=1 cgroup_enable=memory/' /boot/firmware/cmdline.txt
sudo reboot
```

**b. Copy the config** (laptop, in this repo, once the Pi is back). Only `config.yaml`;
`server.yaml` is for the minis.

```sh
scp k3s/config.yaml pi:/tmp/config.yaml
```

**c. Put it in place** (Pi):

```sh
sudo mkdir -p /etc/rancher/k3s
sudo mv /tmp/config.yaml /etc/rancher/k3s/config.yaml
sudo sh -c 'printf "nameserver 1.1.1.1\nnameserver 8.8.8.8\n" > /etc/rancher/k3s/resolv.conf'
```

**d. Set the token** (Pi; paste, Enter, no echo):

```sh
read -s K3S_TOKEN && export K3S_TOKEN
```

**e. Install as agent** (Pi):

```sh
curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION=v1.36.5+k3s1 K3S_URL=https://k3s-mini1:6443 sh -s - agent \
  --node-name pi \
  --node-ip "$(tailscale ip -4)"
```

**f. Check** (laptop): `kubectl get nodes` shows `pi` `Ready` within a minute.

## 7. kubeconfig on your laptop

k3s writes an admin kubeconfig in the first server VM, pointing at `127.0.0.1`. Copy it out, point
it at `k3s-mini1`, and merge it into `~/.kube/config` as context `homelab`. Never commit it: it's
full cluster admin.

**a. Copy it out.** SSH as the macOS user that created the VM, with the full `limactl` path
(non-interactive SSH skips Homebrew's PATH):

```sh
ssh mini1 '/opt/homebrew/bin/limactl shell k3s-mini1 sudo cat /etc/rancher/k3s/k3s.yaml' \
  | sed 's/127.0.0.1/k3s-mini1/' > /tmp/homelab.yaml
```

**b. Rename the context** (k3s calls it `default`):

```sh
KUBECONFIG=/tmp/homelab.yaml kubectl config rename-context default homelab
```

**c. Merge:**

```sh
KUBECONFIG=~/.kube/config:/tmp/homelab.yaml kubectl config view --flatten > /tmp/kubeconfig \
  && mv /tmp/kubeconfig ~/.kube/config && rm /tmp/homelab.yaml
```

**d. Use it:**

```sh
kubectl config use-context homelab
kubectl get nodes -o wide
```

`kubectl config get-contexts` lists clusters; `use-context <name>` switches.

## 8. Flux

Laptop, in this repo, on the `homelab` context.

**a. Install the CLI and pre-check:**

```sh
brew install fluxcd/tap/flux
flux check --pre
```

**b. Bootstrap.** Installs Flux, commits its manifests to `clusters/homelab/flux-system/`, and adds
a deploy key to the GitHub repo.

```sh
export GITHUB_TOKEN=$(gh auth token)
flux bootstrap github \
  --owner=jakehulberg \
  --repository=homelab \
  --private=false --personal \
  --path=clusters/homelab \
  --components-extra=image-reflector-controller,image-automation-controller \
  --read-write-key
git pull
```

- `--components-extra`: image automation controllers, which watch registries and commit tag bumps
  (used by life-dashboard).
- `--read-write-key`: lets those bumps be pushed.
- Change Flux's own install (components, repo name) by re-running bootstrap, never by editing
  `gotk-components.yaml` or `gotk-sync.yaml`.

**c. Check:**

```sh
flux check                    # "all checks passed"
flux get kustomizations       # flux-system, infrastructure, infra-configs, apps all READY True
```

From here on, change the cluster by committing.

## 9. SOPS key

Flux decrypts this repo's SOPS files with an age private key from the `sops-age` Secret. It's the
one secret never in git, so recreate it on every rebuild.

**First time only: make the key.**

```sh
brew install sops age
age-keygen -o /tmp/age.txt
```

- `# public key: age1...` goes in `.sops.yaml` (safe to commit).
- `AGE-SECRET-KEY-1...` goes in Infisical Cloud as `SOPS_AGE_KEY` (homelab project, env `dev`,
  folder `/`), plus a password manager copy.
- `rm /tmp/age.txt`.

**Every rebuild: give it to the cluster.** First confirm it comes back; if this prints nothing, the
next command creates an empty Secret and decryption fails.

```sh
infisical secrets get SOPS_AGE_KEY --plain --env=dev --path=/
```

```sh
infisical secrets get SOPS_AGE_KEY --plain --env=dev --path=/ \
  | kubectl create secret generic sops-age -n flux-system --from-file=age.agekey=/dev/stdin
flux reconcile kustomization apps --with-source
```

## 10. Infisical Operator login

With `sops-age` in place, Flux decrypts the operator's credentials
(`infrastructure/configs/infisical-universal-auth.sops.yaml`) and the operator syncs everything
else. Nothing to do on a rebuild unless that Client Secret was revoked.

**First time only:** in Infisical Cloud, create machine identity `homelab-cluster` (Universal Auth,
no org-level access), add it to the `homelab` project as Viewer, create a Client Secret, then build
the SOPS file ([secrets](secrets.md#rotate-the-operators-credentials)).

```sh
kubectl get infisicalauth -n infisical-operator-system   # READY True
kubectl get infisicalstaticsecret -A
```
