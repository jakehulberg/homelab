# Certificates

Every app gets HTTPS from one wildcard certificate for `*.home.jakehulberg.com`, issued by Let's
Encrypt and renewed automatically by cert-manager. Nothing is exposed to the internet: ownership is
proved through DNS, and the DNS records point at Tailscale IPs only the tailnet can reach.

## Why a real domain

Browsers only trust certificates from public authorities like Let's Encrypt, and those only issue
for domains you can prove you own. A made-up name (`infisical.home`) can't get one, which would mean
running a private certificate authority and installing it on every device.

A wildcard also keeps the app list private: issued certificates are published in public logs, and a
wildcard shows only `*.home.jakehulberg.com`.

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

1. cert-manager asks Let's Encrypt for a certificate for `*.home.jakehulberg.com`.
2. Let's Encrypt says: create a TXT record at `_acme-challenge.home.jakehulberg.com` with this value.
3. cert-manager creates it through the Cloudflare API, then checks public DNS until it's visible.
4. Let's Encrypt checks it, issues the certificate, and cert-manager deletes the TXT record.
5. cert-manager stores the certificate and key in `home-wildcard-tls`.

About 30 days before expiry, cert-manager repeats this and overwrites the same Secret. Traefik
picks up the new one automatically.

## DNS records

In Cloudflare, jakehulberg.com → DNS → Records, four `A` records named `*.home`, one per node
Tailscale IP, all **DNS only** (grey cloud). Cloudflare's proxy runs on the public internet and
can't reach Tailscale IPs.

Multiple records means browsers try another node if one is down (after a connect timeout).

```sh
dig +short podinfo.home.jakehulberg.com   # should print the 4 Tailscale IPs
```

## Cloudflare API token

- Cloudflare: My Profile → API Tokens → "Edit zone DNS" template, zone `jakehulberg.com` only,
  no expiration. Named `homelab cert-manager DNS-01`.
- Infisical: `CLOUDFLARE_API_TOKEN` (homelab project, env `dev`, folder `/cert-manager`).
- Cluster: the Infisical Operator syncs `/cert-manager` into the Secret
  `cert-manager/cloudflare-api-token-infisical` (`infrastructure/configs/cloudflare-token-sync.yaml`).
  The ClusterIssuer reads its `CLOUDFLARE_API_TOKEN` key ([secrets](secrets.md)).

To rotate it, create a new token in Cloudflare and update the value in Infisical Cloud. The cluster
picks it up within 60 seconds; then delete the old token in Cloudflare.

## Check it

```sh
kubectl get clusterissuer                                  # letsencrypt READY True
kubectl get certificate -n traefik                         # home-wildcard READY True
kubectl describe certificate home-wildcard -n traefik      # Not After, Renewal Time
```

## Troubleshooting

While a certificate is being issued, cert-manager creates an Order and a Challenge. The Challenge
says what's wrong:

```sh
kubectl get certificate,order,challenge -n traefik
kubectl describe challenge -n traefik     # Reason: and Events:
```

| Challenge says | Cause | Fix |
|---|---|---|
| `the Cloudflare API token is invalid (does the API token contain a newline?)` | Token stored with a trailing newline (it happened when the token was piped from `infisical secrets get --plain` into a SOPS file) | Make sure the value in Infisical Cloud has no trailing newline; the synced Secret updates within 60 seconds |
| `Found recursive CNAME record ... when looking up _acme-challenge...` | cert-manager checked the record through the cluster's DNS (VMs + Tailscale), which answers wrongly | Already fixed: the cert-manager HelmRelease sets `dns01RecursiveNameservers: 1.1.1.1:53,8.8.8.8:53` and `dns01RecursiveNameserversOnly: true` |

After fixing a cause, cert-manager retries on a backoff that can take minutes. To retry now:

```sh
kubectl delete challenge -n traefik --all    # cert-manager creates a fresh one immediately
```

`infra-configs` has `wait: true`, so while a certificate is failing it holds new pushes until it
times out (10 minutes). A fix to the token itself doesn't need a push: update it in Infisical Cloud
and the operator syncs it within 60 seconds.
