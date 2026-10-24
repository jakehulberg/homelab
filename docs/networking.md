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
| Gateway API CRDs (`v1.6.3`) | `infrastructure/gateway-api/`, from the Gateway API release. The Traefik chart doesn't include them. |
| Traefik, its `LoadBalancer` Service, GatewayClass `traefik`, Gateway `traefik-gateway` | `infrastructure/traefik/`, Traefik Helm chart `42.1.3`. Ports, Service type, GatewayClass and Gateway are chart defaults. |
| HTTPRoutes | Each app's folder, e.g. `apps/podinfo/httproute.yaml` |

- `LoadBalancer` services use k3s's built-in ServiceLB, which listens on each node's Tailscale IP.
  MetalLB is not used, since it needs a shared layer 2 network.
- The Gateway accepts routes from any namespace (`namespacePolicy: from: All`).
- HTTP only for now. HTTPS comes with cert-manager.
- Hostnames use `nip.io` until a real domain is set up: `<app>.100.102.222.95.nip.io` resolves to
  k3s-mini2's Tailscale IP.
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
    - <app>.100.102.222.95.nip.io
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

## Tradeoffs

- etcd runs over Wi-Fi plus Tailscale, which adds latency. Fine for a homelab; if etcd complains,
  moving the minis to ethernet is the fix.
- If Tailscale is down, the cluster network is down.
