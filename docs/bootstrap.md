# Bootstrap

Build the cluster from nothing: three k3s servers in Lima VMs on the minis, the Pi as a k3s
agent, and Flux applying this repo. Steps 1 to 7 build the nodes; steps 8 to 11 hand the cluster
to Flux.

## Prerequisites

- A checkout of this repo on each mini and on the laptop
- The Pi on the tailnet: `tailscale status` on the laptop lists it
- The accounts and laptop tools in [reproduce](reproduce.md#prerequisites)

## 1. Cluster token

Generate the token once:

```sh
openssl rand -hex 32
```

Store it as `K3S_CLUSTER_SECRET` in Infisical Cloud (homelab project, env `dev`, folder `/`), plus
a password manager copy. Infisical Cloud runs outside this cluster, so the token is still reachable
when the cluster is down.

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

Then disable key expiry for each `k3s-*` machine in the Tailscale admin console.

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

The version is pinned so every node matches ([nodes](nodes.md#upgrading-k3s)).

**a. Shell in** (on that mini):

```sh
limactl shell k3s-mini1
```

**b. Set the token.** Run this command on its own, then paste the token and press Enter (the
input isn't echoed). If you paste it together with the next command, `read` takes that command
as the token.

```sh
read -s K3S_TOKEN && export K3S_TOKEN
```

**c. Install.** mini1 creates the cluster:

```sh
curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION=v1.37.1+k3s1 sh -s - server \
  --cluster-init \
  --node-name k3s-mini1 \
  --node-ip "$(tailscale ip -4)"
```

mini2 and mini3 join it (use `k3s-mini3` on mini3):

```sh
curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION=v1.37.1+k3s1 sh -s - server \
  --server https://k3s-mini1:6443 \
  --node-name k3s-mini2 \
  --node-ip "$(tailscale ip -4)"
```

**d. Check:**

```sh
k3s --version   # v1.37.1+k3s1
```

## 6. Add the Pi

The Pi runs k3s directly on its OS, as an agent (worker only).

**a. Enable memory cgroups and reboot** (on the Pi, `ssh pi`):

```sh
grep -q cgroup_enable=memory /boot/firmware/cmdline.txt \
  || sudo sed -i '1 s/$/ cgroup_memory=1 cgroup_enable=memory/' /boot/firmware/cmdline.txt
sudo reboot
```

**b. Copy the config** (laptop, in this repo, once the Pi is back). Copy only `config.yaml`;
`server.yaml` is for the minis.

```sh
scp k3s/config.yaml pi:/tmp/config.yaml
```

**c. Put the config in place** (Pi):

```sh
sudo mkdir -p /etc/rancher/k3s
sudo mv /tmp/config.yaml /etc/rancher/k3s/config.yaml
sudo sh -c 'printf "nameserver 1.1.1.1\nnameserver 8.8.8.8\n" > /etc/rancher/k3s/resolv.conf'
```

**d. Set the token** (Pi), the same way as [step 5b](#5-install-k3s):

```sh
read -s K3S_TOKEN && export K3S_TOKEN
```

**e. Install as agent** (Pi):

```sh
curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION=v1.37.1+k3s1 K3S_URL=https://k3s-mini1:6443 sh -s - agent \
  --node-name pi \
  --node-ip "$(tailscale ip -4)"
```

**f. Check** (laptop, after step 7):

```sh
kubectl get nodes   # pi Ready within a minute
```

## 7. kubeconfig on your laptop

k3s writes an admin kubeconfig on the first server VM, pointing at `127.0.0.1`. These steps copy
it to the laptop, point it at `k3s-mini1`, and merge it into `~/.kube/config` as context
`homelab`. Never commit the kubeconfig: it's full cluster admin.

**a. Copy it out.** SSH as the macOS user that created the VM, and give the full `limactl` path,
because non-interactive SSH doesn't load Homebrew's PATH:

```sh
ssh mini1 '/opt/homebrew/bin/limactl shell k3s-mini1 sudo cat /etc/rancher/k3s/k3s.yaml' \
  | sed 's/127.0.0.1/k3s-mini1/' > /tmp/homelab.yaml
```

**b. Rename the context** from k3s's `default`:

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
kubectl get nodes -o wide   # k3s-mini1, k3s-mini2, k3s-mini3 and pi, all Ready
```

## 8. Flux

Run these on the laptop, in this repo, on the `homelab` context.

**a. Install the CLI and run the pre-check:**

```sh
brew install fluxcd/tap/flux
flux check --pre
```

**b. Bootstrap.** `flux bootstrap` installs Flux, commits its manifests to
`clusters/homelab/flux-system/`, and adds a deploy key to the GitHub repo.

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

- `--components-extra`: adds the image automation controllers, which watch registries and commit
  tag bumps (used by life-dashboard and airlock)
- `--read-write-key`: lets Flux push those commits
- To change Flux's own install (components, repo name), re-run bootstrap. Don't edit
  `gotk-components.yaml` or `gotk-sync.yaml`

**c. Check:**

```sh
flux check                    # all checks passed
flux get kustomizations       # flux-system, infrastructure, infra-configs, apps all READY True
```

From here on, change the cluster by committing ([gitops](gitops.md)).

## 9. SOPS key

Flux decrypts this repo's SOPS files with an age private key from the `sops-age` Secret. That key
is the one secret that's never in git, so recreate the Secret on every rebuild.

**First time only: make the key.**

```sh
brew install sops age
age-keygen -o /tmp/age.txt
```

Then put each half where it belongs:

- `# public key: age1...` goes in `.sops.yaml` (safe to commit)
- `AGE-SECRET-KEY-1...` goes in Infisical Cloud as `SOPS_AGE_KEY` (homelab project, env `dev`,
  folder `/`), plus a password manager copy
- Delete the local copy: `rm /tmp/age.txt`

**Every rebuild: give the key to the cluster.** First check that the key comes back. If this
prints nothing, the next command creates an empty Secret and decryption fails.

```sh
infisical secrets get SOPS_AGE_KEY --plain --env=dev --path=/   # AGE-SECRET-KEY-1...
```

```sh
infisical secrets get SOPS_AGE_KEY --plain --env=dev --path=/ \
  | kubectl create secret generic sops-age -n flux-system --from-file=age.agekey=/dev/stdin
flux reconcile kustomization infra-configs --with-source
```

## 10. Infisical Operator login

Once `sops-age` exists, Flux decrypts the operator's credentials
(`infrastructure/configs/infisical-universal-auth.sops.yaml`), and the operator syncs every other
secret from Infisical Cloud. On a rebuild there's nothing to do unless that Client Secret was
revoked.

**First time only:** in Infisical Cloud:

1. Create the machine identity `homelab-cluster` (Universal Auth, no org-level access)
2. Add it to the `homelab` project as Viewer
3. Create a Client Secret
4. Build the SOPS file ([secrets](secrets.md#rotate-the-operators-credentials))

**Check:**

```sh
kubectl get infisicalauth -n infisical-operator-system   # homelab-cluster READY True
kubectl get infisicalstaticsecret -A
```

## 11. gVisor on the minis

Agent sandboxes run under gVisor (`RuntimeClass` `gvisor`, [airlock](airlock.md#gvisor)). The
`RuntimeClass` only schedules onto the minis, and the runtime is installed on each VM by hand, not
from git. If a mini's VM is rebuilt, sandboxes can't run on it until you redo this step there.

**a. Install `runsc`** (in each mini's VM, `limactl shell k3s-miniN`):

```sh
curl -fsSL https://gvisor.dev/archive.key | sudo gpg --dearmor -o /usr/share/keyrings/gvisor-archive-keyring.gpg
echo "deb [arch=arm64 signed-by=/usr/share/keyrings/gvisor-archive-keyring.gpg] https://storage.googleapis.com/gvisor/releases release main" | sudo tee /etc/apt/sources.list.d/gvisor.list
sudo apt-get update && sudo apt-get install -y runsc
```

The package installs `runsc` and `containerd-shim-runsc-v1`. k3s doesn't add a containerd runtime
for gVisor on its own, so step b adds it.

**b. Add the runtime to containerd.** On start, k3s renders `config.toml` from this template.
`{{ template "base" . }}` keeps k3s's default config.

```sh
sudo tee /var/lib/rancher/k3s/agent/etc/containerd/config-v3.toml.tmpl <<'EOF'
{{ template "base" . }}

[plugins.'io.containerd.cri.v1.runtime'.containerd.runtimes.'runsc']
  runtime_type = "io.containerd.runsc.v1"
EOF
sudo systemctl restart k3s
sudo grep -A1 "runtimes.'runsc'" /var/lib/rancher/k3s/agent/etc/containerd/config.toml
# prints the runsc runtime and its runtime_type
```

Do one mini at a time: wait for its node to show `Ready` in `kubectl get nodes` before the next,
so etcd keeps quorum.

**c. Check** (laptop, with a claimed sandbox):

```sh
kubectl exec -n agents <pod> -c agent -- dmesg | head -1   # [    0.000000] Starting gVisor...
```
