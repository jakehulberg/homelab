# Networking

## Nodes talk over Tailscale

The minis are on Wi-Fi, and bridged VM networking over Wi-Fi is unreliable (most access points drop frames from extra MAC addresses). So each Lima VM uses Lima's default NAT networking and joins the tailnet itself. k3s uses the Tailscale interface for everything:

- `--node-ip` is the node's Tailscale IP.
- `flannel-iface: tailscale0` sends pod traffic over Tailscale (`k3s/config.yaml`).
- Nodes find each other by MagicDNS name (`k3s-mini1`, `k3s-mini2`, `k3s-mini3`, `pi`).

Flannel reads the interface MTU, so the smaller Tailscale MTU is handled automatically.

## Reaching services

Traefik is the single front door, configured with [Gateway API](https://gateway-api.sigs.k8s.io/)
(not the older Ingress).

```
browser
  → node Tailscale IP, port 80        (ServiceLB, on every node)
  → Service traefik 80 → pod port 8000
  → Traefik pod
  → Gateway traefik-gateway           (listener "web", port 8000)
  → HTTPRoute matching the hostname   (lives next to each app)
  → the app's Service → its pods
```

| Piece | Where it comes from |
|---|---|
| Gateway API CRDs (`v1.6.3`) | `infrastructure/controllers/gateway-api/`, from the Gateway API release. The Traefik chart doesn't include them. |
| Traefik, its `LoadBalancer` Service, GatewayClass `traefik`, Gateway `traefik-gateway` | `infrastructure/controllers/traefik/`, Traefik Helm chart `42.1.3`. Ports, Service type, GatewayClass and Gateway are chart defaults. |
| HTTPRoutes | Each app's folder, e.g. `apps/podinfo/httproute.yaml` |

- `LoadBalancer` services use k3s's built-in ServiceLB, which listens on each node's Tailscale IP.
  MetalLB is not used, since it needs a shared layer 2 network.
- The Gateway accepts routes from any namespace (`namespacePolicy: from: All`).
- Hostnames are `<app>.home.jakehulberg.com`. A wildcard DNS record in Cloudflare points every
  name under `home.jakehulberg.com` at the four node Tailscale IPs, so a new app needs no DNS
  change. See [certificates](certificates.md).
- The Gateway has two listeners: `web` (HTTP, pod port 8000, Service port 80) and `websecure`
  (HTTPS, pod port 8443, Service port 443, using the `home-wildcard-tls` certificate). A route
  that doesn't name a listener (`sectionName`) attaches to both.
- Traefik redirects all HTTP to HTTPS (308) at its `web` port, before routing
  (`ports.web.http.redirections` in the Traefik values). Apps don't need their own redirect.
- Everything is reachable from devices on the tailnet. Public exposure (for demos) is a later
  decision: Tailscale Funnel or Cloudflare Tunnel.

### Expose an app

Add an `httproute.yaml` to the app's folder and list it in that folder's `kustomization.yaml`:

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

Check it with `kubectl describe httproute <app> -n <app-namespace>`: `Accepted` and `ResolvedRefs`
should both be `True`. A `404 page not found` means Traefik got the request but no route matched
the hostname.

### Check the plumbing

```sh
kubectl get gateway traefik-gateway -n traefik -o yaml   # listeners: port 8000
kubectl get svc traefik -n traefik                       # 80:<nodeport>, external IPs = Tailscale IPs
helm get manifest traefik -n traefik                     # everything the chart created
```

## DNS inside the cluster

Pods get their DNS settings from k3s, which by default copies the node's. The nodes' settings
include Tailscale's and the router's search domains (`tail00e871.ts.net`, `attlocal.net`), so pods
looked up names like `redis.infisical.svc.cluster.local` by first asking those outside servers, and
lookups took 0.3 to 5 seconds instead of milliseconds. Infisical couldn't reach Redis within its
10-second connect timeout.

Fix: every node has its own `/etc/rancher/k3s/resolv.conf` with only public DNS servers and no
search domains, and `k3s/config.yaml` points k3s at it with `resolv-conf:`.

```
nameserver 1.1.1.1
nameserver 8.8.8.8
```

Pods' search list now stops at `cluster.local`, and CoreDNS forwards outside names to 1.1.1.1.
Pods only pick this up when they're recreated. Check with a throwaway pod:

```sh
kubectl run dnscheck --rm -it --restart=Never --image=busybox:1.37 -- cat /etc/resolv.conf
# search <ns>.svc.cluster.local svc.cluster.local cluster.local   (nothing after cluster.local)
```

Tip: in-cluster addresses resolve fastest as the short Service name from the same namespace
(`redis`) or the full name (`redis.infisical.svc.cluster.local`).

## Tradeoffs

- etcd runs over Wi-Fi plus Tailscale, which adds latency. Fine for a homelab; if etcd complains,
  moving the minis to ethernet is the fix.
- If Tailscale is down, the cluster network is down.
- k3s-mini2 and k3s-mini3 currently reach the tailnet only through Tailscale's relay
  (`tailscale ping k3s-mini2` shows `via DERP(dfw)`), not directly. That adds latency and short
  drops, which once made the database primary shut itself down and fail over. Likely Lima's default
  VM networking; on the to-do list in `PLAN.md`.
