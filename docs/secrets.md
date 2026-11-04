# Secrets

Infisical Cloud is the single source of truth for secrets. Every secret lives once in the
`homelab` project (slug `homelab`, env `dev`) at `app.infisical.com`, and the Infisical Operator
syncs each folder into a Kubernetes Secret. A change in Cloud lands in the cluster within 60
seconds.

SOPS covers only the operator's own login: its machine identity credentials. The age key that
decrypts them is the one secret that's never in git.

```
Infisical Cloud, project homelab, env dev
  /                         SOPS_AGE_KEY, K3S_CLUSTER_SECRET   (bootstrap, read by hand, not synced)
  /cert-manager             CLOUDFLARE_API_TOKEN
  /r2                       R2_ACCESS_KEY_ID, R2_SECRET_ACCESS_KEY
  /infisical                ENCRYPTION_KEY, AUTH_SECRET, REDIS_PASSWORD, REDIS_URL, SITE_URL
  /authentik                AUTHENTIK_SECRET_KEY
  /airlock                  OIDC_ISSUER_URL, OIDC_CLIENT_ID, OIDC_CLIENT_SECRET,
                            AIRLOCK_SESSION_KEY, AIRLOCK_API_TOKEN
  /ghcr                     GHCR_TOKEN
  /agent-vault              INFISICAL_AGENT_VAULT_PROXY_TOKEN
  /life-dashboard           app settings (weather, YouTube, GitHub, Plaid)
  /litestream               LITESTREAM_SSE_C_KEY
  /monitoring               GRAFANA_ADMIN_USER, GRAFANA_ADMIN_PASSWORD
  /monitoring/alertmanager  NTFY_URL
  /kagent                   ANTHROPIC_API_KEY                  (experiment)
  /agentgateway             Authorization                      (experiment)
        │
        │  Infisical Operator, logged in as machine identity homelab-cluster
        ▼
Kubernetes Secrets in each app's namespace (see Synced secrets)
```

## Synced secrets

