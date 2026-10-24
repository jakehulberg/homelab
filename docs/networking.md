# Networking

## Nodes talk over Tailscale

The minis are on Wi-Fi, and bridged VM networking over Wi-Fi is unreliable (most access points
drop frames from extra MAC addresses). So each Lima VM uses Lima's default NAT networking and
joins the tailnet itself. k3s uses the Tailscale interface for everything:

- `--node-ip` is the node's Tailscale IP.
- `flannel-iface: tailscale0` sends pod traffic over Tailscale (`k3s/config.yaml`).
- Nodes find each other by MagicDNS name (`k3s-mini1`, `k3s-mini2`, `k3s-mini3`, `pi`).

Flannel reads the interface MTU, so the smaller Tailscale MTU is handled automatically.

## Reaching services

- `LoadBalancer` services use k3s's built-in ServiceLB, which listens on each node's Tailscale IP.
  MetalLB is not used, since it needs a shared layer 2 network.
- Traefik (installed by Flux) is the single `LoadBalancer`, routing hostnames to services.
- Everything is reachable from devices on the tailnet. Public exposure (for demos) is a later
  decision: Tailscale Funnel or Cloudflare Tunnel.

## Tradeoffs

- etcd runs over Wi-Fi plus Tailscale, which adds latency. Fine for a homelab; if etcd complains,
  moving the minis to ethernet is the fix.
- If Tailscale is down, the cluster network is down.
