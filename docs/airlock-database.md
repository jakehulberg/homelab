# Airlock database and recovery

Airlock uses its own PostgreSQL cluster for durable agent work. This keeps execution history out
of the Infisical database and gives queued work a recovery path after orchestrator restarts.

## Prerequisites

- `flux get kustomizations` shows the infrastructure controllers Ready
- The CloudNativePG and Barman plugin HelmReleases in `cnpg-system` are Ready
- The existing Infisical `/r2` credentials permit the `airlock` prefix in
  `s3://homelab-postgres-backups`; verify this outside Git before release
- Nodes have capacity for two 20 GiB database volumes and the configured memory limits

## What's where

| Path or resource | Purpose |
|---|---|
| `apps/airlock/postgres.yaml` | Cluster, ObjectStore, ScheduledBackup, R2 secret synchronization, database ingress |
| `apps/airlock/orchestrator.yaml` | CNPG application URI, CA mount, readiness, liveness, scoped API ingress |
| `apps/airlock/monitoring.yaml` | Orchestrator and database PodMonitors |
| `airlock-db-app` | CNPG-generated application password and URI; owner/database `airlock` |
| `airlock-db-ca` | CNPG certificate authority mounted read-only in the orchestrator |
| `airlock-r2-credentials` | Existing `/r2` access and secret keys synchronized by Infisical |
| `airlock-r2` | ObjectStore using a dedicated `airlock` backup prefix with 30-day retention |

```
Infisical /r2 ──1. secret sync──▶ airlock-r2-credentials
                                      │
airlock-orchestrator ──2. TLS──▶ airlock-db-rw:5432 ──3. Barman──▶ R2 /airlock
```

1. Infisical synchronizes the existing R2 keys into the `airlock` namespace
2. The orchestrator uses the CNPG application URI and validates the server with `airlock-db-ca`
3. Barman archives WAL and runs the daily base backup at 03:30 UTC

## Release order

1. Review the matching Airlock and homelab PRs together. Neither branch deploys itself
2. Introduce the database, R2 Secret, ObjectStore and backup resources through Flux first. Wait for
   `airlock-db` readiness and validate a backup before depending on durable history
3. Coordinate the application configuration with the new images. The new orchestrator requires
   `AIRLOCK_DATABASE_URL`; the old image does not serve `/readyz`. Merging either full PR alone
   can leave a replacement pod unready until the matching change arrives. Preserve the healthy
   existing pod during the rollout and don't scale it down manually
4. Confirm the migration completes, `/readyz` succeeds, and the existing sandbox flow works
5. Create a small approved task and run the research workflow before enabling recurring schedules

Migrations are embedded in the Airlock binary and execute transactionally under a PostgreSQL
advisory lock. They create only Airlock-owned tables. Concurrent starts serialize migration work;
checksum mismatches or newer schemas stop startup. Never edit an applied migration.

## Checking it

```sh
kubectl -n airlock get cluster airlock-db # Ready, two instances
kubectl -n airlock get secret airlock-db-app airlock-db-ca airlock-r2-credentials # All present
kubectl -n airlock get scheduledbackup airlock-db-daily # Daily plugin schedule
kubectl -n airlock get backups -o wide # A completed base backup, not only a schedule
kubectl -n airlock describe cluster airlock-db # ContinuousArchiving succeeds
kubectl -n airlock logs deployment/airlock-orchestrator --tail=100 # No migration or database errors
kubectl -n airlock get podmonitor # airlock-orchestrator and airlock-db
```

A manifest referencing Barman is not evidence of a working backup. Inspect the Backup resource's
phase, completion time and error, verify WAL archiving on the Cluster, and perform the restore
drill below. Record the tested backup, recovery target and recovered task IDs before relying on
this database. These checks have not been run against the live cluster by this implementation.

## Restore drill

1. Keep the source cluster and backup prefix intact. Select a completed base backup and recovery
   target with available WAL. Add a new Cluster to a temporary, reviewed Flux path, with a new
   name such as `airlock-db-restore`; don't replace `airlock-db` in place
2. Use the bootstrap and external-cluster settings below instead of `bootstrap.initdb`. Keep
   storage at least 20 GiB, set the same application resources, and allow the restoration pods
   through a matching database policy. Don't reuse the original cluster name as the new name
3. Wait for recovery to complete, connect through the recovered cluster's application Secret,
   and inspect `airlock_migrations`, task counts, attempts, outputs, and schedule fire history
4. Keep automation stopped during verification. A restored queue can replay external work that
   happened after the recovery point. Reconcile recorded sandbox claims and external side effects
   before permitting task retries or enabling schedules
5. For a real cutover, update the orchestrator's application URI and CA references through Flux
   only after recovery checks pass. Keep the previous cluster for rollback until the recovered
   application is verified. Give restored production backups a new, empty destination prefix

```yaml
bootstrap:
  recovery:
    source: airlock-backup
externalClusters:
  - name: airlock-backup
    plugin:
      name: barman-cloud.cloudnative-pg.io
      parameters:
        barmanObjectName: airlock-r2
        serverName: airlock-db
```

The ObjectStore is namespace-scoped. Restore into `airlock`, or provide the same ObjectStore and
credential reference in a separate restore namespace. Confirm the installed Barman plugin supports
the restore fields before submitting the drill through Flux. See the
[upstream recovery procedure](https://cloudnative-pg.io/plugin-barman-cloud/docs/usage/#restoring-a-cluster).

## Rollback and task recovery

Roll back application images through the existing Flux workflow; never drop tables to roll back
code. Returning to the old in-memory orchestrator leaves durable operations unavailable and requires
restoring its `/healthz` readiness path. Retain the database and backups. A later incompatible
migration needs a reviewed forward fix or restore; application rollback alone cannot undo data
changes.

If an exec connection or worker lease is lost, the task enters `uncertain`. The orchestrator deletes
the recorded claim and waits for the recorded pod UID to disappear before resolving the attempt.
It doesn't automatically replay uncertain work. Review external effects, then use **Retry** if
appropriate. Interrupted interactive sessions remain locked until their sandbox is released.

## Troubleshooting

- **Orchestrator unready:** verify `airlock-db` readiness, the CNPG URI Secret, the CA mount, and
  database ingress from `app: airlock-orchestrator`
- **Backup remains pending or fails:** inspect `airlock-r2-credentials`, the R2 prefix permission,
  Cluster archiving condition, and Barman plugin logs. Don't count it as a valid backup
- **Agent coordination fails:** verify the Claude pod communication label, namespace and pod
  selectors on both policies, and use `curl --noproxy` for the internal orchestrator hostname
- **Tasks remain uncertain:** inspect the attempt's claim and pod UID. Check Agent Sandbox cleanup
  and pod termination before retrying; deleting a database lease isn't process termination
- **Kubernetes health workflow lacks access:** provide a separately reviewed read-only integration
  through a restricted pool or Agent Vault. This change grants agents no Kubernetes API credentials
