# Secrets

Two kinds of secrets, for two jobs:

- **SOPS (in this repo):** secrets the cluster needs before Infisical is running, like Infisical's
  own database password. Encrypted in git, decrypted by Flux.
- **Infisical (later):** everything else. Apps pull secrets from Infisical through the Infisical
  Operator.

## How SOPS works here

- `.sops.yaml` says: any file ending in `.sops.yaml` gets its `data` / `stringData` values encrypted
  with our age public key. Names and metadata stay readable.
- Encrypting only needs the public key (in `.sops.yaml`), so anyone with the repo can add a secret.
- Decrypting needs the private key: Flux has it in the `sops-age` Secret
  ([bootstrap step 9](bootstrap.md#9-sops-key)), and you have it in Infisical as `SOPS_AGE_KEY`.
- The `infrastructure`, `infra-configs` and `apps` Flux Kustomizations all have `decryption`
  turned on.

## Secrets in the repo

| File | What | Source |
|---|---|---|
| `infrastructure/configs/cloudflare-api-token.sops.yaml` | Cloudflare token for cert-manager ([certificates](certificates.md)) | Infisical `CLOUDFLARE_API_TOKEN` |
| `apps/infisical/r2-credentials.sops.yaml` | R2 keys for database backups ([storage](storage.md#backups)) | Infisical Cloud, env `dev`, folder `/homelab/r2` |
| `apps/infisical/infisical-secrets.sops.yaml` | Infisical's `ENCRYPTION_KEY`, `AUTH_SECRET`, `REDIS_PASSWORD`, plus `REDIS_URL` and `SITE_URL` built from them | Infisical Cloud, env `dev`, folder `/homelab/infisical` |

Database passwords are not in this list: CloudNativePG generates them in the cluster
([storage](storage.md#credentials)).

Every secret here also lives in Infisical Cloud, which is the source of truth. The plan is to
replace these SOPS copies with the Infisical Operator syncing from Cloud (see `PLAN.md`, step 5).

### Infisical's own keys

`ENCRYPTION_KEY` and `AUTH_SECRET` encrypt everything stored in the cluster's Infisical. Lose them
and even a perfect database backup is unreadable, so the copy in Infisical Cloud matters.

They were generated straight into Cloud, never shown on screen:

```sh
infisical secrets folders create --name infisical --path /homelab --env=dev
infisical secrets set --env=dev --path=/homelab/infisical \
  ENCRYPTION_KEY=$(openssl rand -hex 16) \
  AUTH_SECRET=$(openssl rand -base64 32) \
  REDIS_PASSWORD=$(openssl rand -hex 24)
```

and turned into the SOPS file from the repo root, passing values only through environment
variables:

```sh
infisical run --env=dev --path=/homelab/infisical -- sh -c '
kubectl create secret generic infisical-secrets -n infisical \
  --from-literal=ENCRYPTION_KEY="$ENCRYPTION_KEY" \
  --from-literal=AUTH_SECRET="$AUTH_SECRET" \
  --from-literal=REDIS_PASSWORD="$REDIS_PASSWORD" \
  --from-literal=REDIS_URL="redis://:$REDIS_PASSWORD@redis:6379" \
  --from-literal=SITE_URL="https://infisical.home.jakehulberg.com" \
  --dry-run=client -o yaml' > apps/infisical/infisical-secrets.sops.yaml
sops -e -i apps/infisical/infisical-secrets.sops.yaml
```

Never regenerate `ENCRYPTION_KEY` once Infisical has data; rerun only the second command to rebuild
the file.

## Add a secret

From the repo root. The file name must end in `.sops.yaml`.

**a. Write the Secret.** For a value stored in Infisical, pipe it in so it never shows on screen
or in shell history:

```sh
infisical secrets get MY_SECRET --plain --env=dev --path=/homelab | tr -d '\n' \
  | kubectl create secret generic my-secret -n my-app --from-file=password=/dev/stdin \
    --dry-run=client -o yaml > apps/my-app/my-secret.sops.yaml
```

- `tr -d '\n'` strips the newline `infisical secrets get --plain` adds at the end. Without it the
  newline ends up inside the Secret, and many apps reject the value.
- If `infisical` fails, it pipes nothing and you get an empty Secret. Check the value isn't empty
  after encrypting (the `ENC[AES256_GCM,data:...` part should have characters after `data:`).
- The file is plain text until step b, so run b immediately.

**b. Encrypt it in place** before it's ever committed:

```sh
sops -e -i apps/my-app/my-secret.sops.yaml
```

The values now read `ENC[...]`.

**c. List it** in that folder's `kustomization.yaml`, then commit and push.

## Edit or read a secret

Needs the private key, pulled from Infisical:

```sh
infisical run --env=dev --path=/homelab -- sops apps/my-app/my-secret.sops.yaml          # edit
infisical run --env=dev --path=/homelab -- sops -d apps/my-app/my-secret.sops.yaml       # print
```

## Troubleshooting

`flux get kustomizations` shows `decryption failed ... failed to load age identities`: the
`sops-age` Secret is missing or empty. Check it:

```sh
kubectl get secret sops-age -n flux-system -o jsonpath='{.data.age\.agekey}' | base64 -d | head -c 17; echo
```

It should print `AGE-SECRET-KEY-1`. If not, delete it and redo [bootstrap step 9](bootstrap.md#9-sops-key).
