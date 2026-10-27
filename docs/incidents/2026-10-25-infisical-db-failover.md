# 2026-10-25: Infisical database failover and Redis timeouts

**Impact:** Infisical couldn't write to its database for ~10 minutes, and a restarted Infisical pod
took minutes to become ready. No data lost.

## What happened

1. Enabling WAL archiving (backups to R2) made CloudNativePG restart the primary (`infisical-db-1`)
   in place, per the default `primaryUpdateMethod: restart`.
2. On shutdown, the old primary tried to archive its last logs via the Barman Cloud plugin, but the
   pod had no plugin helper container (`wal archive plugin is not available`). Shutdown hung, with
   up to a 30-minute grace period.
3. We force-deleted the pod; CloudNativePG failed over to `infisical-db-2` as designed.
4. The new primary briefly lost contact with the Kubernetes API and `infisical-db-1`, so its
   liveness check deliberately shut it down (it won't write if it might be partitioned). It failed
   back to `infisical-db-1` and recovered.
5. Infisical then kept logging `Redis connection error [client=bootstrap] ... ETIMEDOUT`. In-cluster
   DNS lookups took 0.3 to 5 seconds because pods inherited the nodes' Tailscale and router search
   domains and queried those first.

## Root causes

- Default `restart` update method, combined with enabling archiving on a running cluster.
- k3s-mini2 and k3s-mini3 reached the tailnet only via Tailscale's relay, adding latency and short
  drops between nodes.
- Pods inherited node search domains, making DNS slow.

## Fixes

- `primaryUpdateMethod: switchover`: promote the copy before restarting the primary.
- k3s `resolv-conf` points at a node file with only `1.1.1.1` and `8.8.8.8`
  ([networking](../networking.md#dns-inside-the-cluster)).
- Infisical's `REDIS_URL` uses the short Service name `redis`.
- Lima VMs on `vzNAT` so Tailscale connects nodes directly instead of relaying
  ([networking](../networking.md#lima-vm-networking-vznat)).

## Lessons

- Enabling archiving on a running cluster can hang the old primary's shutdown; force-deleting it is
  safe.
- `kubectl get cluster <name> -n <ns>` and the database pod logs explain most CloudNativePG
  behavior: look for "Failing over", "promoted", "liveness check failed".
