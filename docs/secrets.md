# Secrets

**Infisical Cloud is the single source of truth.** Every secret lives once, in the `homelab`
project (slug `homelab`, env `dev`) at `app.infisical.com`, and the Infisical Operator syncs it into
the cluster. Rotating a value in Cloud updates the cluster within 60 seconds.

**SOPS covers only what's needed before the operator can log in:** the operator's own machine
identity credentials. The age key that decrypts them is the one secret never in git.

```
Infisical Cloud (homelab project)
  /                  SOPS_AGE_KEY, K3S_CLUSTER_SECRET     (bootstrap; used by you, not synced)
  /cert-manager      CLOUDFLARE_API_TOKEN
  /r2                R2_ACCESS_KEY_ID, R2_SECRET_ACCESS_KEY
  /infisical         ENCRYPTION_KEY, AUTH_SECRET, REDIS_PASSWORD, REDIS_URL, SITE_URL
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

Database passwords aren't here: CloudNativePG generates them in the cluster
([storage](storage.md#credentials)).

Synced Secrets are named `<name>-infisical`, and their keys are the secret names from Cloud.

## How the operator connects

| Piece | Where | What |
|---|---|---|
| Operator | `infrastructure/controllers/infisical-operator/` | Helm chart `secrets-operator` `0.11.15`, cluster-wide, namespace `infisical-operator-system` |
| Machine identity | Infisical Cloud | `homelab-cluster`: Universal Auth, no org-level access, **Viewer on the `homelab` project only** |
| `infisical-universal-auth` | `infrastructure/configs/infisical-universal-auth.sops.yaml` | The identity's `clientId` and `clientSecret`, SOPS-encrypted |
| `InfisicalConnection` `infisical-cloud` | `infrastructure/configs/infisical-cloud.yaml` | `address: https://app.infisical.com` |
| `InfisicalAuth` `homelab-cluster` | same file | Universal Auth using `infisical-universal-auth` |

Uses the operator's `v1beta1` types (`InfisicalConnection`, `InfisicalAuth`,
`InfisicalStaticSecret`); the older `InfisicalSecret` is deprecated.

Why Universal Auth and not Kubernetes Auth: Kubernetes Auth needs Infisical Cloud to reach this
cluster's API server, which is only on the tailnet. That takes an Infisical Gateway, which needs a
paid plan.

The identity sees only the `homelab` project. Keep it that way: Viewer on a project means read
access to everything in it.

```sh
kubectl get infisicalconnection,infisicalauth -n infisical-operator-system   # READY True
kubectl get infisicalstaticsecret -A                                         # one per sync
```

## Give an app secrets

1. **In Infisical Cloud:** create a folder for the app in the `homelab` project (env `dev`) and add
   its secrets. A value can reference another in the same folder, e.g.
   `REDIS_URL = redis://:${REDIS_PASSWORD}@redis:6379`; the operator receives it expanded.
2. **In the repo:** add `apps/<app>/<app>-secrets-sync.yaml` and list it in the app's
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

3. **Point the app at it** with `envFrom`, `secretKeyRef`, or the chart's equivalent.

One sync copies a whole folder, so give each app its own folder; never point a sync at `/`, which
holds the bootstrap secrets.

### Migrating a secret safely

To swap a Secret something already depends on: add the sync first, compare the two Secrets by
fingerprint, then switch the consumer and delete the old one. The fingerprints identify each value
without printing it:

```sh
for s in <old> <new>; do echo "== $s"; kubectl get secret $s -n <ns> -o json | python3 -c 'import sys,json,hashlib,base64; d=json.load(sys.stdin)["data"]; [print(k, hashlib.sha256(base64.b64decode(v)).hexdigest()[:12]) for k,v in sorted(d.items())]'; done
```

## Infisical's own keys

`ENCRYPTION_KEY` and `AUTH_SECRET` encrypt everything stored in the cluster's Infisical. Lose them
and even a perfect database backup is unreadable. They live in Cloud at `/infisical`.

**Never change `ENCRYPTION_KEY` once the cluster's Infisical has data.**

They were generated straight into Cloud, never shown on screen:

```sh
infisical secrets folders create --name infisical --path / --env=dev
infisical secrets set --env=dev --path=/infisical \
  ENCRYPTION_KEY=$(openssl rand -hex 16) \
  AUTH_SECRET=$(openssl rand -base64 32) \
  REDIS_PASSWORD=$(openssl rand -hex 24)
```

## SOPS

Used only for `infrastructure/configs/infisical-universal-auth.sops.yaml`.

- `.sops.yaml` says: any file ending in `.sops.yaml` gets its `data` / `stringData` values encrypted
  with our age public key. Names and metadata stay readable.
- Encrypting only needs the public key, so anyone with the repo can encrypt.
- Decrypting needs the private key: Flux has it in the `sops-age` Secret
  ([bootstrap step 9](bootstrap.md#9-sops-key)), and it's in Cloud as `SOPS_AGE_KEY`.
- The `infrastructure`, `infra-configs` and `apps` Flux Kustomizations all have `decryption`
  turned on.

### Rotate the operator's credentials

In Infisical Cloud, create a new Client Secret for `homelab-cluster` (optionally restricted to your
home's public IP). Then from the repo root, pasting each value when prompted (nothing shows on
screen):

```sh
read -s CLIENT_ID && read -s CLIENT_SECRET
kubectl create secret generic infisical-universal-auth -n infisical-operator-system \
  --from-literal=clientId="$CLIENT_ID" --from-literal=clientSecret="$CLIENT_SECRET" \
  --dry-run=client -o yaml > infrastructure/configs/infisical-universal-auth.sops.yaml
sops -e -i infrastructure/configs/infisical-universal-auth.sops.yaml
```

The file is plain text between those two commands. Commit, push, then revoke the old Client Secret.
A lost Client Secret is never recovered, only replaced this way, so it needs no copy anywhere else.

### Read or edit a SOPS file

```sh
infisical run --env=dev --path=/ -- sops -d infrastructure/configs/infisical-universal-auth.sops.yaml
```

## Troubleshooting

- **`decryption failed ... failed to load age identities`** in `flux get kustomizations`: the
  `sops-age` Secret is missing or empty. Check it:

  ```sh
  kubectl get secret sops-age -n flux-system -o jsonpath='{.data.age\.agekey}' | base64 -d | head -c 17; echo
  ```

  It should print `AGE-SECRET-KEY-1`. If not, delete it and redo [bootstrap step 9](bootstrap.md#9-sops-key).
- **`InfisicalAuth` not ready:** the identity's credentials are wrong or revoked, or it isn't a
  member of the `homelab` project. `kubectl describe infisicalauth homelab-cluster -n infisical-operator-system`.
- **Synced Secret missing or stale:** `kubectl describe infisicalstaticsecret <name> -n <ns>`. Check
  the folder path and that the identity can read it.
- **A value has a trailing newline:** values piped from `infisical secrets get --plain` end in one.
  Strip it with `tr -d '\n'`. Synced values don't have this problem.
