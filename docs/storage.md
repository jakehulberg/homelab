# Storage

## How data is stored

- **Volumes:** k3s's built-in **local-path** storage (the cluster default). Each volume is a plain
  folder on one node's disk, under `/var/lib/rancher/k3s/storage/`. A pod using it can only run on
  that node.
- **Postgres:** **CloudNativePG** (`infrastructure/controllers/cloudnative-pg/`, chart `0.29.1`).
  Each app declares a `Cluster` next to it; the operator runs a primary plus a live copy on a
  second node, and fails over if the primary's node dies.
- **Not used:** Longhorn. It replicates every volume across nodes, which is heavy, slow over Wi-Fi
  plus Tailscale, and needs more disk than k3s-mini1 has. Replication happens per database
  (CloudNativePG) instead.

### Sizes aren't enforced

`storage.size` on local-path is a request, not a limit. A volume can grow until the node's disk is
full. k3s-mini1's VM disk is only 15GiB, so keep an eye on databases that land there.

## Databases

| Database | App | Namespace | Instances | Backups |
|---|---|---|---|---|
| `infisical-db` | Infisical | `infisical` | 2 | None yet (planned: Cloudflare R2) |

The demo Infisical on mini3 (Docker) has its own Postgres outside the cluster and isn't covered
here.

## Credentials

CloudNativePG generates each database's app user and password and stores them in
`<cluster-name>-app` (e.g. `infisical-db-app`), with keys like `username`, `password` and `uri`.
They never go in git or Infisical: on a rebuild the operator makes new ones and the app reads them
from the Secret.

The `<cluster-name>-ca`, `-server` and `-replication` Secrets are the operator's own TLS for
encrypted connections and replication.

## Backups

Not set up yet. A second instance only protects against a node dying; a deleted database or a bad
prune takes the copy with it. Plan: CloudNativePG continuous backups (nightly base backup plus
write-ahead logs, so restore to any minute) to Cloudflare R2, set up before any app with data worth
keeping goes in.

## Check it

```sh
kubectl get cluster -A                          # STATUS "Cluster in healthy state", READY 2/2
kubectl get pods -n infisical -o wide           # instances on different nodes
kubectl get pvc -A                              # every volume and its size
kubectl get pv -o 'custom-columns=CLAIM:.spec.claimRef.name,NODE:.spec.nodeAffinity.required.nodeSelectorTerms[0].matchExpressions[0].values[0],PATH:.spec.local.path'
```

The last command shows which node and folder each volume physically lives in.
