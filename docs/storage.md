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
| `infisical-db` | Infisical | `infisical` | 2 | R2 `s3://homelab-postgres-backups/infisical`, continuous + daily 03:00, 30 days |

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

A second instance only protects against a node dying. A deleted database or a bad prune takes the
copy with it, so every database also backs up off-site to Cloudflare R2.

### How

- **Barman Cloud plugin** (`infrastructure/controllers/cloudnative-pg/barman-cloud.yaml`, chart
  `0.8.1`) does the copying. It adds a helper container to each Postgres pod, which is why database
  pods show `2/2`. CloudNativePG's built-in backup support is deprecated since 1.26 in favor of this
  plugin.
- **Continuous:** every write-ahead log file (Postgres's record of each change) is shipped to R2
  within minutes.
- **Daily:** a full base backup at 03:00 (`ScheduledBackup`).
- **Restore** replays a base backup plus the logs after it, so any minute in the last 30 days.

### R2

- Bucket `homelab-postgres-backups`, Standard storage class (Infrequent Access charges for reads,
  which the plugin does regularly). One folder per app.
- Access: an R2 Account API token named `homelab cnpg backups`, Object Read & Write on that bucket
  only, no expiry. Its S3 keys are in Infisical Cloud, homelab project, env `dev`, folder `/r2`
  (`R2_ACCESS_KEY_ID`, `R2_SECRET_ACCESS_KEY`).
- Endpoint: `https://e8dc2af5d31bf4e016b6fab1e2515a27.r2.cloudflarestorage.com`.
- Cost: inside R2's free tier (10GB) for databases this size.

### Per database (`apps/<app>/`)

| File | What |
|---|---|
| `r2-credentials.sops.yaml` | The R2 keys as a Secret in the app's namespace (Secrets can't be shared across namespaces) |
| `objectstore.yaml` | `ObjectStore` named `r2`: bucket folder, endpoint, credentials, `retentionPolicy: "30d"` |
| `postgres.yaml` | The `Cluster` gets `plugins: barman-cloud.cloudnative-pg.io` with `isWALArchiver: true` and `barmanObjectName: r2` |
| `scheduled-backup.yaml` | `ScheduledBackup`, schedule `"0 0 3 * * *"` (this format starts with a seconds field) |

Build the credentials file from the repo root:

```sh
infisical run --env=dev --path=/r2 -- sh -c '
kubectl create secret generic r2-credentials -n <namespace> \
  --from-literal=ACCESS_KEY_ID="$R2_ACCESS_KEY_ID" \
  --from-literal=ACCESS_SECRET_KEY="$R2_SECRET_ACCESS_KEY" \
  --dry-run=client -o yaml' > apps/<app>/r2-credentials.sops.yaml
sops -e -i apps/<app>/r2-credentials.sops.yaml
```

### Check it

```sh
kubectl get cluster infisical-db -n infisical \
  -o jsonpath='{range .status.conditions[*]}{.type}={.status}{"\n"}{end}'   # ContinuousArchiving=True
kubectl get backup -n infisical                                             # PHASE completed
```

### Back up now

```sh
cat <<'YAML' | kubectl apply -f -
apiVersion: postgresql.cnpg.io/v1
kind: Backup
metadata:
  name: infisical-db-manual-20261025   # any unique name
  namespace: infisical
spec:
  cluster:
    name: infisical-db
  method: plugin
  pluginConfiguration:
    name: barman-cloud.cloudnative-pg.io
YAML
```

### Restore (not yet tested)

Restoring creates a **new** Cluster from the backups. Sketch, per the plugin's docs:

```yaml
spec:
  bootstrap:
    recovery:
      source: origin
  externalClusters:
    - name: origin
      plugin:
        name: barman-cloud.cloudnative-pg.io
        parameters:
          barmanObjectName: r2
          serverName: infisical-db   # the original cluster's name, its folder in the bucket
```

Infisical also needs its original `ENCRYPTION_KEY` and `AUTH_SECRET` (in Infisical Cloud) to read
a restored database. A restore drill belongs on the to-do list.

## Restarts and failover

- `primaryUpdateMethod: switchover` on each Cluster: when the primary needs restarting (a spec or
  image change), CloudNativePG promotes the copy first, then restarts the old primary. The default,
  `restart`, restarts the primary in place, so the database is down until it's back.
- If the primary can't reach the Kubernetes API or the other instance for a while, it shuts itself
  down on purpose so two primaries never write at once, and the copy takes over.
- Turning on WAL archiving for a running cluster: the old primary can't archive without the plugin's
  helper container, so its shutdown can hang for up to 30 minutes. Force-deleting that pod is safe
  (Postgres recovers from abrupt stops, and the unarchived logs stay on disk):
  `kubectl delete pod <pod> -n <namespace> --grace-period=0 --force`.

## Check it

```sh
kubectl get cluster -A                          # STATUS "Cluster in healthy state", READY 2/2
kubectl get pods -n infisical -o wide           # instances on different nodes
kubectl get pvc -A                              # every volume and its size
kubectl get pv -o 'custom-columns=CLAIM:.spec.claimRef.name,NODE:.spec.nodeAffinity.required.nodeSelectorTerms[0].matchExpressions[0].values[0],PATH:.spec.local.path'
```

The last command shows which node and folder each volume physically lives in.
