# Ansible Role: Let's Encrypt SSL Certificate with Cloudflare DNS Verification

This Ansible role automates the process of generating SSL certificates using [Let's Encrypt](https://letsencrypt.org/) with DNS verification, managed via [Cloudflare](https://www.cloudflare.com/) DNS.

## Features

- Generates a private key and Certificate Signing Request (CSR).
- Uses the DNS challenge to verify domain ownership with Cloudflare DNS.
- Obtains SSL certificates from Let's Encrypt.
- Supports automatic certificate renewal.
- Encrypts the generated keys and certificates using Ansible Vault.
- Cleans up the DNS challenge record after verification.

## Requirements

- Ansible 2.17.4 or higher.
- Cloudflare account with API token access.

## Role Variables

Here are the role variables and their default values. You will need to override them in your playbook or inventory to suit your environment:

| Variable | Description | Default |
| - | - | - |
| ssl_cloudflare_domain | The domain name for which the SSL certificate will be generated. | - |
| ssl_cloudflare_token | Cloudflare API token with permissions to manage DNS records. | - |
| ssl_letsencrypt_email | Email address used to notify you when the certificate is approaching its expiration date. | `admin@{{ ssl_cloudflare_domain }}` |
| ssl_san_certificates | Subject Alternative Name(s) for the certificate. | `'["DNS:www.{{ ssl_cloudflare_domain }}"]'` |
| ssl_cert_directory | Directory where SSL certificates will be stored. | `/etc/ssl/certs` |
| ssl_key_directory | Directory where SSL private keys will be stored. | `/etc/ssl/private` |

## Example Playbook

```yaml
- hosts: webserver
  become: true
  roles:
    - role: dsegurag.ssl
      vars:
        ssl_cloudflare_domain: "example.com"
        ssl_cloudflare_token: "{{ lookup('ansible.builtin.env', 'CLOUDFLARE_API_TOKEN') }}"
        ssl_letsencrypt_email: "admin@{{ ssl_cloudflare_domain }}"
        ssl_san_certificates: '["DNS:*.{{ ssl_cloudflare_domain }}"]'
```

## Tasks Overview

1. Create Directories: Generates directories for decrypted and encrypted SSL files with appropriate permissions.
2. Generate Private Key and CSR: Creates a 4096-bit RSA private key and generates a CSR for the specified domain.
3. Create Let’s Encrypt Account Key: Generates a separate private key for Let's Encrypt interactions.
4. Create DNS Challenge Record: Creates a DNS TXT record for the DNS challenge via Cloudflare.
5. Validate and Retrieve Certificate: Waits for the challenge to be validated and retrieves the certificate, intermediate chain, and full chain.
6. Clean Up: Deletes the DNS TXT record for the challenge once verification is complete.

## Dependencies

- None.

## License

MIT License

## Author Information

This role was created by Daniel Segura.
