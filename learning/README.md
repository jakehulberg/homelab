# Learning

Reading list for understanding what this homelab is built from.

| # | Read | Time | Why it matters here |
|---|---|---|---|
| 1 | [Kubernetes concepts](https://kubernetes.io/docs/concepts/): "Workloads" and "Services, Load Balancing, and Networking" | ~1h | Pods, Deployments, Services and Namespaces are what every app in `apps/` turns into |
| 2 | [Flux core concepts](https://fluxcd.io/flux/concepts/) | ~20m | Sources, Kustomizations and HelmReleases: how pushes to this repo reach the cluster |
| 3 | [Flux repository structure](https://fluxcd.io/flux/guides/repository-structure/) | ~15m | Why the repo splits `clusters/`, `infrastructure/` and `apps/` |
| 4 | [Gateway API overview](https://gateway-api.sigs.k8s.io/concepts/api-overview/) | ~15m | GatewayClass, Gateway and HTTPRoute: how traffic reaches apps through Traefik |

