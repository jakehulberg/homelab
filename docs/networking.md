# Networking

All traffic between nodes, pod traffic included, runs over Tailscale, and Traefik is the one
entry point for apps. The minis are on Ethernet, and each VM is bridged onto the home LAN, so
Tailscale connects the nodes directly at about 1ms.

## Nodes talk over Tailscale

Each Lima VM has its own LAN address ([bridged](#lima-vm-networking-bridged)) and runs its own
Tailscale. k3s sends everything over Tailscale:

- `--node-ip` on each node's install command is that node's Tailscale IP
  ([bootstrap](bootstrap.md#5-install-k3s))
- `flannel-iface: tailscale0` in `k3s/config.yaml` sends pod traffic over Tailscale
- Nodes reach each other by MagicDNS name: `k3s-mini1`, `k3s-mini2`, `k3s-mini3`, `pi`

Flannel reads the MTU of `tailscale0`, so pod traffic gets Tailscale's smaller MTU without extra
config.

## Lima VM networking: bridged

Each VM is bridged onto the mini's Ethernet port (`- lima: bridged` in `lima/k3s-mini*.yaml`), so
the router gives it its own `192.168.1.x` address. Tailscale then connects the VMs straight across
the LAN.

```
k3s-mini1 VM ──┐
k3s-mini2 VM ──┼── switch ── router 192.168.1.254
k3s-mini3 VM ──┘   Tailscale: VM ──▶ VM over a LAN address (IPv4 or IPv6), ~1ms
```

Without the bridge, each VM sits behind its Mac's NAT, and Tailscale has to punch through that NAT
and the router to connect two VMs. When that fails, Tailscale sends node traffic through its relay
servers at 60 to 500ms, and etcd drops the node: the relay made the database primary shut down
([incident](incidents/2026-10-25-infisical-db-failover.md)). etcd expects round trips well under
10ms between members.

Bridging needs Ethernet: most Wi-Fi access points drop frames from the VM's extra MAC address. The
`vzNAT` network in the same files is left from the Wi-Fi setup; Tailscale prefers the LAN path.

To bridge an existing VM, install socket_vmnet on its mini ([bootstrap step
2a](bootstrap.md#2-create-the-vms)), take the node down ([nodes](nodes.md#taking-a-node-down)),
then run this on the mini as the VM's owner:

```sh
limactl shell k3s-mini2 sudo poweroff
limactl stop -f k3s-mini2
limactl edit k3s-mini2 --set '.networks += [{"lima": "bridged"}]'   # answer yes to start it
limactl shell k3s-mini2 ip -4 addr show lima1                       # inet 192.168.1.x
```

## Tradeoffs

- If Tailscale is down, the cluster network is down
- If a mini's Ethernet link drops, its VM loses its LAN address, Tailscale falls back to the
  vzNAT path or a relay, and etcd may drop that node

## Reaching services

Traefik serves every app through [Gateway API](https://gateway-api.sigs.k8s.io/), not Ingress. A
request from a browser on the tailnet takes this path:

```
browser ──:80 / :443──▶ node Tailscale IP (ServiceLB, on every node)
        ──▶ Service traefik (80 ──▶ pod 8000, 443 ──▶ pod 8443)
        ──▶ Gateway traefik-gateway (listener web or websecure)
        ──▶ HTTPRoute matching the hostname (in the app's folder)
        ──▶ the app's Service ──▶ its pods
```

- Hostnames are `<app>.home.jakehulberg.com`. A Cloudflare wildcard record points them all at the
  four node Tailscale IPs, so a new app needs no DNS change
  ([certificates](certificates.md#dns-records))
- Listeners: `web` (HTTP, pod 8000, Service 80) and `websecure` (HTTPS, pod 8443, Service 443,
  certificate `home-wildcard-tls`). An HTTPRoute without a `sectionName` attaches to both
- Traefik redirects every HTTP request on `web` to HTTPS with a 308
  (`ports.web.http.redirections`), so apps don't need their own redirect
- Both listeners accept routes from any namespace (`namespacePolicy: from: All`)
- `LoadBalancer` Services use k3s's ServiceLB, which listens on each node's Tailscale IP. MetalLB
  isn't used: it needs a shared layer 2 network
- Only devices on the tailnet can reach apps. Public exposure (for demos) is undecided: Tailscale
  Funnel or Cloudflare Tunnel

## What's where

| Path | What |
|---|---|
| `infrastructure/controllers/gateway-api/` | Gateway API CRDs `v1.6.3`, from the Gateway API release (the Traefik chart doesn't ship them) |
| `infrastructure/controllers/traefik/` | Traefik chart `42.1.3`: Traefik, its `LoadBalancer` Service, GatewayClass `traefik`, Gateway `traefik-gateway`. Ports, Service type, GatewayClass and Gateway name are chart defaults |
| `apps/<app>/httproute.yaml` | Each app's HTTPRoute, e.g. `apps/podinfo/httproute.yaml` |
| `k3s/config.yaml` | `flannel-iface` and `resolv-conf` for every node |
| `lima/k3s-mini*.yaml` | Each VM's network (`lima: bridged`) |
| Each mini | socket_vmnet at `/opt/socket_vmnet/bin/`, `/etc/sudoers.d/lima`, and `bridged` on `en0` in `~/.lima/_config/networks.yaml` ([bootstrap](bootstrap.md#2-create-the-vms)) |
| Each node | `/etc/rancher/k3s/resolv.conf` ([DNS inside the cluster](#dns-inside-the-cluster)) |
| Cloudflare | The `*.home` DNS records ([certificates](certificates.md#dns-records)) |

## Expose an app

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

```sh
kubectl describe httproute <app> -n <app-namespace>   # Accepted True, ResolvedRefs True
```

## DNS inside the cluster

Each node has `/etc/rancher/k3s/resolv.conf` with only public servers and no search domains, and
`resolv-conf:` in `k3s/config.yaml` points k3s at that file:

```
nameserver 1.1.1.1
nameserver 8.8.8.8
```

By default, k3s gives pods the node's `/etc/resolv.conf`, which includes Tailscale's and the
router's search domains (`tail00e871.ts.net`, `attlocal.net`). With those, a lookup like
`redis.infisical.svc.cluster.local` went to the outside servers first and took 0.3 to 5 seconds,
so Infisical missed its 10-second Redis connect timeout.

Now pod search lists end at `cluster.local`, and CoreDNS forwards outside names to `1.1.1.1`. A pod
gets the new settings only when it's recreated. The fastest in-cluster names are the short Service
name from the same namespace (`redis`) or the full name (`redis.infisical.svc.cluster.local`).

## Checking it

From the laptop:

```sh
kubectl get gateway traefik-gateway -n traefik -o yaml   # listeners: ports 8000 and 8443
kubectl get svc traefik -n traefik                       # 80 and 443, external IPs = Tailscale IPs
helm get manifest traefik -n traefik                     # everything the chart created
kubectl run dnscheck --rm -it --restart=Never --image=busybox:1.37 -- cat /etc/resolv.conf
# search <ns>.svc.cluster.local svc.cluster.local cluster.local   (nothing after cluster.local)
```

On the mini that owns the VM:

```sh
route -n get default | grep interface                     # en0 (Ethernet)
limactl shell k3s-mini2 ip -4 addr show lima1             # inet 192.168.1.x
limactl shell k3s-mini2 sudo tailscale ping k3s-mini1     # via a LAN address, ~1ms, not DERP(...)
limactl shell k3s-mini2 tailscale status | grep relay     # nothing
```

## Troubleshooting

- **`404 page not found` on an app's hostname:** Traefik got the request, but no HTTPRoute matched
  the hostname. Run `kubectl describe httproute <app> -n <app-namespace>` and check that the route
  is `Accepted`
- **`tailscale ping` shows `via DERP(...)`, or `tailscale status` shows `relay`:** the VM has no
  LAN address, so its node traffic goes through Tailscale's relays. Check the mini is on `en0` and
  the VM has an address on `lima1` ([Lima VM networking](#lima-vm-networking-bridged))
- **`KubePodNotReady` alerts for pods whose health checks time out, on several nodes:** check
  node-to-node latency first with `tailscale ping`. A relayed path makes database round trips slow
  enough to fail probes
