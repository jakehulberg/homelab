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
- Both the `infrastructure` and `apps` Flux Kustomizations have `decryption` turned on.

## Add a secret

From the repo root. The file name must end in `.sops.yaml`.

**a. Write the Secret** (without saving it to shell history in plain text: `--from-file` or
`read -s` work too):

```sh
kubectl create secret generic my-secret -n my-app --from-literal=password=hunter2 \
  --dry-run=client -o yaml > apps/my-app/my-secret.sops.yaml
```

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
