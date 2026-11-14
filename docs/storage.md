# Storage

Volumes use k3s's built-in local-path storage, Postgres databases run on CloudNativePG, and every
database backs up off-site to Cloudflare R2. Infisical, Authentik and life-dashboard keep their
data here.

## How data is stored

- **Volumes:** local-path, the cluster's default StorageClass. Each volume is a folder on one
  node's disk under `/var/lib/rancher/k3s/storage/`, so its pod can only run on that node
- **Sizes:** local-path treats `storage.size` as a request, not a limit, so a volume can fill the
  node's disk. Watch databases that land on k3s-mini1: its VM has the smallest disk (`disk: 30GiB`
  in `lima/k3s-mini1.yaml`)
- **Postgres:** CloudNativePG (`infrastructure/controllers/cloudnative-pg/`, chart `0.29.1`). Each
  app declares a `Cluster`. With `instances: 2`, the operator runs a primary plus a live copy on a
  second node and fails over if the primary's node dies
- **Not used:** Longhorn. Replicating every volume is heavy, adds constant traffic over Tailscale, and
  needs more disk than k3s-mini1 has. Replication happens per database instead

## Databases

| Database | App | Namespace | Instances | Backups |
|---|---|---|---|---|
| `infisical-db` | Infisical | `infisical` | 2 | R2 `s3://homelab-postgres-backups/infisical`, continuous + daily 03:00, 30 days |
| `authentik-db` | Authentik | `authentik` | 1 | R2 `s3://homelab-postgres-backups/authentik`, continuous + daily 03:00, 30 days |
| `finance.db`, `tasks.db` (SQLite) | life-dashboard | `life-dashboard` | 1 | Litestream to R2 `s3://homelab-postgres-backups/life-dashboard`, SSE-C encrypted, continuous + daily snapshot, 30 days |

The demo Infisical on mini3 (Docker) has its own Postgres outside the cluster.

## Credentials

CloudNativePG generates each database's app user and password in the Secret `<cluster-name>-app`
(e.g. `infisical-db-app`), with keys like `username`, `password` and `uri`. They're never in git
or Infisical: a rebuild generates new ones, and the app reads them from the Secret.

`<cluster-name>-ca`, `-server` and `-replication` hold the operator's own TLS certificates for
connections and replication.

## Backups

A second instance only covers a dead node. A dropped database or a bad prune reaches the copy
too, so every database also backs up off-site to Cloudflare R2.

### How

- **Barman Cloud plugin:** `infrastructure/controllers/cloudnative-pg/barman-cloud.yaml`, chart
  `0.8.1`. The plugin adds a helper container to each Postgres pod, so the pods show `2/2`.
  CloudNativePG deprecated its built-in backups in 1.26 in favor of the plugin
