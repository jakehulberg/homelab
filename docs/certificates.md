# Certificates

One wildcard Let's Encrypt certificate for `*.home.jakehulberg.com` covers every app, and
cert-manager renews it. Let's Encrypt checks domain ownership through a DNS record, so nothing has
to be reachable from the internet, and the DNS records point at Tailscale IPs.

## Why a real domain

Browsers trust only public CAs, and public CAs issue only for domains you own. A made-up name
(`infisical.home`) would need a private CA installed on every device. A wildcard also keeps app
names out of the public certificate logs.

## How it fits together

```
Infisical Cloud /cert-manager ──(1) operator──▶ Secret cloudflare-api-token-infisical
                                                  │ (2) read by ClusterIssuer letsencrypt
                                                  ▼
cert-manager ──(3) TXT _acme-challenge.home.jakehulberg.com, Cloudflare API──▶ Cloudflare DNS
Let's Encrypt ──(4) checks the TXT record, issues *.home.jakehulberg.com──▶ cert-manager
cert-manager ──(5) writes──▶ Secret home-wildcard-tls (traefik) ──▶ Gateway listener websecure
```

1. The Infisical Operator syncs `CLOUDFLARE_API_TOKEN` from Infisical Cloud into
   `cert-manager/cloudflare-api-token-infisical` every 60 seconds ([secrets](secrets.md))
2. ClusterIssuer `letsencrypt` reads the token from key `CLOUDFLARE_API_TOKEN` of that Secret
3. For Certificate `home-wildcard`, cert-manager requests `*.home.jakehulberg.com`, creates the TXT
   record Let's Encrypt asks for through the Cloudflare API, and waits until public DNS shows it
4. Let's Encrypt checks the record and issues the certificate; cert-manager deletes the record
5. cert-manager writes the certificate and key to `home-wildcard-tls` in `traefik`, and the
   Gateway's `websecure` listener serves it

About 30 days before expiry, cert-manager repeats steps 3 to 5 and overwrites the same Secret.
Traefik reloads the new certificate automatically.

## What's where

| Path | What |
|---|---|
| `infrastructure/controllers/cert-manager/` | cert-manager chart `v1.21.3`, checking DNS-01 records against `1.1.1.1` and `8.8.8.8` |
| `infrastructure/configs/cluster-issuer.yaml` | ClusterIssuer `letsencrypt`, DNS-01 through Cloudflare |
| `infrastructure/configs/certificate.yaml` | Certificate `home-wildcard` in `traefik` |
| `infrastructure/configs/cloudflare-token-sync.yaml` | Syncs the Cloudflare token from Infisical Cloud |
| `infrastructure/controllers/traefik/traefik.yaml` | The `websecure` listener's `certificateRefs: home-wildcard-tls` |
| Cloudflare | DNS records and API token, set by hand |

## Secrets

| Secret | What | Used by |
|---|---|---|
| `cloudflare-api-token-infisical` (`cert-manager`) | `CLOUDFLARE_API_TOKEN`, from Infisical Cloud (homelab project, env `dev`, folder `/cert-manager`) | ClusterIssuer `letsencrypt` |
| `home-wildcard-tls` (`traefik`) | The issued certificate and key, created by cert-manager, never in git | Gateway listener `websecure` |

## DNS records

In Cloudflare, under jakehulberg.com → DNS → Records, there are four `A` records named `*.home`,
one per node Tailscale IP, set to **DNS only** (grey cloud). Cloudflare's proxy can't reach
Tailscale IPs.

With several records, if one node is down, browsers try another IP after a connect timeout.

```sh
dig +short podinfo.home.jakehulberg.com   # the 4 Tailscale IPs
```

## Cloudflare API token

The token is `homelab cert-manager DNS-01` in Cloudflare (My Profile → API Tokens), made from the
"Edit zone DNS" template, for zone `jakehulberg.com` only, with no expiry.

To rotate the token:

1. Create a new token in Cloudflare with the same settings
2. Replace `CLOUDFLARE_API_TOKEN` in Infisical Cloud `/cert-manager` with the new token
3. Wait 60 seconds for the operator to sync the new token into `cloudflare-api-token-infisical`
4. Delete the old token in Cloudflare

## Checking it

```sh
kubectl get clusterissuer                                  # letsencrypt READY True
kubectl get certificate -n traefik                         # home-wildcard READY True
kubectl describe certificate home-wildcard -n traefik      # Not After, Renewal Time
```

## Troubleshooting

During issuance, cert-manager creates an Order and a Challenge. The Challenge's `Reason:` and
`Events:` show the error:

```sh
kubectl get certificate,order,challenge -n traefik
kubectl describe challenge -n traefik     # Reason: and Events:
```

- **Challenge says `the Cloudflare API token is invalid (does the API token contain a
  newline?)`:** the token value ends in a newline, for example from piping
  `infisical secrets get --plain` into a file. Remove the newline in Infisical Cloud; the Secret
  updates within 60 seconds
- **Challenge says `Found recursive CNAME record ... when looking up _acme-challenge...`:**
  cert-manager checked the record through cluster DNS (the VMs and Tailscale), which answers wrongly
  for it. The HelmRelease prevents this with `dns01RecursiveNameservers: 1.1.1.1:53,8.8.8.8:53` and
  `dns01RecursiveNameserversOnly: true`; check those values are still set
- **Certificate stays not Ready for minutes after a fix:** cert-manager retries on a backoff. To
  retry now, delete the Challenge, and cert-manager recreates it immediately:
  `kubectl delete challenge -n traefik --all`
- **New pushes don't apply while a certificate is failing:** `infra-configs` has `wait: true`, so a
  failing certificate holds the Kustomization until it times out after 10 minutes
  ([gitops](gitops.md#troubleshooting)). A token fix doesn't need a push: update Infisical Cloud,
  and the operator syncs the new token within 60 seconds
