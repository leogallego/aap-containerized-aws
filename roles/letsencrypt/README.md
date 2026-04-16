# letsencrypt

Provisions Let's Encrypt SSL certificates using the Red Hat CoP **provider pattern**.
Two providers are available:

| | acme (default) | certbot |
|---|---|---|
| External binary | None | certbot via pip |
| DNS-01 | `amazon.aws.route53` | certbot-dns-route53 + IAM |
| HTTP-01 | Temp Python HTTP server | `--standalone` (built-in) |
| Idempotency | Native `remaining_days` | `changed_when` workarounds |
| Renewal | `ansible-playbook` via cron | `certbot renew` via cron |
| Dependencies | `community.crypto` collection | certbot + pip |

## Requirements

- `community.crypto` collection (acme provider)
- `amazon.aws` collection (DNS-01 challenge)
- EC2 IAM instance profile with Route53 access (DNS-01 renewal)

## Role Variables

See `defaults/main.yml` for all variables. Key inputs:

```yaml
letsencrypt_provider: acme          # or 'certbot'
letsencrypt_email: you@example.com  # required
letsencrypt_domain: aap.example.com # required (defaults to installer_fqdn_hostname)
letsencrypt_route53_zone_id: ""     # set for DNS-01, empty for HTTP-01
letsencrypt_deploy_to_aap: false    # true for standalone/post-install use
```

## Example Playbooks

### Pre-install (during AAP deployment)

```yaml
- name: Provision certificates before AAP install
  ansible.builtin.include_role:
    name: letsencrypt
  when: enable_letsencrypt | default(false) | bool
```

### Standalone (on running AAP instance)

```yaml
- name: Provision Let's Encrypt SSL certificates
  hosts: all
  gather_facts: true
  roles:
    - role: letsencrypt
      letsencrypt_deploy_to_aap: true
```

## Idempotency

Both providers are idempotent:
- **acme**: `remaining_days` parameter skips renewal if cert has enough validity
- **certbot**: checks existing cert before requesting

Running the role twice with the same parameters produces no changes the second time.

## Renewal

- **certbot**: `certbot renew` via cron at 2:30 and 14:30 daily, with deploy hook
- **acme**: `ansible-playbook /opt/aap-certs/renew.yml` via cron at 2:30 daily

## Testing with staging

Set `letsencrypt_acme_directory` to the staging URL to avoid rate limits:

```yaml
letsencrypt_acme_directory: https://acme-staging-v02.api.letsencrypt.org/directory
```

## License

GPL-3.0-or-later
