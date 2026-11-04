# 2026-10-25: Infisical database failover and Redis timeouts

**Impact:** Infisical couldn't write to its database for ~10 minutes, and a restarted Infisical pod
took minutes to become ready. No data lost.

## What happened

1. Enabling WAL archiving (backups to R2) made CloudNativePG restart the primary (`infisical-db-1`)
   in place, because the default `primaryUpdateMethod` is `restart`
2. On shutdown, the old primary started archiving its last WAL files through the Barman Cloud
   plugin, but the pod had no plugin helper container (`wal archive plugin is not available`). The
   shutdown hung, with a grace period of up to 30 minutes
3. Jake force-deleted the pod, and CloudNativePG failed over to `infisical-db-2` as designed
4. The new primary briefly lost contact with the Kubernetes API and `infisical-db-1`, so its
   liveness check shut the new primary down on purpose: CloudNativePG stops a primary that might
   be partitioned, so the primary can't accept writes. The cluster failed back to
   `infisical-db-1`, which recovered
5. Infisical then kept logging `Redis connection error [client=bootstrap] ... ETIMEDOUT`. In-cluster
   DNS lookups took 0.3 to 5 seconds, because pods inherited the nodes' Tailscale and router search
   domains and tried those domains first

## Root causes

- The default `restart` update method, combined with enabling archiving on a running cluster
- k3s-mini2 and k3s-mini3 reached the tailnet only through Tailscale's relay, which added latency
  and short drops between nodes
- Pods inherited the nodes' search domains, which made DNS slow

## Fixes

- `primaryUpdateMethod: switchover`: CloudNativePG promotes the copy before it restarts the
  primary ([storage](../storage.md#restarts-and-failover))
- k3s `resolv-conf` points at a node file with only `1.1.1.1` and `8.8.8.8`
  ([networking](../networking.md#dns-inside-the-cluster))
- Infisical's `REDIS_URL` uses the short Service name `redis`
  ([infisical](../infisical.md#how-it-fits-together))
- Lima VMs run on `vzNAT`, so Tailscale connects nodes directly instead of relaying
  ([networking](../networking.md#lima-vm-networking-vznat))

## Lessons

- If you enable archiving on a running cluster, the old primary's shutdown can hang;
  force-deleting the pod is safe
- `kubectl get cluster <name> -n <ns>` and the database pod logs explain most CloudNativePG
  behavior: look for "Failing over", "promoted" and "liveness check failed"