- **Continuous:** the plugin ships every write-ahead log file (Postgres's change record) to R2
  within minutes
- **Daily:** a `ScheduledBackup` takes a full base backup at 03:00
- **Restore:** a restore replays a base backup plus the later logs, to any minute in the last 30
  days

### R2

- **Bucket:** `homelab-postgres-backups`, Standard class (Infrequent Access charges for reads,
  which the plugin does regularly). One folder per app
- **Access:** R2 Account API token `homelab cnpg backups`, Object Read & Write on that bucket
  only, no expiry. Its S3 keys are in Infisical Cloud `/r2` (`R2_ACCESS_KEY_ID`,
  `R2_SECRET_ACCESS_KEY`)
- **Endpoint:** `https://e8dc2af5d31bf4e016b6fab1e2515a27.r2.cloudflarestorage.com`
- **Cost:** within R2's free tier (10GB) at these sizes

### Per database (`apps/<app>/`)

| File | What |
|---|---|
| `r2-sync.yaml` | `InfisicalStaticSecret` that syncs `/r2` into a Secret in the app's namespace (`r2-credentials-infisical`, `r2-credentials-authentik`), because Secrets can't cross namespaces |
| `objectstore.yaml` | `ObjectStore` `r2`: bucket folder, endpoint, the R2 Secret's keys, `retentionPolicy: "30d"` |
| `postgres.yaml` | The `Cluster`'s `plugins` entry `barman-cloud.cloudnative-pg.io`, with `isWALArchiver: true` and `barmanObjectName: r2` |
| `scheduled-backup.yaml` | `ScheduledBackup`, `"0 0 3 * * *"` (the first field is seconds) |

To back up a new database:

1. Copy `r2-sync.yaml`, `objectstore.yaml` and `scheduled-backup.yaml` from `apps/infisical/`
2. In the copies, change the namespace, the Secret name, the bucket folder and the cluster name
3. Add the `plugins` entry to the app's `Cluster`

The keys stay in one place, `/r2` ([secrets](secrets.md#give-an-app-secrets)).

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

A restore creates a new Cluster. A sketch, per the plugin docs:

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

A restored Infisical database is unreadable without the original `ENCRYPTION_KEY` and
`AUTH_SECRET` ([secrets](secrets.md#infisicals-own-keys)).

## SQLite backups: Litestream (life-dashboard)

life-dashboard keeps SQLite on a 2Gi local-path volume (`/data`), with one pod and the `Recreate`
strategy. A Litestream sidecar streams every change to R2 within about a second.

| Piece | Where |
|---|---|
| Litestream config (`litestream.yml`) | ConfigMap `litestream` in `apps/life-dashboard/litestream.yaml` |
| R2 keys and SSE-C key | Synced from Infisical `/r2` and `/litestream` into `litestream-infisical`, same file |
| Sidecar and restore init containers | `apps/life-dashboard/life-dashboard.yaml` (`initContainers`) |

- **Encryption:** SSE-C. Litestream sends the 256-bit key `LITESTREAM_SSE_C_KEY` (Infisical
  `/litestream`) with each request; R2 encrypts with it and never stores it. **If the key is lost,
  the backups are lost.** Litestream 0.5 has no age support
- **Snapshots:** a full copy daily, kept 30 days, so a restore can reach any point in that window
- **WAL mode:** Litestream requires it. The app sets `journal_mode = WAL` and `busy_timeout = 5000`
- **Automatic restore:** before the app starts, two init containers run `litestream restore
  -if-db-not-exists -if-replica-exists`. On an empty volume (new node, rebuilt cluster) they
  refill the databases from R2; if the data exists, they do nothing
- **Sidecar:** an init container with `restartPolicy: Always`, so Litestream starts before the
  app and runs alongside it

```sh
kubectl get pods -n life-dashboard                                          # 2/2 Running
kubectl logs -n life-dashboard deploy/life-dashboard -c litestream --tail=20 # "snapshot complete", no errors
```

To load data by hand (done once from mini1):

1. Suspend the `apps` Kustomization and scale `life-dashboard` to 0
2. Mount the volume in a busybox pod as user 1000 and `kubectl cp` the `.db` files in
3. Delete the `-wal` and `-shm` files
4. Scale `life-dashboard` to 1 and resume `apps`

## Restarts and failover

- **Spec or image change:** with `primaryUpdateMethod: switchover` on each Cluster, CloudNativePG
  promotes the copy first, then restarts the old primary. The default, `restart`, takes the
  database down until the primary is back
- **Lost contact:** if a primary loses the Kubernetes API or the other instance for a while, the
  primary shuts itself down so two primaries never write at once, and the copy takes over

## Check it

```sh
kubectl get cluster -A                          # STATUS "Cluster in healthy state", READY = INSTANCES
kubectl get pods -n infisical -o wide           # instances on different nodes
kubectl get pvc -A                              # every volume and its size
kubectl get pv -o 'custom-columns=CLAIM:.spec.claimRef.name,NODE:.spec.nodeAffinity.required.nodeSelectorTerms[0].matchExpressions[0].values[0],PATH:.spec.local.path'   # node and folder of each volume
```

## Troubleshooting

- **Old primary stuck terminating for up to 30 minutes after WAL archiving is turned on:** the
  old primary can't archive without the plugin's helper container. Force-deleting the pod is safe
  (Postgres recovers from abrupt stops, and unarchived logs stay on disk):
  `kubectl delete pod <pod> -n <namespace> --grace-period=0 --force`
