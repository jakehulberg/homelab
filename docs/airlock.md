# Airlock and agent sandboxes

[Airlock](https://github.com/jakehulberg/airlock) is a web UI for claiming agent sandboxes and
running agent tasks in them, at `https://airlock.home.jakehulberg.com`. Sandboxes come from
[Agent Sandbox](https://github.com/kubernetes-sigs/agent-sandbox). Sign-in is through
[Authentik](authentik.md).

## How it fits together

```
browser ──OIDC (Authentik)──▶ airlock-web ──bearer token + user headers──▶ airlock-orchestrator
                                                                                │
                                         SandboxClaim, pods/exec, in "agents" only
                                                                                ▼
                     SandboxWarmPool "claude" ── sandbox per claim ──▶ sandbox pod (container "agent")
```

1. You sign in to web through Authentik.
2. You pick a pool and a TTL. The orchestrator creates a `SandboxClaim` with `shutdownTime` and
   `shutdownPolicy: Delete`, so every sandbox expires. Each session gets its own claim.
3. Agent Sandbox binds the claim to a sandbox from the pool. `claude` keeps `replicas: 0`, so each
   claim starts a fresh sandbox.
4. A task runs the pool's task command in its agent container through `pods/exec`, with the prompt
   on stdin. Output streams back to the browser.

Claims are labeled with a hash of your OIDC subject; each user sees only their own. Task output lives
in the orchestrator's memory and is lost on restart, so it runs one replica.

## What's where

| Path | What |
|---|---|
| `infrastructure/controllers/agent-sandbox/` | Agent Sandbox `v1.0.8`: CRDs (`Sandbox`, `SandboxTemplate`, `SandboxWarmPool`, `SandboxClaim`) and the controller with `--extensions`, in `agent-sandbox-system`. The release's `sandbox-with-extensions.yaml`, downloaded. |
| `apps/agents/sandbox.yaml` | Namespace `agents` (Pod Security `restricted`) |
| `apps/agents/claude.yaml` | ConfigMap `agent-session`, ServiceAccount `coding-agent`, `SandboxTemplate` and `SandboxWarmPool` `claude` |
| `apps/agents/runtimeclass.yaml` | `RuntimeClass` `gvisor`: handler `runsc`, scheduled on the minis only |
| `apps/agents/token-reviewer.yaml` | ServiceAccount `infisical-token-reviewer` in `infisical` with `system:auth-delegator`, and its token Secret, for Infisical's Kubernetes Auth |
| `apps/agent-vault/` | Agent Vault proxy: Deployment, Service `agent-vault-proxy:17323`, NetworkPolicy (ingress from `agents` only), and `/agent-vault` synced from Infisical Cloud |
| `apps/airlock/secrets-sync.yaml` | Namespace `airlock`; syncs `/airlock` into `airlock-secrets-infisical` and `/ghcr` into the image pull secret `ghcr-pull` |
| `apps/airlock/orchestrator.yaml` | ServiceAccount, `Role` and `RoleBinding` in `agents`, Deployment, Service `airlock-orchestrator:8081`, NetworkPolicy |
| `apps/airlock/web.yaml` | Deployment, Service `airlock-web:8080` |
| `apps/airlock/httproute.yaml` | `airlock.home.jakehulberg.com` → `airlock-web:8080` |
| `apps/airlock/image-policy.yaml` | Flux `ImageRepository` and `ImagePolicy` for both images |

## Images

GitHub Actions in the Airlock repo builds `ghcr.io/jakehulberg/airlock-orchestrator` and
`ghcr.io/jakehulberg/airlock-web` on every push to `main`, for amd64 and arm64. Both share a tag,
`YYYYMMDDTHHMMSS-<sha7>`. Flux image automation commits new tags here, like life-dashboard. The
packages are private, so both Deployments use `ghcr-pull`.

## Secrets

From `/airlock` in Infisical Cloud:

| Secret | What | Used by |
|---|---|---|
| `OIDC_ISSUER_URL` | `https://auth.home.jakehulberg.com/application/o/airlock/` | web |
| `OIDC_CLIENT_ID`, `OIDC_CLIENT_SECRET` | From the Authentik provider | web |
| `AIRLOCK_SESSION_KEY` | Encrypts the session cookie. `openssl rand -hex 32` | web |
| `AIRLOCK_API_TOKEN` | Web's credential for the orchestrator. `openssl rand -hex 32` | web, orchestrator |

Web loads all of them with `envFrom`. The orchestrator gets only `AIRLOCK_API_TOKEN`.
`AIRLOCK_PUBLIC_URL` and `AIRLOCK_ORCHESTRATOR_URL` aren't secret and are set in `web.yaml`.

## Isolation

| Layer | How |
|---|---|
| Kubernetes API | The orchestrator's Role covers only the `agents` namespace: claims, pools, sandboxes, pods, `pods/exec`. Web has no ServiceAccount token. Sandboxes have `automountServiceAccountToken: false`. |
| Orchestrator | NetworkPolicy: only pods labeled `app: airlock-web` may reach port 8081. |
| Sandbox network | The template's `networkPolicy`: ingress denied, egress only to DNS, the Agent Vault proxy and Infisical. The proxy allows only the access bundle's hosts. k3s enforces NetworkPolicies. |
| Credentials | None in the agent container. The proxy adds the API key on the way out. |
| Kernel | gVisor (`runtimeClassName: gvisor`). The agent's syscalls go to gVisor's user-space kernel, not the VM's. Both containers in the pod share one gVisor kernel. |
| Sandbox pod | Non-root, `RuntimeDefault` seccomp, no privilege escalation, all capabilities dropped, read-only root, memory limit. The namespace enforces Pod Security `restricted`. |

## The sandbox template

`claude` runs `ghcr.io/jakehulberg/agent-claude` from
[agent-images](https://github.com/jakehulberg/agent-images), plus a `session` sidecar:

1. The sidecar logs in to the homelab Infisical with Kubernetes Auth, using a projected
   ServiceAccount token with audience `infisical`.
2. It opens an Agent Vault session on the `airlock` access bundle and writes the proxy settings
   and CA to `/session/env` and `/session/ca.pem`. The session is revoked when the pod is deleted.
3. The task command sources `/session/env` and runs `claude -p`, with `--continue` after the
   first task, so a claim is one conversation.

Kubernetes Auth needs `ALLOW_INTERNAL_IP_CONNECTIONS=true` in Infisical, because the token review
goes to `https://kubernetes.default.svc`. The proxy's CA fingerprint is pinned in `agent-session`.

Pool annotations Airlock reads:

| Annotation | What |
|---|---|
| `airlock.jakehulberg.com/description` | Shown in the claim form |
| `airlock.jakehulberg.com/agent-container` | Container to exec into (default `agent`) |
| `airlock.jakehulberg.com/task-command` | Run with `sh -c`, prompt on stdin |

## gVisor

`RuntimeClass` `gvisor` maps to the `runsc` runtime in containerd. `runsc` is installed by hand in
each mini's VM ([bootstrap](bootstrap.md#11-gvisor-on-the-minis)), so it isn't in git. The
RuntimeClass's node selector (`node-role.kubernetes.io/control-plane: "true"`) keeps sandboxes on
the minis, since the Pi has no `runsc`. A mini added as a plain agent wouldn't match; label nodes
instead if that happens. Other pods keep the default runtime.

Inside a sandbox, `dmesg` prints gVisor's boot lines and `uname -r` reports `4.4.0`. On runc, a
non-root `dmesg` is refused.

## Checking it

```sh
kubectl get pods -n agent-sandbox-system           # controller
kubectl get sandboxwarmpool,sandboxclaim,sandbox,pods -n agents
kubectl get pods -n airlock
kubectl logs -n airlock deploy/airlock-orchestrator  # claims and tasks, as JSON
```

## Resources

Controller, web and orchestrator are small Go programs, under 100MiB each. Each claimed sandbox is a
pod with up to 1 CPU and 1GiB for the agent. Warm pool `replicas` keep idle pods running, so they
stay at 0.
