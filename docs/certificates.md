# Certificates

One wildcard Let's Encrypt certificate for `*.home.jakehulberg.com`, renewed by cert-manager, covers
every app. Nothing is exposed: ownership is proved via DNS, and the records point at Tailscale IPs.

## Why a real domain

Browsers trust only public CAs, which only issue for domains you own. A made-up name
(`infisical.home`) would need a private CA installed on every device.

A wildcard also keeps app names out of the public certificate logs.

## How it fits together

```
Cloudflare DNS: *.home.jakehulberg.com → 4 Tailscale IPs        (set by hand, once)
Cloudflare API token (DNS edit, jakehulberg.com only)           (Infisical Cloud, synced by the operator)
cert-manager                                                    (infrastructure/controllers/cert-manager)
ClusterIssuer letsencrypt: "get certs from Let's Encrypt,       (infrastructure/configs/cluster-issuer.yaml)
  prove ownership via Cloudflare DNS"
Certificate home-wildcard: "*.home.jakehulberg.com"             (infrastructure/configs/certificate.yaml)
  → Secret home-wildcard-tls in the traefik namespace           (created by cert-manager, never in git)
  → used by the Gateway's HTTPS listener (websecure, in the Traefik HelmRelease values)
```

### Issuing (DNS-01)

1. cert-manager requests `*.home.jakehulberg.com`.
2. Let's Encrypt asks for a TXT record at `_acme-challenge.home.jakehulberg.com`.
3. cert-manager creates it via the Cloudflare API and waits until public DNS shows it.
4. Let's Encrypt verifies and issues; cert-manager deletes the record.
5. The cert and key land in `home-wildcard-tls`.

~30 days before expiry this repeats, overwriting the same Secret. Traefik reloads it automatically.

## DNS records

Cloudflare → jakehulberg.com → DNS → Records: four `A` records named `*.home`, one per node
Tailscale IP, **DNS only** (grey cloud). Cloudflare's proxy can't reach Tailscale IPs.

With several records, browsers fall back to another node if one is down (after a connect timeout).

```sh
dig +short podinfo.home.jakehulberg.com   # the 4 Tailscale IPs
```

## Cloudflare API token

- Cloudflare: My Profile → API Tokens → "Edit zone DNS" template, zone `jakehulberg.com` only, no
  expiry. Named `homelab cert-manager DNS-01`.
- Infisical: `CLOUDFLARE_API_TOKEN` (homelab project, env `dev`, folder `/cert-manager`).
- Cluster: the operator syncs it into `cert-manager/cloudflare-api-token-infisical`
  (`infrastructure/configs/cloudflare-token-sync.yaml`); the ClusterIssuer reads key
  `CLOUDFLARE_API_TOKEN` ([secrets](secrets.md)).

Rotate: new token in Cloudflare, update Infisical Cloud, wait 60 seconds, delete the old token.

## Check it

```sh
kubectl get clusterissuer                                  # letsencrypt READY True
kubectl get certificate -n traefik                         # home-wildcard READY True
kubectl describe certificate home-wildcard -n traefik      # Not After, Renewal Time
```

## Troubleshooting

During issuance cert-manager creates an Order and a Challenge. The Challenge says what's wrong:

```sh
kubectl get certificate,order,challenge -n traefik
kubectl describe challenge -n traefik     # Reason: and Events:
```

| Challenge says | Cause | Fix |
|---|---|---|
| `the Cloudflare API token is invalid (does the API token contain a newline?)` | Trailing newline in the token (happened when piping `infisical secrets get --plain` into a SOPS file) | Remove it in Infisical Cloud; the Secret updates within 60 seconds |
| `Found recursive CNAME record ... when looking up _acme-challenge...` | cert-manager checked via cluster DNS (VMs + Tailscale), which answers wrongly | Already fixed: HelmRelease sets `dns01RecursiveNameservers: 1.1.1.1:53,8.8.8.8:53` and `dns01RecursiveNameserversOnly: true` |

cert-manager retries on a backoff that can take minutes. To retry now:

```sh
kubectl delete challenge -n traefik --all    # recreated immediately
```

`infra-configs` has `wait: true`, so a failing certificate blocks new pushes until it times out (10
minutes). Token fixes don't need a push: update Infisical Cloud and it syncs within 60 seconds.
