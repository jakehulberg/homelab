# Networking

## Nodes talk over Tailscale

The minis are on Wi-Fi, where bridged VM networking is unreliable (most access points drop frames
from extra MAC addresses). So each Lima VM uses NAT ([vzNAT](#lima-vm-networking-vznat)) and joins
the tailnet itself. k3s uses Tailscale for everything:

- `--node-ip` is the node's Tailscale IP.
- `flannel-iface: tailscale0` sends pod traffic over Tailscale (`k3s/config.yaml`).
- Nodes find each other by MagicDNS name (`k3s-mini1`, `k3s-mini2`, `k3s-mini3`, `pi`).

Flannel reads the interface MTU, so Tailscale's smaller MTU is handled automatically.

## Reaching services

Traefik is the single front door, using [Gateway API](https://gateway-api.sigs.k8s.io/) (not
Ingress).

```
browser
  → node Tailscale IP, port 80        (ServiceLB, on every node)
  → Service traefik 80 → pod port 8000
  → Traefik pod
  → Gateway traefik-gateway           (listener "web", port 8000)
  → HTTPRoute matching the hostname   (lives next to each app)
  → the app's Service → its pods
```

| Piece | Source |
|---|---|
| Gateway API CRDs (`v1.6.3`) | `infrastructure/controllers/gateway-api/`, from the Gateway API release (the Traefik chart doesn't ship them) |
| Traefik, its `LoadBalancer` Service, GatewayClass `traefik`, Gateway `traefik-gateway` | `infrastructure/controllers/traefik/`, Traefik chart `42.1.3`. Ports, Service type, GatewayClass and Gateway are chart defaults. |
| HTTPRoutes | Each app's folder, e.g. `apps/podinfo/httproute.yaml` |

- `LoadBalancer` Services use k3s's ServiceLB, listening on each node's Tailscale IP. No MetalLB
  (needs a shared layer 2 network).
- The Gateway accepts routes from any namespace (`namespacePolicy: from: All`).
- Hostnames are `<app>.home.jakehulberg.com`. A Cloudflare wildcard record points them all at the
  four node Tailscale IPs, so new apps need no DNS change ([certificates](certificates.md)).
- Listeners: `web` (HTTP, pod 8000, Service 80) and `websecure` (HTTPS, pod 8443, Service 443,
  `home-wildcard-tls`). Routes without a `sectionName` attach to both.
- Traefik redirects all HTTP to HTTPS (308) at the `web` port before routing
  (`ports.web.http.redirections`), so apps don't need their own.
- Reachable only from the tailnet. Public exposure (for demos) is undecided: Tailscale Funnel or
  Cloudflare Tunnel.

### Expose an app

Add `httproute.yaml` to the app's folder and list it in that folder's `kustomization.yaml`:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: <app>
  namespace: <app-namespace>
spec:
  parentRefs:
    - name: traefik-gateway
      namespace: traefik
  hostnames:
    - <app>.home.jakehulberg.com
  rules:
    - backendRefs:
        - name: <app-service>
          port: <service-port>
```

`kubectl describe httproute <app> -n <app-namespace>` should show `Accepted` and `ResolvedRefs`
`True`. `404 page not found` means Traefik got the request but no route matched the hostname.

### Check the plumbing

```sh
kubectl get gateway traefik-gateway -n traefik -o yaml   # listeners: port 8000
kubectl get svc traefik -n traefik                       # 80:<nodeport>, external IPs = Tailscale IPs
helm get manifest traefik -n traefik                     # everything the chart created
```

## DNS inside the cluster

k3s gives pods the node's DNS settings by default, which include Tailscale's and the router's search
domains (`tail00e871.ts.net`, `attlocal.net`). Lookups like `redis.infisical.svc.cluster.local`
asked those outside servers first and took 0.3 to 5 seconds, so Infisical missed its 10-second
Redis connect timeout.

Fix: each node has `/etc/rancher/k3s/resolv.conf` with only public servers and no search domains,
and `k3s/config.yaml` points k3s at it via `resolv-conf:`.

```
nameserver 1.1.1.1
nameserver 8.8.8.8
```

Pod search lists now end at `cluster.local`; CoreDNS forwards outside names to 1.1.1.1. Pods pick
this up only when recreated. Check:

```sh
kubectl run dnscheck --rm -it --restart=Never --image=busybox:1.37 -- cat /etc/resolv.conf
# search <ns>.svc.cluster.local svc.cluster.local cluster.local   (nothing after cluster.local)
```

Fastest in-cluster names: the short Service name from the same namespace (`redis`) or the full name
(`redis.infisical.svc.cluster.local`).

## Tradeoffs

- etcd runs over Wi-Fi plus Tailscale. Fine for a homelab; if etcd complains, ethernet is the fix.
- Tailscale down means the cluster network is down.
- Node-to-node latency is Wi-Fi latency: 60 to 280ms on direct paths, with spikes. Ethernet would
  fix that and allow bridged networking.

## Lima VM networking: vzNAT

Each VM uses Apple's VM NAT (`networks: - vzNAT: true` in `lima/`). Lima's default user-mode network
is a "hard" NAT (different outside port per destination), so Tailscale couldn't connect VMs directly
and relayed node traffic through its servers (Dallas), adding latency and drops. That once made the
database primary shut itself down ([incident](incidents/2026-10-25-infisical-db-failover.md)). vzNAT
also gives VMs IPv6, which Tailscale uses for direct paths.

Check a VM, then a path (on the mini that owns the VM):

```sh
limactl shell k3s-mini2 tailscale netcheck        # MappingVariesByDestIP: false
limactl shell k3s-mini2 tailscale ping k3s-mini1  # "via <IP>:<port>", not "via DERP(...)"
```

Adding it to an existing VM, one node at a time (drain first, see
[nodes](nodes.md#taking-a-node-down)):

```sh
limactl stop k3s-mini2
limactl edit k3s-mini2 --set '.networks += [{"vzNAT": true}]'   # answer yes to start it
```
