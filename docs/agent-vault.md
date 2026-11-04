# Agent Vault

[Infisical Agent Vault](https://infisical.com/docs) keeps credentials out of agent sandboxes. The
sandbox sends its traffic through a proxy, and the proxy attaches the real API keys on the way out.
The agent container never holds a credential. Used by [Airlock](airlock.md) sandboxes.

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
   ServiceAccount token (audience `infisical`, 10 minutes).
2. It opens an Agent Vault session on the `airlock` access bundle, TTL 8h, and checks the proxy's
   CA against the pinned fingerprint.
3. It writes the proxy variables (`HTTPS_PROXY`, `NO_PROXY`, CA bundle paths) to `/session/env`
   and the CA to `/session/ca.pem`, on an in-memory volume. The startup probe holds the agent until
   `/session/env` exists.
4. The agent sources `/session/env`, so its HTTPS goes through the proxy.
5. The proxy matches the session to the bundle, refuses hosts outside it, and adds the credential.

The session is revoked when the sidecar exits, so deleting the pod ends it. Warm pools keep
`replicas: 0`, so sessions open at claim time, not while a sandbox sits idle.

## What's where

| Path | What |
|---|---|
| `apps/agent-vault/secrets-sync.yaml` | Namespace `agent-vault`; syncs `/agent-vault` from Infisical Cloud into `agent-vault-secrets-infisical` |
| `apps/agent-vault/proxy.yaml` | Deployment and Service `agent-vault-proxy:17323`, NetworkPolicy |
| `apps/agents/claude.yaml` | ConfigMap `agent-session` (identity, bundle, proxy address, CA fingerprint) and the `session` sidecar |
| `apps/agents/token-reviewer.yaml` | ServiceAccount `infisical-token-reviewer`, which Infisical uses to review sandbox tokens |
| Cluster Infisical | Kubernetes Auth identity, the `airlock` access bundle (hosts, credentials), the proxy's settings and CA |

Which hosts are allowed, and which credential each gets, is set in Infisical, not in git.

## The proxy

`infisical/cli` running `agent-vault proxy --port=17323`, pointed at the cluster's Infisical. On
every start it loads its settings and CA with `INFISICAL_AGENT_VAULT_PROXY_TOKEN`, from Infisical
Cloud `/agent-vault`.

The token comes from creating the proxy in the cluster Infisical UI. It's long-lived, so it's
stored in Cloud `/agent-vault` and synced, not minted at runtime.

```
Infisical Cloud /agent-vault ──operator──▶ agent-vault-secrets-infisical ──envFrom──▶ proxy
                                                                                        │
                                                       settings, CA ◀── cluster Infisical
```

No ServiceAccount token, all capabilities dropped, 256Mi memory limit, one replica.

## Network

```
agents (sandbox) ──▶ kube-dns:53
                 ──▶ agent-vault-proxy:17323     only ingress the proxy accepts
                 ──▶ infisical:8080              Kubernetes Auth, session
                 ✕   anything else
```

The sandbox's NetworkPolicy allows only those three. The proxy's NetworkPolicy accepts only pods
in `agents`. Direct egress to Infisical goes away once sessions are minted outside the pod
([PLAN](../PLAN.md)).

## Kubernetes Auth

```
sidecar ──projected token──▶ Infisical ──TokenReview (infisical-token-reviewer)──▶ kubernetes.default.svc
```

- Identity `a56b50e2-...` (in `agent-session`), bound to ServiceAccount `coding-agent` in `agents`.
- `coding-agent` has no RBAC and no mounted API token. The projected token's audience
  `infisical` makes it useless against the Kubernetes API.
- Infisical needs `ALLOW_INTERNAL_IP_CONNECTIONS=true`, because the token review goes to
  `https://kubernetes.default.svc`.

## CA fingerprint

The proxy terminates TLS with its own CA. The sidecar trusts it only if its fingerprint matches
`CA_FINGERPRINT` in `agent-session`. The fingerprint is in the proxy's startup log:

```sh
kubectl logs -n agent-vault deploy/agent-vault-proxy | grep -i fingerprint
```

If the CA changes, update `CA_FINGERPRINT`, commit, and claim a new sandbox.

## Checking it

```sh
kubectl get pods,infisicalstaticsecret -n agent-vault
kubectl logs -n agent-vault deploy/agent-vault-proxy
kubectl logs -n agents <sandbox-pod> -c session      # login and session
kubectl exec -n agents <sandbox-pod> -c agent -- sh -c '. /session/env; env | grep -i proxy'
```

## Troubleshooting

- **Sandbox stuck starting:** the startup probe waits 90s for `/session/env`. Check the `session`
  logs: login failures point at Kubernetes Auth or the token reviewer; a fingerprint mismatch
  points at `CA_FINGERPRINT`.
- **Proxy crash-looping:** `INFISICAL_AGENT_VAULT_PROXY_TOKEN` missing or revoked. Check the sync
  with `kubectl describe infisicalstaticsecret agent-vault-secrets -n agent-vault`.
- **Request refused by the proxy:** the host isn't in the `airlock` bundle. Add it in Infisical.
