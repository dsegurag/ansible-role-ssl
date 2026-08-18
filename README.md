# Ansible Role: Let's Encrypt SSL Certificate with Cloudflare DNS Verification

This Ansible role installs and configures [certbot](https://certbot.eff.org/) so a host
issues its own TLS certificate from an ACME CA ([Let's Encrypt](https://letsencrypt.org/)
by default) and **renews it unattended**, solving the DNS-01 challenge through
[Cloudflare](https://www.cloudflare.com/) DNS.

Renewal is handled on the instance by certbot's own systemd timer, which runs twice a
day. Ansible is only needed for the initial setup: once the role has run, the host keeps
its certificate valid on its own.

Because the challenge is solved over DNS, a single certificate can cover the apex and
every subdomain (`*.example.com`), and it is reusable by any service rather than locked
inside one reverse proxy.

## Features

- Installs `certbot` and the Cloudflare DNS plugin from the distribution repositories.
- Issues a certificate covering every name in `ssl_domains`, wildcards included.
- Leaves renewal to `certbot.timer` - no scheduled Ansible run required.
- Optionally reloads services after each successful renewal via a deploy hook.
- Re-running the role is a no-op unless the domain list changed or the certificate is
  due for renewal.

## Requirements

- `ansible-core` 2.18 or higher. No external collections.
- A Debian or Ubuntu host with systemd. `certbot` and `python3-certbot-dns-cloudflare`
  come from the distribution repositories (Debian bookworm ships certbot 2.0, trixie 4.0).
- A Cloudflare API token with permission to edit DNS in the zone holding the challenge
  records - the "Edit zone DNS" template (`Zone:DNS:Edit` + `Zone:Read`) scoped to that
  zone. Keep it out of plain text, for example with `ansible-vault encrypt_string`.

### About the token on the host

Unattended DNS-01 renewal requires the token to live on the instance, at
`{{ ssl_credentials_file }}` (`0600 root:root`). This is inherent to renewing without a
human or a control node present, not specific to certbot. To limit the blast radius:

- Scope the token to the single zone it needs, never an account-wide key.
- Or delegate validation: `CNAME _acme-challenge.example.com` to a record in a throwaway
  zone, and issue a token that can only edit that zone. A compromised host then cannot
  touch your real DNS records.

## Role Variables

### Required

| Variable | Description |
| - | - |
| `ssl_domains` | List of domain names for the certificate, apex first, e.g. `["example.com", "*.example.com"]`. |
| `ssl_cloudflare_token` | Cloudflare API token with permission to manage DNS records in the zone. |

### Optional

| Variable | Description | Default |
| - | - | - |
| `ssl_cert_name` | Certbot lineage name; decides the output directory. | First entry of `ssl_domains`, with a leading `*.` stripped |
| `ssl_email` | Address notified when the certificate approaches expiry. | `admin@{{ ssl_cert_name }}` |
| `ssl_acme_directory` | ACME endpoint. Point it at staging while testing. | `https://acme-v02.api.letsencrypt.org/directory` |
| `ssl_key_type` | `ecdsa` or `rsa`. | `ecdsa` |
| `ssl_rsa_key_size` | Key size, only used when `ssl_key_type` is `rsa`. | `4096` |
| `ssl_dns_propagation_seconds` | Seconds to wait for the TXT records to propagate before validation. | `30` |
| `ssl_credentials_file` | Where the Cloudflare token is stored on the host. | `/etc/letsencrypt/cloudflare.ini` |
| `ssl_auto_renew` | Enable and start `certbot.timer`. | `true` |
| `ssl_renew_days` | Renew when fewer than this many days remain. Empty keeps certbot's default (a third of the lifetime). | `""` |
| `ssl_reload_services` | Units to `systemctl reload` after a successful renewal. | `[]` |
| `ssl_deploy_hook_command` | Extra command to run after a successful renewal. | `""` |
| `ssl_dir` | Directory holding the issued material. | `/etc/letsencrypt/live/{{ ssl_cert_name }}` |
| `ssl_privatekey`, `ssl_cert`, `ssl_chain`, `ssl_fullchain` | Individual output paths. | see below |

Set `ssl_cert_name` explicitly if the derived default is not what you want; changing it
later moves the output directory, so pick it once.

## Example Playbook

```yaml
- hosts: webserver
  become: true
  roles:
    - role: dsegurag.ssl
      vars:
        ssl_domains:
          - example.com
          - "*.example.com"
        ssl_cloudflare_token: "{{ lookup('ansible.builtin.env', 'CLOUDFLARE_API_TOKEN') }}"
        ssl_email: admin@example.com
        ssl_reload_services:
          - nginx
```

## Issuing and renewal

Test against the staging endpoint first:

```yaml
ssl_acme_directory: https://acme-staging-v02.api.letsencrypt.org/directory
```

Let's Encrypt limits you to 5 duplicate certificates per week and a failed run still
counts, so staging is worth the extra round. Staging certificates are not trusted by
browsers; remove `/etc/letsencrypt` before the first production run.

Verify unattended renewal once, on the host:

```bash
systemctl list-timers certbot.timer   # active, with a next trigger
certbot renew --dry-run               # full renewal cycle against the CA's staging
```

Re-running the role is a no-op while the certificate covers the same domains and is not
due for renewal. Adding or removing a name in `ssl_domains` does trigger a reissue.

## Output files

`/etc/letsencrypt/live/<ssl_cert_name>/`:

| File | Contents |
| - | - |
| `privkey.pem` | Certificate private key |
| `cert.pem` | Certificate |
| `chain.pem` | Intermediate certificates |
| `fullchain.pem` | Certificate plus intermediates - what most servers want |

Reference `ssl_fullchain` and `ssl_privatekey` rather than hardcoding these paths.

These are symlinks into `/etc/letsencrypt/archive/`, which matters for containers: bind
mount `/etc/letsencrypt` as a whole, not just `live/`, or the symlinks will dangle.

## Upgrading from 1.x

Version 2.0.0 replaces the in-Ansible ACME client with certbot, so the host renews on its
own. See the [CHANGELOG](CHANGELOG.md) for the full mapping; in short:

| 1.x | 2.0 |
| - | - |
| `ssl_cloudflare_domain` | `ssl_domains` (list), and `ssl_cert_name` if you want a different lineage name |
| `ssl_san_certificates` | folded into `ssl_domains`, without the `DNS:` prefix |
| `ssl_letsencrypt_email` | `ssl_email` |
| `ssl_cert_directory`, `ssl_key_directory` | `ssl_dir` and the per-file variables above |

Output moved to `/etc/letsencrypt/live/<name>/`. Certificates issued by 1.x are not
adopted; the role issues a fresh one on first run.

## Dependencies

- None.

## License

MIT License

## Author Information

This role was created by Daniel Segura.
