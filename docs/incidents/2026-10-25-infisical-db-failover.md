# 2026-10-25: Infisical database failover and Redis timeouts

**Impact:** Infisical couldn't write to its database for about 10 minutes, and a restarted
Infisical pod took minutes to become ready. No data lost.

## What happened

1. Turning on WAL archiving (backups to R2) made CloudNativePG restart the database primary
   (`infisical-db-1`) in place. The Cluster used the default `primaryUpdateMethod: restart`.
2. On shutdown, the old primary tried to archive its last logs through the Barman Cloud plugin,
   but the old pod had no plugin helper container (`wal archive plugin is not available`). Its
   shutdown hung, with up to a 30-minute grace period.
3. We force-deleted the pod. CloudNativePG failed over to `infisical-db-2`, as designed.
4. The new primary briefly couldn't reach the Kubernetes API or `infisical-db-1`, so its liveness
   check shut it down on purpose (it won't keep writing if it might be cut off). It failed back to
   `infisical-db-1`, and the cluster recovered to healthy.
5. Infisical then kept logging `Redis connection error [client=bootstrap] ... ETIMEDOUT`. Every
   in-cluster DNS lookup took 0.3 to 5 seconds, because pods inherited the nodes' Tailscale and
   router search domains and queried those first.

## Root causes

- Default `restart` update method for the primary, combined with turning on archiving on a
  running cluster.
- Two nodes (k3s-mini2, k3s-mini3) reach the tailnet only through Tailscale's relay, so cross-node
  traffic has extra latency and short drops.
- Pods inherited node search domains, making DNS slow.

## Fixes

- `primaryUpdateMethod: switchover` on the Cluster: promote the copy before restarting the primary.
- k3s `resolv-conf` pointing at a node file with only `1.1.1.1` and `8.8.8.8`
  ([networking](../networking.md#dns-inside-the-cluster)).
- Infisical's `REDIS_URL` uses the short Service name `redis`.

- Lima VMs switched to `vzNAT` networking so Tailscale connects nodes directly instead of through
  its relay servers ([networking](../networking.md#lima-vm-networking-vznat)).

## Lessons

- When enabling archiving on a running cluster, expect the old primary's shutdown to hang;
  force-deleting it is safe.
- `kubectl get cluster <name> -n <ns>` and the database pods' logs explain most CloudNativePG
  behavior: look for "Failing over", "promoted", and "liveness check failed".
