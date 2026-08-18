# Changelog

## 2.0.0 (2026-08-18)

The role no longer implements the ACME protocol with Ansible modules. It now installs and
configures certbot on the target host, which means **the certificate renews itself**
instead of only being renewed when a playbook happens to run.

### Breaking changes:

- Certificates are issued by `certbot`, installed from the distribution repositories
  along with `python3-certbot-dns-cloudflare`. The host must be Debian or Ubuntu with
  systemd.
- The Cloudflare API token is now stored on the host at `/etc/letsencrypt/cloudflare.ini`
  (`0600 root:root`). This is unavoidable for unattended DNS-01 renewal; scope the token
  to a single zone, or delegate `_acme-challenge` by CNAME to a throwaway zone.
- Output moved to `/etc/letsencrypt/live/<ssl_cert_name>/`, keeping the names
  `privkey.pem`, `cert.pem`, `chain.pem` and `fullchain.pem`. Certificates issued by 1.x
  are not adopted; a fresh one is issued on first run.
- Default key type is now `ecdsa`, following certbot's own default. Set
  `ssl_key_type: rsa` with `ssl_rsa_key_size` for the previous behaviour.
- Variables reworked:

  | 1.x | 2.0 |
  | - | - |
  | `ssl_cloudflare_domain` (string) | `ssl_domains` (list of names, no `DNS:` prefix), plus `ssl_cert_name` for the lineage name |
  | `ssl_san_certificates` | folded into `ssl_domains` |
  | `ssl_letsencrypt_email` | `ssl_email` |
  | `ssl_cert_directory`, `ssl_key_directory` | `ssl_dir`, plus `ssl_privatekey`, `ssl_cert`, `ssl_fullchain`, `ssl_chain` |
  | `ssl_cloudflare_token` | unchanged |

- `min_ansible_version` raised from 2.17 to 2.18.

### Features:

- Unattended renewal on the instance via `certbot.timer`, enabled by default and
  controlled with `ssl_auto_renew`. No scheduled Ansible run needed.
- `ssl_reload_services` and `ssl_deploy_hook_command` install a deploy hook that runs
  after every successful renewal.
- `ssl_renew_days` moves the renewal window; empty keeps certbot's default.
- `ssl_acme_directory` makes the endpoint configurable, so the role can be tested against
  Let's Encrypt staging instead of burning production rate limits.
- Changing `ssl_domains` reissues the certificate instead of being silently ignored.

### Fixes:

The following bugs were all in the hand-rolled DNS-01 handling, which no longer exists:

- Challenge TXT records were published with `solo: true`. The apex and the wildcard share
  the `_acme-challenge.<zone>` record name, so publishing the second value deleted the
  first one and validation failed.
- The challenge was read from `challenge_data`, which yields one entry per identifier and
  therefore duplicate record names, instead of `challenge_data_dns`.
- Validation was requested with no wait for DNS propagation. The `until`/`retries` in
  place only retried when the module itself errored, which does not recover an
  authorization already burned by a failed validation.
- The ACME account was handled implicitly, with two different contact addresses for the
  same account key. That behaviour is deprecated and removed in `community.crypto` 4.0.0.

### Other changes:

- The `community.crypto` and `community.general` collections are no longer required; the
  role is pure `ansible.builtin`.
- `requirements-ansible.txt` installs `ansible-core` instead of the full `ansible`
  package.
- README no longer claims the role encrypts anything with Ansible Vault; it never did.

## 1.0.0 (2024-09-20)

### Features:

- Initial release.
