# Airlock and agent sandboxes

[Airlock](https://github.com/jakehulberg/airlock) is a web UI for claiming agent sandboxes and
running agent tasks in them, at `https://airlock.home.jakehulberg.com`. Jake uses it to run coding
agents on the cluster, each in its own pod that expires. Sandboxes come from
[Agent Sandbox](https://github.com/kubernetes-sigs/agent-sandbox), sign-in goes through
[Authentik](authentik.md), and API keys go through [Agent Vault](agent-vault.md).

## How it fits together

```
browser ──1. OIDC (Authentik)──▶ airlock-web:8080 ──bearer token + user headers──▶ orchestrator
                                                                                     │
                                       2. SandboxClaim, 4. pods/exec, in agents only │
                                                                                     ▼
     SandboxWarmPool claude ──3. one sandbox per claim──▶ sandbox pod (container agent)
```

1. You sign in to `airlock-web` through Authentik
2. You pick a pool and a TTL. The orchestrator creates a `SandboxClaim` with `shutdownTime` and
   `shutdownPolicy: Delete`, so every sandbox expires. Each session gets its own claim
3. Agent Sandbox binds the claim to a sandbox from the pool. The `claude` pool keeps
   `replicas: 0`, so each claim starts a fresh sandbox
4. For each task, the orchestrator runs the pool's task command in the `agent` container through
   `pods/exec`, with the prompt on stdin, and streams the output back to the browser

- Claims are labeled with a hash of your OIDC subject; each user sees only their own
- Task output lives in the orchestrator's memory and is lost on restart, so the orchestrator runs
  one replica

## What's where

| Path | What |
|---|---|
| `infrastructure/controllers/agent-sandbox/` | Agent Sandbox `v1.0.8`: CRDs (`Sandbox`, `SandboxTemplate`, `SandboxWarmPool`, `SandboxClaim`) and the controller with `--extensions`, in `agent-sandbox-system`. The release's `sandbox-with-extensions.yaml`, downloaded |
| `apps/agents/sandbox.yaml` | Namespace `agents` (Pod Security `restricted`) |
| `apps/agents/claude.yaml` | ConfigMap `agent-session`, ServiceAccount `coding-agent`, `SandboxTemplate` and `SandboxWarmPool` `claude` |
| `apps/agents/runtimeclass.yaml` | `RuntimeClass` `gvisor`: handler `runsc`, scheduled on the minis only |
| `apps/agent-vault/`, `apps/agents/token-reviewer.yaml` | Agent Vault proxy and Kubernetes Auth ([agent-vault](agent-vault.md#whats-where)) |
| `apps/airlock/secrets-sync.yaml` | Namespace `airlock`; syncs `/airlock` into `airlock-secrets-infisical` and `/ghcr` into the image pull secret `ghcr-pull` |
| `apps/airlock/orchestrator.yaml` | ServiceAccount, `Role` and `RoleBinding` in `agents`, Deployment, Service `airlock-orchestrator:8081`, NetworkPolicy |
| `apps/airlock/web.yaml` | Deployment, Service `airlock-web:8080` |
| `apps/airlock/httproute.yaml` | `airlock.home.jakehulberg.com` → `airlock-web:8080` |
| `apps/airlock/image-policy.yaml` | Flux `ImageRepository` and `ImagePolicy` for both images |
| Infisical Cloud | `/airlock` and `/ghcr` ([Secrets](#secrets)) |

## Secrets

| Secret | What | Used by |
|---|---|---|
| `OIDC_ISSUER_URL` | `https://auth.home.jakehulberg.com/application/o/airlock/` | web |
| `OIDC_CLIENT_ID`, `OIDC_CLIENT_SECRET` | From the Authentik provider | web |
| `AIRLOCK_SESSION_KEY` | Encrypts the session cookie. `openssl rand -hex 32` | web |
| `AIRLOCK_API_TOKEN` | Web's credential for the orchestrator. `openssl rand -hex 32` | web, orchestrator |
| `GHCR_TOKEN` | In `/ghcr`. ghcr.io token, built into the `ghcr-pull` pull secret | both Deployments |

The first four are in Infisical Cloud `/airlock`:

- Web loads all of them with `envFrom`
- The orchestrator gets only `AIRLOCK_API_TOKEN`
- `AIRLOCK_PUBLIC_URL` and `AIRLOCK_ORCHESTRATOR_URL` aren't secret, so they're set in `web.yaml`

## Images

GitHub Actions in the Airlock repo builds `ghcr.io/jakehulberg/airlock-orchestrator` and
`ghcr.io/jakehulberg/airlock-web` on every push to `main`, for amd64 and arm64. Both images share
a tag, `YYYYMMDDTHHMMSS-<sha7>`.

Every 5 minutes, Flux image automation checks ghcr.io and commits the newest tag to both
Deployments. The packages are private, so both Deployments pull with `ghcr-pull`. The
`ImageRepository`s use the `ghcr-pull` in `flux-system`, which
`apps/life-dashboard/secrets-sync.yaml` creates.

## Isolation

| Layer | How |
|---|---|
| Kubernetes API | The orchestrator's Role covers only the `agents` namespace: claims, pools, sandboxes, pods, `pods/exec`. Web has no ServiceAccount token. Sandboxes have `automountServiceAccountToken: false` |
| Orchestrator | NetworkPolicy: only pods labeled `app: airlock-web` may reach port 8081 |
| Sandbox network | The template's `networkPolicy` denies ingress and allows egress only to DNS, the Agent Vault proxy and Infisical ([agent-vault](agent-vault.md#network)). k3s enforces NetworkPolicies |
| Credentials | None in the agent container. The Agent Vault proxy adds the API key on the way out |
| Kernel | gVisor (`runtimeClassName: gvisor`): the agent's syscalls go to gVisor's user-space kernel, not the VM's ([gVisor](#gvisor)) |
| Sandbox pod | Non-root, `RuntimeDefault` seccomp, no privilege escalation, all capabilities dropped, read-only root filesystem on `agent`, memory limits. The namespace enforces Pod Security `restricted` |

## The sandbox template

`claude` runs `ghcr.io/jakehulberg/agent-claude` from
[agent-images](https://github.com/jakehulberg/agent-images), plus a `session` sidecar. The sidecar
opens an Agent Vault session and writes the proxy settings to `/session/env`
([agent-vault](agent-vault.md#how-it-fits-together)).

The pool's task command sources `/session/env` and runs
`claude -p --dangerously-skip-permissions`. Permission prompts are off because the sandbox is the
boundary. If `~/.claude/projects` exists from an earlier task, the command adds `--continue`, so
one claim is one conversation.

Airlock reads these pool annotations:

| Annotation | What |
|---|---|
| `airlock.jakehulberg.com/description` | Shown in the claim form |
| `airlock.jakehulberg.com/agent-container` | Container to exec into (default `agent`) |
| `airlock.jakehulberg.com/task-command` | Run with `sh -c`, prompt on stdin |

## gVisor

`RuntimeClass` `gvisor` maps to the `runsc` runtime in containerd. Pods without
`runtimeClassName: gvisor` keep the node's default runtime, runc.

- `runsc` is installed by hand in each mini's VM ([bootstrap](bootstrap.md#11-gvisor-on-the-minis)),
  so it isn't in git
- The RuntimeClass's node selector, `node-role.kubernetes.io/control-plane: "true"`, keeps
  sandboxes on the minis, because the Pi has no `runsc`
- If a mini joins as an agent instead of a server, the selector won't match it. Label the nodes
  that have `runsc` and select on that label instead
- All containers in a sandbox pod share one gVisor kernel

## Resources

Controller, web and orchestrator are small Go programs, under 100MiB each. Each claimed sandbox
is a pod with limits of 1 CPU and 1GiB for `agent`, plus 200m and 128Mi for `session`.

## Checking it

```sh
kubectl get pods -n agent-sandbox-system                    # controller Running
kubectl get sandboxwarmpool,sandboxclaim,sandbox,pods -n agents  # one claim and pod per session
kubectl get pods -n airlock                                 # web and orchestrator 1/1 Running
kubectl logs -n airlock deploy/airlock-orchestrator         # claims and tasks, as JSON
kubectl exec -n agents <sandbox-pod> -c agent -- uname -r   # 4.4.0, gVisor's kernel
kubectl exec -n agents <sandbox-pod> -c agent -- dmesg      # gVisor boot lines; refused on runc
```
