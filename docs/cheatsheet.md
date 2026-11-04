# Cheatsheet

Most used: `flux get all -A` and `kubectl get pods -A`.

## Is everything healthy?

```sh
kubectl get nodes                          # all Ready
flux get kustomizations                    # all Ready True
flux get helmreleases -A                   # all Ready True
kubectl get pods -A | grep -v -E 'Running|Completed'   # anything stuck
kubectl get cluster -A                     # databases: "Cluster in healthy state"
kubectl get certificate -A                 # READY True
kubectl get infisicalstaticsecret -A       # secret syncs
```

## Lima (on the minis)

```sh
limactl list                  # VMs and status
limactl shell k3s-mini1       # shell into a VM
limactl start k3s-mini1
limactl stop k3s-mini1
```

Resizing a VM: [nodes](nodes.md#taking-a-node-down).

## kubectl

```sh
kubectl config get-contexts           # list clusters (* = current)
kubectl config use-context homelab    # switch cluster
kubectl get nodes
kubectl get pods -A                   # every pod, every namespace
kubectl get pods -n <ns>
kubectl logs -n <ns> <pod>
kubectl logs -n <ns> deploy/<name> -f # follow a Deployment's logs
kubectl logs -n <ns> <pod> -c <container> --previous   # the crash before the restart
kubectl describe pod -n <ns> <pod>    # events: why a pod won't start
kubectl get events -n <ns> --sort-by=.lastTimestamp
kubectl rollout restart -n <ns> deploy/<name>
kubectl port-forward -n <ns> svc/<name> <port>:<port>
```

## Flux

```sh
flux get all -A                                          # everything Flux manages, and Ready status
flux get helmreleases -A
flux reconcile kustomization flux-system --with-source   # apply the latest push now
flux reconcile helmrelease <name> -n <ns>                # retry one chart
flux suspend kustomization apps                          # stop Flux undoing a manual change
flux resume kustomization apps                           # hand control back
flux logs                                                # controller logs, when something isn't Ready
```

To change the cluster, commit and push here; Flux applies it within a minute
([gitops](gitops.md)). Anything changed by hand with `kubectl` is reverted unless suspended.

## Helm

```sh
helm list -A                              # installed charts and versions
helm get values <release> -n <ns>         # values Flux passed in
helm get manifest <release> -n <ns>       # everything the chart created
```

## Secrets

```sh
kubectl get infisicalstaticsecret -A                              # one per synced folder
kubectl describe infisicalstaticsecret <name> -n <ns>             # why a sync failed
kubectl get infisicalauth -n infisical-operator-system            # operator login READY True
infisical run --env=dev --path=/ -- sops -d <file>.sops.yaml      # read a SOPS file
```

Secrets live in Infisical Cloud; a change there lands within 60 seconds
([secrets](secrets.md)).

## Certificates

```sh
kubectl get clusterissuer                                # letsencrypt READY True
kubectl describe certificate home-wildcard -n traefik    # Not After, Renewal Time
kubectl get certificate,order,challenge -n traefik       # issuance in progress
kubectl delete challenge -n traefik --all                # retry now
```

More in [certificates](certificates.md#troubleshooting).

## Databases (CloudNativePG)

```sh
kubectl get cluster -A                    # STATUS, READY 2/2, PRIMARY
kubectl get backup -A                     # PHASE completed
kubectl get pvc -A                        # volumes and sizes
```

Backups and failover: [storage](storage.md).

## Nodes

```sh
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data   # before maintenance
kubectl uncordon <node>                                           # after
```

Promote a database copy off the node first ([nodes](nodes.md#taking-a-node-down)).

## Agent sandboxes

```sh
kubectl get sandboxwarmpool,sandboxclaim,sandbox,pods -n agents
kubectl logs -n airlock deploy/airlock-orchestrator     # claims and tasks
kubectl logs -n agents <pod> -c session                 # Agent Vault login and session
kubectl logs -n agent-vault deploy/agent-vault-proxy
```

See [airlock](airlock.md) and [agent vault](agent-vault.md).

## zsh

Pasting commands with `#` comments into zsh needs `setopt interactive_comments` in `~/.zshrc`;
otherwise copy without the comments.
