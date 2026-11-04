# Agent Vault

[Infisical Agent Vault](https://infisical.com/docs) keeps credentials out of agent sandboxes. The
sandbox sends its traffic through a proxy, and the proxy adds the real API keys on the way out, so
the agent container never holds a credential. [Airlock](airlock.md) sandboxes use it.

## How it fits together

```
session sidecar ──1. Kubernetes Auth, 2. open session──▶ cluster Infisical
      │                                                         │
      │ 3. /session/env, /session/ca.pem                        │ settings, CA
      ▼                                                         ▼
agent container ──4. HTTPS_PROXY──▶ agent-vault-proxy:17323 ──5. + API key──▶ bundle hosts
└── one sandbox pod (agents, gVisor) ──┘                                      (api.anthropic.com)
```

1. The `session` sidecar logs in to the cluster's Infisical with Kubernetes Auth, using a projected
   ServiceAccount token (audience `infisical`, 10 minutes)
2. The sidecar opens an Agent Vault session on the `airlock` access bundle, TTL 8h, and checks the
   proxy's CA against the pinned fingerprint
3. The sidecar writes the proxy variables (`HTTPS_PROXY`, `NO_PROXY`, CA bundle paths) to
   `/session/env` and the CA to `/session/ca.pem`, on an in-memory volume. Until `/session/env`
   exists, the sidecar's startup probe fails and the agent container doesn't start
4. The agent sources `/session/env`, so its HTTPS traffic goes through the proxy
5. The proxy looks up the session's bundle, refuses hosts outside the bundle, and adds the
   bundle's credential for the host

Session lifetime:

- The session is revoked when the sidecar exits, so deleting the pod ends the session
- Warm pools keep `replicas: 0`, so sessions open at claim time, not while a sandbox sits idle

## What's where

| Path | What |
|---|---|
| `apps/agent-vault/secrets-sync.yaml` | Namespace `agent-vault`; syncs `/agent-vault` from Infisical Cloud into `agent-vault-secrets-infisical` |
| `apps/agent-vault/proxy.yaml` | Deployment and Service `agent-vault-proxy:17323`, NetworkPolicy |
| `apps/agents/claude.yaml` | ConfigMap `agent-session` (identity, bundle, proxy address, CA fingerprint) and the `session` sidecar |
| `apps/agents/token-reviewer.yaml` | ServiceAccount `infisical-token-reviewer` in `infisical`, bound to `system:auth-delegator`, and its token Secret |
| Cluster Infisical | Kubernetes Auth identity, the `airlock` access bundle (allowed hosts, and the credential for each), the proxy's settings and CA |

## The proxy

The proxy is `infisical/cli` running `agent-vault proxy --port=17323`, pointed at the cluster's
Infisical. On every start, the proxy loads its settings and CA with
`INFISICAL_AGENT_VAULT_PROXY_TOKEN`:

- Created with the proxy in the cluster Infisical UI
- Long-lived, so it's stored in Infisical Cloud `/agent-vault` and synced, not minted at runtime

```
Infisical Cloud /agent-vault ──operator──▶ agent-vault-secrets-infisical ──envFrom──▶ proxy
                                                                                        │
                                                       settings, CA ◀── cluster Infisical
```

The Deployment runs one replica, with no ServiceAccount token, all capabilities dropped and a
256Mi memory limit.

## Network

```
sandbox pod (agents) ──▶ kube-dns:53 (kube-system)
                     ──▶ agent-vault-proxy:17323 (agent-vault)
                     ──▶ port 8080 in infisical          Kubernetes Auth, session
                     ✕   anything else
```

- The sandbox template's `networkPolicy` allows only those three destinations
- The proxy's NetworkPolicy accepts connections only from pods in `agents`
- The sandbox's egress to Infisical goes away once sessions are minted outside the pod
  ([PLAN](../PLAN.md#to-do))

## Kubernetes Auth

```
sidecar ──projected token──▶ Infisical ──TokenReview (infisical-token-reviewer)──▶ kubernetes.default.svc
```

- Identity `a56b50e2-...` (in `agent-session`), bound to ServiceAccount `coding-agent` in `agents`
- `coding-agent` has no RBAC and no mounted API token. The projected token's audience
  `infisical` makes it useless against the Kubernetes API
- Infisical calls TokenReview with the token of ServiceAccount `infisical-token-reviewer`, from
  the Secret of the same name. The ServiceAccount's only permission is `system:auth-delegator`
- Infisical needs `ALLOW_INTERNAL_IP_CONNECTIONS=true`, because the token review goes to
  `https://kubernetes.default.svc`

## CA fingerprint

The proxy terminates TLS with its own CA. The sidecar trusts the CA only if the CA's fingerprint
matches `CA_FINGERPRINT` in `agent-session`. The fingerprint is in the proxy's startup log:

```sh
kubectl logs -n agent-vault deploy/agent-vault-proxy | grep -i fingerprint  # SHA256:...
```

If the CA changes, update `CA_FINGERPRINT` in `apps/agents/claude.yaml`, commit, and claim a new
sandbox.

## Checking it

```sh
kubectl get pods,infisicalstaticsecret -n agent-vault  # proxy 1/1 Running
kubectl logs -n agent-vault deploy/agent-vault-proxy   # no errors after startup
kubectl logs -n agents <sandbox-pod> -c session        # login and session opened
kubectl exec -n agents <sandbox-pod> -c agent -- sh -c '. /session/env; env | grep -i proxy'  # HTTPS_PROXY set
```

## Troubleshooting

- **Sandbox stuck starting:** the startup probe waits 90s for `/session/env`. Check the sidecar
  with `kubectl logs -n agents <sandbox-pod> -c session`. A login failure points at Kubernetes
  Auth or the token reviewer; a fingerprint mismatch points at `CA_FINGERPRINT`
- **Proxy crash-looping:** `INFISICAL_AGENT_VAULT_PROXY_TOKEN` is missing or revoked. Check the
  sync with `kubectl describe infisicalstaticsecret agent-vault-secrets -n agent-vault`
- **Request refused by the proxy:** the host isn't in the `airlock` bundle. Add the host to the
  bundle in the cluster Infisical