| `InfisicalStaticSecret` (file) | From | Kubernetes Secret | Used by |
|---|---|---|---|
| `infrastructure/configs/cloudflare-token-sync.yaml` | `/cert-manager` | `cert-manager/cloudflare-api-token-infisical` | ClusterIssuer `letsencrypt` ([certificates](certificates.md)) |
| `apps/infisical/r2-sync.yaml` | `/r2` | `infisical/r2-credentials-infisical` | ObjectStore `r2` ([storage](storage.md#backups)) |
| `apps/infisical/infisical-secrets-sync.yaml` | `/infisical` | `infisical/infisical-secrets-infisical` | Infisical (`kubeSecretRef`), Redis ([infisical](infisical.md)) |
| `apps/authentik/r2-sync.yaml` | `/r2` | `authentik/r2-credentials-authentik` | ObjectStore `r2` ([storage](storage.md#backups)) |
| `apps/authentik/secrets-sync.yaml` | `/authentik` | `authentik/authentik-secrets-infisical` | Authentik (`existingSecret`) ([authentik](authentik.md)) |
| `apps/airlock/secrets-sync.yaml` | `/airlock` | `airlock/airlock-secrets-infisical` | Airlock web (all), orchestrator (`AIRLOCK_API_TOKEN`) ([airlock](airlock.md#secrets)) |
| `apps/airlock/secrets-sync.yaml` | `/ghcr` | `airlock/ghcr-pull` | `imagePullSecrets` of Airlock web and orchestrator |
| `apps/agent-vault/secrets-sync.yaml` | `/agent-vault` | `agent-vault/agent-vault-secrets-infisical` | Agent Vault proxy (`envFrom`) ([agent-vault](agent-vault.md)) |
| `apps/life-dashboard/secrets-sync.yaml` | `/life-dashboard` | `life-dashboard/life-dashboard-secrets-infisical` | life-dashboard app (`envFrom`) |
| `apps/life-dashboard/secrets-sync.yaml` | `/ghcr` | `life-dashboard/ghcr-pull` | `imagePullSecrets` of life-dashboard |
| `apps/life-dashboard/secrets-sync.yaml` | `/ghcr` | `flux-system/ghcr-pull` | Flux `ImageRepository` objects for life-dashboard and Airlock |
| `apps/life-dashboard/litestream.yaml` | `/r2` and `/litestream` | `life-dashboard/litestream-infisical` | Litestream sidecar and restore init containers ([storage](storage.md#sqlite-backups-litestream-life-dashboard)) |
| `apps/monitoring/secrets-sync.yaml` | `/monitoring` | `monitoring/grafana-admin` | Grafana admin login (`existingSecret`) |
| `apps/monitoring/secrets-sync.yaml` | `/monitoring/alertmanager` | `monitoring/alertmanager-ntfy` | Alertmanager ntfy webhook (`url_file`) |
| `apps/experiments/kagent/secrets-sync.yaml` | `/kagent` | `kagent/kagent-anthropic` | kagent (`apiKeySecretRef`) |
| `apps/experiments/agentgateway/secrets-sync.yaml` | `/agentgateway` | `agentgateway-system/anthropic-secret` | `AgentgatewayBackend` `anthropic` |

Each key in a synced Secret matches the secret's name in Cloud. The `ghcr-pull` Secrets are the
exception: a `template` builds a `.dockerconfigjson` for `ghcr.io` from `GHCR_TOKEN`.

Database passwords aren't here: CloudNativePG generates them ([storage](storage.md#credentials)).

## How the operator connects

| Piece | Where | What |
|---|---|---|
| Operator | `infrastructure/controllers/infisical-operator/` | Chart `secrets-operator` `0.11.16`, cluster-wide, namespace `infisical-operator-system` |
| Machine identity | Infisical Cloud | `homelab-cluster`: Universal Auth, no org-level access, Viewer on `homelab` only |
| `infisical-universal-auth` | `infrastructure/configs/infisical-universal-auth.sops.yaml` | `clientId` and `clientSecret`, SOPS-encrypted |
| `InfisicalConnection` `infisical-cloud` | `infrastructure/configs/infisical-cloud.yaml` | `address: https://app.infisical.com` |
| `InfisicalAuth` `homelab-cluster` | same file | Universal Auth using `infisical-universal-auth` |

- Keep the identity in `homelab` only: Viewer can read everything in the project
- Use the `v1beta1` types (`InfisicalConnection`, `InfisicalAuth`, `InfisicalStaticSecret`). The
  older `InfisicalSecret` is deprecated
- Not Kubernetes Auth: that needs Infisical Cloud to reach this tailnet-only API server, which
  takes an Infisical Gateway (paid plan)

```sh
kubectl get infisicalconnection,infisicalauth -n infisical-operator-system   # READY True
kubectl get infisicalstaticsecret -A                                         # one per sync
```

## Give an app secrets

1. **Infisical Cloud:** create a folder for the app in `homelab` (env `dev`) and add its secrets.
   Values can reference others in the same folder, e.g.
   `REDIS_URL = redis://:${REDIS_PASSWORD}@redis:6379`. The operator receives them expanded
2. **Repo:** add `apps/<app>/<app>-secrets-sync.yaml` and list it in the app's
   `kustomization.yaml`:

   ```yaml
   apiVersion: secrets.infisical.com/v1beta1
   kind: InfisicalStaticSecret
   metadata:
     name: <app>-secrets
     namespace: <app-namespace>
   spec:
     infisicalAuthRef:
       name: homelab-cluster
       namespace: infisical-operator-system
     syncOptions:
       refreshInterval: 60s
     sources:
       - projectSlug: homelab
         environmentSlug: dev
         secretPath: /<folder>
     targets:
       - name: <app>-secrets-infisical
         namespace: <app-namespace>
         kind: Secret
         secretType: Opaque
         creationPolicy: Owner
   ```

3. **App:** point the app at the Secret with `envFrom`, `secretKeyRef`, or the chart's equivalent

A sync copies a whole folder, so give each app its own folder. Never sync `/`: it holds the
bootstrap secrets.

### Migrating a secret safely

To replace a Secret that something already uses:

1. Add the sync
2. Compare fingerprints of the old and new Secrets with the command below. Fingerprints identify
   values without printing them
3. Switch the consumer to the new Secret
4. Delete the old Secret

```sh
for s in <old> <new>; do echo "== $s"; kubectl get secret $s -n <ns> -o json | python3 -c 'import sys,json,hashlib,base64; d=json.load(sys.stdin)["data"]; [print(k, hashlib.sha256(base64.b64decode(v)).hexdigest()[:12]) for k,v in sorted(d.items())]'; done   # same fingerprints for both
```

## Infisical's own keys

`ENCRYPTION_KEY` and `AUTH_SECRET` encrypt everything in the cluster's Infisical. Without them,
even a perfect database backup is unreadable. They live in Cloud at `/infisical`.

**Never change `ENCRYPTION_KEY` once the cluster's Infisical has data.**

To create the folder and generate the keys straight into Cloud, without showing them on screen:

```sh
infisical secrets folders create --name infisical --path / --env=dev
infisical secrets set --env=dev --path=/infisical \
  ENCRYPTION_KEY=$(openssl rand -hex 16) \
  AUTH_SECRET=$(openssl rand -base64 32) \
  REDIS_PASSWORD=$(openssl rand -hex 24)
```

## SOPS

SOPS encrypts only `infrastructure/configs/infisical-universal-auth.sops.yaml`.

- `.sops.yaml`: in files ending in `.sops.yaml`, SOPS encrypts the `data` and `stringData` values
  with the repo's age public key. Names and metadata stay readable
- Encrypting needs only the public key, so anyone with the repo can encrypt
- Decrypting needs the private key. Flux reads it from `sops-age`
  ([bootstrap step 9](bootstrap.md#9-sops-key)), and Cloud stores it as `SOPS_AGE_KEY` in `/`
- The `infrastructure`, `infra-configs` and `apps` Kustomizations all have `decryption` on

### Rotate the operator's credentials

1. In Infisical Cloud, create a new Client Secret for `homelab-cluster` (optionally restricted to
   your home's public IP)
2. From the repo root, run the commands below and paste each value when prompted (no echo). The
   file is plain text between the two commands:

   ```sh
   read -s CLIENT_ID && read -s CLIENT_SECRET
   kubectl create secret generic infisical-universal-auth -n infisical-operator-system \
     --from-literal=clientId="$CLIENT_ID" --from-literal=clientSecret="$CLIENT_SECRET" \
     --dry-run=client -o yaml > infrastructure/configs/infisical-universal-auth.sops.yaml
   sops -e -i infrastructure/configs/infisical-universal-auth.sops.yaml
   ```

3. Commit and push the file
4. In Infisical Cloud, revoke the old Client Secret

A lost Client Secret is replaced this way, never recovered, so it needs no backup copy.

### Read or edit a SOPS file

```sh
infisical run --env=dev --path=/ -- sops -d infrastructure/configs/infisical-universal-auth.sops.yaml
```

## Troubleshooting

- **`decryption failed ... failed to load age identities` in `flux get kustomizations`:**
  `sops-age` is missing or empty. If this check doesn't print `AGE-SECRET-KEY-1`, delete
  `sops-age` and redo [bootstrap step 9](bootstrap.md#9-sops-key):

  ```sh
  kubectl get secret sops-age -n flux-system -o jsonpath='{.data.age\.agekey}' | base64 -d | head -c 17; echo   # AGE-SECRET-KEY-1
  ```

- **`InfisicalAuth` not ready:** the credentials are wrong or revoked, or the identity isn't in
  `homelab`. Run `kubectl describe infisicalauth homelab-cluster -n infisical-operator-system`
- **Synced Secret missing or stale:** run `kubectl describe infisicalstaticsecret <name> -n <ns>`,
  then check the folder path and that the identity can read the folder
- **Trailing newline in a value:** `infisical secrets get --plain` output ends in a newline; strip
  it with `tr -d '\n'`. Synced values don't have one
