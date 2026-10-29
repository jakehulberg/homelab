# Secrets

**Infisical Cloud is the single source of truth.** Every secret lives once in the `homelab` project
(slug `homelab`, env `dev`) at `app.infisical.com`. The Infisical Operator syncs it into the
cluster; a change in Cloud lands within 60 seconds.

**SOPS covers only what the operator needs to log in:** its own machine identity credentials. The
age key that decrypts them is the one secret never in git.

```
Infisical Cloud (homelab project)
  /                  SOPS_AGE_KEY, K3S_CLUSTER_SECRET     (bootstrap; used by you, not synced)
  /cert-manager      CLOUDFLARE_API_TOKEN
  /r2                R2_ACCESS_KEY_ID, R2_SECRET_ACCESS_KEY
  /infisical         ENCRYPTION_KEY, AUTH_SECRET, REDIS_PASSWORD, REDIS_URL, SITE_URL
  /authentik         AUTHENTIK_SECRET_KEY
        │
        │  Infisical Operator, logged in as machine identity "homelab-cluster"
        ▼
Kubernetes Secrets, one per folder per app
```

## Synced secrets

| `InfisicalStaticSecret` (file) | From | Kubernetes Secret | Used by |
|---|---|---|---|
| `infrastructure/configs/cloudflare-token-sync.yaml` | `/cert-manager` | `cert-manager/cloudflare-api-token-infisical` | ClusterIssuer `letsencrypt` ([certificates](certificates.md)) |
| `apps/infisical/r2-sync.yaml` | `/r2` | `infisical/r2-credentials-infisical` | ObjectStore `r2` ([storage](storage.md#backups)) |
| `apps/infisical/infisical-secrets-sync.yaml` | `/infisical` | `infisical/infisical-secrets-infisical` | Infisical (`kubeSecretRef`), Redis ([infisical](infisical.md)) |
| `apps/authentik/secrets-sync.yaml` | `/authentik` | `authentik/authentik-secrets-infisical` | Authentik (`existingSecret`) ([authentik](authentik.md)) |

Database passwords aren't here; CloudNativePG generates them ([storage](storage.md#credentials)).

Synced Secrets are named `<name>-infisical`; keys match the secret names in Cloud.

## How the operator connects

| Piece | Where | What |
|---|---|---|
| Operator | `infrastructure/controllers/infisical-operator/` | Chart `secrets-operator` `0.11.15`, cluster-wide, namespace `infisical-operator-system` |
| Machine identity | Infisical Cloud | `homelab-cluster`: Universal Auth, no org-level access, **Viewer on `homelab` only** |
| `infisical-universal-auth` | `infrastructure/configs/infisical-universal-auth.sops.yaml` | `clientId` and `clientSecret`, SOPS-encrypted |
| `InfisicalConnection` `infisical-cloud` | `infrastructure/configs/infisical-cloud.yaml` | `address: https://app.infisical.com` |
| `InfisicalAuth` `homelab-cluster` | same file | Universal Auth using `infisical-universal-auth` |

Uses the `v1beta1` types (`InfisicalConnection`, `InfisicalAuth`, `InfisicalStaticSecret`); the
older `InfisicalSecret` is deprecated.

Not Kubernetes Auth: that needs Infisical Cloud to reach this tailnet-only API server, which takes
an Infisical Gateway (paid plan).

Keep the identity scoped to `homelab`: Viewer means read access to everything in the project.

```sh
kubectl get infisicalconnection,infisicalauth -n infisical-operator-system   # READY True
kubectl get infisicalstaticsecret -A                                         # one per sync
```

## Give an app secrets

1. **Infisical Cloud:** create a folder for the app in `homelab` (env `dev`) and add its secrets.
   Values can reference others in the same folder, e.g.
   `REDIS_URL = redis://:${REDIS_PASSWORD}@redis:6379`; the operator receives them expanded.
2. **Repo:** add `apps/<app>/<app>-secrets-sync.yaml` and list it in the app's `kustomization.yaml`:

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

3. **Point the app at it** via `envFrom`, `secretKeyRef`, or the chart's equivalent.

A sync copies a whole folder, so one folder per app. Never sync `/`; it holds the bootstrap secrets.

### Migrating a secret safely

To replace a Secret something already uses: add the sync, compare fingerprints of both Secrets,
switch the consumer, delete the old one. Fingerprints identify values without printing them:

```sh
for s in <old> <new>; do echo "== $s"; kubectl get secret $s -n <ns> -o json | python3 -c 'import sys,json,hashlib,base64; d=json.load(sys.stdin)["data"]; [print(k, hashlib.sha256(base64.b64decode(v)).hexdigest()[:12]) for k,v in sorted(d.items())]'; done
```

## Infisical's own keys

`ENCRYPTION_KEY` and `AUTH_SECRET` encrypt everything in the cluster's Infisical. Without them even a
perfect database backup is unreadable. They live in Cloud at `/infisical`.

**Never change `ENCRYPTION_KEY` once the cluster's Infisical has data.**

Generated straight into Cloud, never shown on screen:

```sh
infisical secrets folders create --name infisical --path / --env=dev
infisical secrets set --env=dev --path=/infisical \
  ENCRYPTION_KEY=$(openssl rand -hex 16) \
  AUTH_SECRET=$(openssl rand -base64 32) \
  REDIS_PASSWORD=$(openssl rand -hex 24)
```

## SOPS

Used only for `infrastructure/configs/infisical-universal-auth.sops.yaml`.

- `.sops.yaml`: files ending in `.sops.yaml` get `data` / `stringData` values encrypted with our age
  public key. Names and metadata stay readable.
- Encrypting needs only the public key, so anyone with the repo can.
- Decrypting needs the private key: Flux has it in `sops-age`
  ([bootstrap step 9](bootstrap.md#9-sops-key)); Cloud has it as `SOPS_AGE_KEY`.
- `infrastructure`, `infra-configs` and `apps` all have `decryption` on.

### Rotate the operator's credentials

In Infisical Cloud, create a new Client Secret for `homelab-cluster` (optionally restricted to your
home's public IP). From the repo root, paste each value when prompted (no echo):

```sh
read -s CLIENT_ID && read -s CLIENT_SECRET
kubectl create secret generic infisical-universal-auth -n infisical-operator-system \
  --from-literal=clientId="$CLIENT_ID" --from-literal=clientSecret="$CLIENT_SECRET" \
  --dry-run=client -o yaml > infrastructure/configs/infisical-universal-auth.sops.yaml
sops -e -i infrastructure/configs/infisical-universal-auth.sops.yaml
```

The file is plain text between those two commands. Commit, push, revoke the old Client Secret. A
lost Client Secret is replaced this way, never recovered, so it needs no backup copy.

### Read or edit a SOPS file

```sh
infisical run --env=dev --path=/ -- sops -d infrastructure/configs/infisical-universal-auth.sops.yaml
```

## Troubleshooting

- **`decryption failed ... failed to load age identities`** in `flux get kustomizations`:
  `sops-age` is missing or empty. Check:

  ```sh
  kubectl get secret sops-age -n flux-system -o jsonpath='{.data.age\.agekey}' | base64 -d | head -c 17; echo
  ```

  Should print `AGE-SECRET-KEY-1`. If not, delete it and redo [bootstrap step 9](bootstrap.md#9-sops-key).
- **`InfisicalAuth` not ready:** credentials wrong or revoked, or the identity isn't in `homelab`.
  `kubectl describe infisicalauth homelab-cluster -n infisical-operator-system`.
- **Synced Secret missing or stale:** `kubectl describe infisicalstaticsecret <name> -n <ns>`; check
  the folder path and that the identity can read it.
- **Trailing newline in a value:** `infisical secrets get --plain` output ends in one; strip with
  `tr -d '\n'`. Synced values are unaffected.
