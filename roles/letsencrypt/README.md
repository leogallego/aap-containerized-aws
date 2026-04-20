# letsencrypt

Provisions Let's Encrypt SSL certificates for AAP containerized deployments using the Red Hat CoP **provider pattern**. Handles certificate issuance, deployment to AAP services, and automated renewal.

## Provider Comparison

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

Install collections locally:

```bash
ansible-galaxy collection install -r requirements.yml -p collections
```

## Role Variables

See `defaults/main.yml` for all variables. Key inputs:

```yaml
letsencrypt_provider: acme          # or 'certbot'
letsencrypt_email: you@example.com  # required
letsencrypt_domain: aap.example.com # required (defaults to installer_fqdn_hostname)
letsencrypt_route53_zone_id: ""     # set for DNS-01, empty for HTTP-01
letsencrypt_deploy_to_aap: false    # true for standalone/post-install use
letsencrypt_include_mcp: false      # also deploy certs to MCP server
```

## Deployment Flows

### New AAP instance with SSL (recommended: Route53 DNS-01)

This is the fully automated path. Set these env vars before deploying:

```bash
export ENABLE_LETSENCRYPT="true"
export LETSENCRYPT_EMAIL="you@example.com"
export ROUTE53_HOSTED_ZONE_ID="Z0123456789"
export INSTALLER_FQDN_HOSTNAME="aap.yourdomain.com"
```

Then deploy as normal:

```bash
source env-vars.sh
ansible-playbook playbooks/deploy-aap.yml -e @extra-vars.yml
```

The flow handles everything automatically:

1. `create_infrastructure.yml` creates EC2, gets the public IP, creates the Route53 A record, and attaches an IAM instance profile with Route53 permissions
2. `pre-install.yml` runs the letsencrypt role with DNS-01 challenge (no port 80 needed)
3. AAP installer runs with the trusted certificate already in place

### New AAP instance with SSL (HTTP-01 — no Route53)

Without Route53, DNS must be configured manually between infrastructure creation and AAP installation:

```bash
# Step 1: Create infrastructure
source env-vars.sh
ansible-playbook playbooks/aws/create_infrastructure.yml -e @extra-vars.yml
# Note the public IP from the output

# Step 2: Create a DNS A record pointing your domain to the public IP
# (use your DNS provider's console or API)

# Step 3: Verify DNS is resolving
host aap.yourdomain.com  # should return the public IP

# Step 4: Install AAP (letsencrypt runs during pre-install)
ansible-playbook -i inventory/hosts.yml playbooks/aap/install.yml -e @extra-vars.yml
```

Port 80 must be reachable from the internet for HTTP-01 validation. The role temporarily stops the gateway proxy if it is using port 80.

### Standalone — add SSL to an existing AAP instance

For instances already running without SSL:

```bash
source env-vars.sh
ansible-playbook -i inventory/<instance>-hosts.yml \
  playbooks/provision-letsencrypt.yml -e @extra-vars.yml
```

This playbook sets `letsencrypt_deploy_to_aap: true`, so it issues the certificate, copies it to the AAP gateway (and MCP if enabled), and restarts the services.

## Example Playbook

```yaml
- name: Provision Let's Encrypt SSL certificates
  hosts: all
  gather_facts: true
  roles:
    - role: letsencrypt
      letsencrypt_deploy_to_aap: true
```

## AAP Upgrades and Installer Re-runs

When upgrading AAP or re-running the installer, certificates are preserved automatically if you use the project playbooks (`deploy-aap.yml` or `install.yml`) — the inventory template includes the TLS paths and the letsencrypt role skips re-issuance when certs are still valid.

If you re-run the AAP containerized installer manually (e.g., `./setup.sh`), include these lines in your inventory to keep the Let's Encrypt certificates:

```ini
gateway_tls_cert=/etc/letsencrypt/live/<your-domain>/fullchain.pem
gateway_tls_key=/etc/letsencrypt/live/<your-domain>/privkey.pem

# If using MCP server:
mcp_tls_cert=/etc/letsencrypt/live/<your-domain>/fullchain.pem
mcp_tls_key=/etc/letsencrypt/live/<your-domain>/privkey.pem
```

Without these settings, the installer regenerates self-signed certificates and the Let's Encrypt certs on disk go unused.

## Idempotency

Both providers are idempotent:
- **acme**: `remaining_days` parameter skips renewal if the cert has enough validity
- **certbot**: checks existing cert before requesting

Running the role twice with the same parameters produces no changes the second time.

Switching providers (e.g. from certbot to acme) is handled safely — the role detects and cleans up the previous provider's file structure before generating new keys.

## Check Mode

The role supports `--check` mode for validation without making changes. ACME/certbot certificate requests and `command:` tasks are skipped in check mode. Stat-based checks and assertions run normally, allowing you to verify configuration before applying.

## Rollback

The role does not provide automated rollback. To revert to the previous state:

1. **Restore self-signed certificates**: re-run the AAP installer without `ENABLE_LETSENCRYPT` — the installer regenerates its own self-signed certificates.
2. **Restore previous Let's Encrypt certificate**: template and copy tasks use `backup: true`, so the previous files are preserved on disk with a timestamped suffix. Manually restore and restart the gateway.
3. **Remove renewal automation**: delete the cron job (`crontab -e -u root`) and the renewal directory (`/opt/aap-certs` for acme, `/etc/letsencrypt` for certbot).

## Renewal

Both providers set up automatic renewal via cron:

- **acme**: `ansible-playbook /opt/aap-certs/renew.yml` at 2:30 AM daily
- **certbot**: `certbot renew --quiet` at 2:30 and 14:30 daily, with a deploy hook that copies certs and restarts services

Renewal includes automatic certificate deployment to the gateway (and MCP if enabled).

## Testing with staging

Set `letsencrypt_acme_directory` to the staging URL to avoid rate limits during development:

```yaml
letsencrypt_acme_directory: https://acme-staging-v02.api.letsencrypt.org/directory
```

Staging certificates are functionally identical but not trusted by browsers. Let's Encrypt production has strict rate limits (50 certificates per domain per week).

## License

GPL-3.0-or-later
