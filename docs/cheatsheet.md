# Cheatsheet

The two you'll use most: `flux get all -A` and `kubectl get pods -A`.

## Lima (on the minis)

```sh
limactl list                  # VMs and their status
limactl shell k3s-mini1       # shell into a VM
limactl start k3s-mini1
limactl stop k3s-mini1
```

## kubectl

```sh
kubectl config get-contexts           # list clusters (* = current)
kubectl config use-context homelab    # switch cluster
kubectl get nodes
kubectl get pods -A                   # every pod in every namespace
kubectl get pods -n <ns>
kubectl logs -n <ns> <pod>
kubectl describe pod -n <ns> <pod>    # events: why a pod won't start
kubectl port-forward -n <ns> svc/<name> <port>:<port>
```

## Flux

```sh
flux get all -A                                          # everything Flux manages, and whether it's Ready
flux get helmreleases -A
flux reconcile kustomization flux-system --with-source   # apply the latest push now
flux logs                                                # controller logs, for when something isn't Ready
```

zsh doesn't treat `#` as a comment in pasted commands unless `setopt interactive_comments` is in
`~/.zshrc`. Without it, copy the commands without the comments.
