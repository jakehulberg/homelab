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
                     SandboxWarmPool "basic" ── keeps 1 sandbox ready ──▶ sandbox pod (container "agent")
```

1. You sign in to web through Authentik.
2. You pick a pool and a TTL. The orchestrator creates a `SandboxClaim` with `shutdownTime` and
   `shutdownPolicy: Delete`, so every sandbox expires. Each session gets its own claim.
3. Agent Sandbox hands the claim a ready sandbox from the warm pool, and the pool starts a
   replacement.
4. A task runs `AIRLOCK_TASK_COMMAND` in the `agent` container through `pods/exec`, with the prompt
   on stdin. Output streams back to the browser.

Claims are labeled with a hash of your OIDC subject; each user sees only their own. Task output lives
in the orchestrator's memory and is lost on restart, so it runs one replica.

## What's where

| Path | What |
|---|---|
| `infrastructure/controllers/agent-sandbox/` | Agent Sandbox `v1.0.8`: CRDs (`Sandbox`, `SandboxTemplate`, `SandboxWarmPool`, `SandboxClaim`) and the controller with `--extensions`, in `agent-sandbox-system`. The release's `sandbox-with-extensions.yaml`, downloaded. |
| `apps/agents/sandbox.yaml` | Namespace `agents` (Pod Security `restricted`), `SandboxTemplate` `basic`, `SandboxWarmPool` `basic` (1 replica) |
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
| Sandbox network | Agent Sandbox's default policy for each template (no `networkPolicy` set): ingress denied, egress to the public internet only, private ranges and metadata blocked. k3s enforces NetworkPolicies. |
| Sandbox pod | Non-root, `RuntimeDefault` seccomp, no privilege escalation, all capabilities dropped, read-only root, memory limit. The namespace enforces Pod Security `restricted`. |

## The sandbox template

`basic` is a placeholder: busybox running `sleep infinity` in a container named `agent`. It proves
the claim flow; tasks fail because there's no agent in it. The real template needs:

- An `agent` container with the `claude` CLI.
- The Agent Vault session sidecar that writes `/session/env`, which the default task command
  sources ([PLAN](../PLAN.md#to-do)).

A pool's `airlock.jakehulberg.com/description` annotation shows in Airlock's claim form.

## Checking it

```sh
kubectl get pods -n agent-sandbox-system           # controller
kubectl get sandboxwarmpool,sandboxclaim,sandbox,pods -n agents
kubectl get pods -n airlock
kubectl logs -n airlock deploy/airlock-orchestrator  # claims and tasks, as JSON
```

## Resources

Controller, web and orchestrator are small Go programs, under 100MiB each. The cost is the warm pool:
each ready sandbox is a running pod. Keep `replicas` low.
