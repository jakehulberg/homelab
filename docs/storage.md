# Storage

## How data is stored

- **Volumes:** k3s's built-in **local-path** (cluster default). Each volume is a folder on one
  node's disk under `/var/lib/rancher/k3s/storage/`; its pod can only run on that node.
- **Postgres:** **CloudNativePG** (`infrastructure/controllers/cloudnative-pg/`, chart `0.29.1`).
  Each app declares a `Cluster`; the operator runs a primary plus a live copy on a second node and
  fails over if the primary's node dies.
- **Not used:** Longhorn. Replicating every volume is heavy, slow over Wi-Fi plus Tailscale, and
  needs more disk than k3s-mini1 has. Replication happens per database instead.

### Sizes aren't enforced

local-path `storage.size` is a request, not a limit; a volume can fill the node's disk. k3s-mini1
has only 15GiB, so watch databases that land there.

## Databases

| Database | App | Namespace | Instances | Backups |
|---|---|---|---|---|
| `infisical-db` | Infisical | `infisical` | 2 | R2 `s3://homelab-postgres-backups/infisical`, continuous + daily 03:00, 30 days |
| `finance.db`, `tasks.db` (SQLite) | life-dashboard | `life-dashboard` | 1 | Litestream to R2 `life-dashboard/`, SSE-C encrypted, continuous + daily snapshot, 30 days |

The demo Infisical on mini3 (Docker) has its own Postgres outside the cluster.

## Credentials

CloudNativePG generates each database's app user and password in `<cluster-name>-app` (e.g.
`infisical-db-app`), with keys like `username`, `password`, `uri`. Never in git or Infisical: a
rebuild generates new ones and the app reads them from the Secret.

`<cluster-name>-ca`, `-server` and `-replication` are the operator's own TLS for connections and
replication.

## Backups

A second instance only covers a dead node. A dropped database or bad prune hits the copy too, so
every database also backs up off-site to Cloudflare R2.

### How

- **Barman Cloud plugin** (`infrastructure/controllers/cloudnative-pg/barman-cloud.yaml`, chart
  `0.8.1`) adds a helper container to each Postgres pod (hence `2/2`). CloudNativePG's built-in
  backups are deprecated since 1.26 in favor of it.
- **Continuous:** every write-ahead log file (Postgres's change record) ships to R2 within minutes.
- **Daily:** full base backup at 03:00 (`ScheduledBackup`).
- **Restore** replays a base backup plus later logs: any minute in the last 30 days.

### R2

- Bucket `homelab-postgres-backups`, Standard class (Infrequent Access charges for reads, which the
  plugin does regularly). One folder per app.
- Access: R2 Account API token `homelab cnpg backups`, Object Read & Write on that bucket only, no
  expiry. S3 keys in Infisical Cloud, homelab project, env `dev`, folder `/r2` (`R2_ACCESS_KEY_ID`,
  `R2_SECRET_ACCESS_KEY`).
- Endpoint: `https://e8dc2af5d31bf4e016b6fab1e2515a27.r2.cloudflarestorage.com`.
- Cost: within R2's free tier (10GB) at these sizes.

### Per database (`apps/<app>/`)

| File | What |
|---|---|
| `r2-sync.yaml` | `InfisicalStaticSecret` syncing `/r2` into `r2-credentials-infisical` in the app's namespace (Secrets can't cross namespaces) |
| `objectstore.yaml` | `ObjectStore` `r2`: bucket folder, endpoint, credentials (`R2_ACCESS_KEY_ID`, `R2_SECRET_ACCESS_KEY`), `retentionPolicy: "30d"` |
| `postgres.yaml` | `Cluster` gets `plugins: barman-cloud.cloudnative-pg.io` with `isWALArchiver: true`, `barmanObjectName: r2` |
| `scheduled-backup.yaml` | `ScheduledBackup`, `"0 0 3 * * *"` (first field is seconds) |

For a new database, copy `apps/infisical/r2-sync.yaml` and change the namespace; the keys stay in
one place, `/r2` ([secrets](secrets.md#give-an-app-secrets)).

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

Restoring creates a **new** Cluster. Sketch, per the plugin docs:

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

Infisical also needs its original `ENCRYPTION_KEY` and `AUTH_SECRET` (Infisical Cloud) to read it.
A restore drill is on the to-do list.

## SQLite backups: Litestream (life-dashboard)

SQLite on a 2GB local-path volume (`/data`), one pod, `Recreate`. A Litestream sidecar streams
every change to R2 within about a second.

| Piece | Where |
|---|---|
| Litestream config (`litestream.yml`) | ConfigMap `litestream` in `apps/life-dashboard/litestream.yaml` |
| R2 keys + SSE-C key | Synced from Infisical `/r2` and `/litestream` into `litestream-infisical` |
| Sidecar + restore steps | `apps/life-dashboard/life-dashboard.yaml` (`initContainers`) |

- **Encryption:** SSE-C. Litestream sends our 256-bit key (`LITESTREAM_SSE_C_KEY`, Infisical
  `/litestream`) with each request; R2 encrypts with it and never stores it. **Lose the key, lose
  the backups.** Litestream 0.5 has no age support.
- **Snapshots:** daily full copy, kept 30 days; restore to any point in that window.
- **Requires WAL mode:** the app sets `journal_mode = WAL` and `busy_timeout = 5000`.
- **Automatic restore:** two init containers run `litestream restore -if-db-not-exists
  -if-replica-exists` before the app, refilling an empty volume (new node, rebuilt cluster) from R2.
  No-op when data exists.
- **Sidecar:** an init container with `restartPolicy: Always`, so it starts before the app and runs
  alongside.

```sh
kubectl get pods -n life-dashboard                                          # 2/2 Running
kubectl logs -n life-dashboard deploy/life-dashboard -c litestream --tail=20 # "snapshot complete", no errors
```

Loading data by hand (done once from mini1): suspend `apps`, scale to 0, mount the volume in a
busybox pod as user 1000, `kubectl cp` the `.db` files in, delete `-wal`/`-shm` files, scale to 1,
resume.

## Restarts and failover

- `primaryUpdateMethod: switchover` on each Cluster: on a spec or image change, CloudNativePG
  promotes the copy first, then restarts the old primary. The default, `restart`, takes the database
  down until the primary is back.
- A primary that loses the Kubernetes API or the other instance for a while shuts itself down so two
  primaries never write at once; the copy takes over.
- Enabling WAL archiving on a running cluster: the old primary can't archive without the plugin's
  helper container, so shutdown can hang up to 30 minutes. Force-deleting is safe (Postgres
  recovers from abrupt stops; unarchived logs stay on disk):
  `kubectl delete pod <pod> -n <namespace> --grace-period=0 --force`.

## Check it

```sh
kubectl get cluster -A                          # STATUS "Cluster in healthy state", READY 2/2
kubectl get pods -n infisical -o wide           # instances on different nodes
kubectl get pvc -A                              # every volume and its size
kubectl get pv -o 'custom-columns=CLAIM:.spec.claimRef.name,NODE:.spec.nodeAffinity.required.nodeSelectorTerms[0].matchExpressions[0].values[0],PATH:.spec.local.path'
```

The last command shows which node and folder each volume lives in.
