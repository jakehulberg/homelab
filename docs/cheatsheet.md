# Cheatsheet

Most used: `flux get all -A` and `kubectl get pods -A`.

## Lima (on the minis)

```sh
limactl list                  # VMs and status
limactl shell k3s-mini1       # shell into a VM
limactl start k3s-mini1
limactl stop k3s-mini1
```

## kubectl

```sh
kubectl config get-contexts           # list clusters (* = current)
kubectl config use-context homelab    # switch cluster
kubectl get nodes
kubectl get pods -A                   # every pod, every namespace
kubectl get pods -n <ns>
kubectl logs -n <ns> <pod>
kubectl describe pod -n <ns> <pod>    # events: why a pod won't start
kubectl port-forward -n <ns> svc/<name> <port>:<port>
```

## Flux

```sh
flux get all -A                                          # everything Flux manages, and Ready status
flux get helmreleases -A
flux reconcile kustomization flux-system --with-source   # apply the latest push now
flux logs                                                # controller logs, when something isn't Ready
```

Pasting commands with `#` comments into zsh needs `setopt interactive_comments` in `~/.zshrc`;
otherwise copy without the comments.
