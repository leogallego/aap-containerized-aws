# Ansible Automation Platform - AWS Deployment

Automated deployment of Ansible Automation Platform (AAP) on AWS infrastructure using Ansible playbooks.

## Quick Start

```bash
# 1. Configure credentials
cp env-vars.sample env-vars.sh
vim env-vars.sh                    # Update Red Hat registry credentials and AWS profile
aws login                          # Authenticate with AWS (opens browser)
source env-vars.sh

# 2. Install dependencies
ansible-galaxy collection install -r requirements.yml

# 3. Add AAP setup bundle
# Place your AAP bundle in files/ directory:
# files/ansible-automation-platform-containerized-setup-bundle-*.tar.gz

# 4. Deploy everything (choose one)
# Basic deployment (AAP only)
ansible-playbook playbooks/deploy-aap.yml

# Full deployment (AAP + demo content)
ansible-playbook playbooks/deploy-aap-with-content.yml
```

## Prerequisites

- Python 3.8+ with Ansible Core 2.14+
- AWS CLI v2 configured with credentials
- AWS permissions for EC2, VPC, and IAM
- Red Hat subscription with registry access
- AAP containerized setup bundle

## Project Structure

```
aws-aap-container/
├── env-vars.sh                      # Environment configuration
├── playbooks/
│   ├── deploy-aap.yml              # Complete deployment
│   ├── aws/
│   │   ├── create_infrastructure.yml
│   │   └── teardown_infrastructure.yml
│   └── aap/
│       ├── install.yml
│       ├── pre-install.yml
│       └── post-install.yml
└── files/                          # Place AAP bundle here
```

## Deployment Options

### Full Deployment with Demo Content
```bash
ansible-playbook playbooks/deploy-aap-with-content.yml
```

### Basic AAP Deployment Only
```bash
ansible-playbook playbooks/deploy-aap.yml
```

### Step by Step
```bash
# Create AWS infrastructure
ansible-playbook playbooks/aws/create_infrastructure.yml

# Install AAP
ansible-playbook playbooks/aap/install.yml

# Install demo content (optional)
ansible-playbook playbooks/aap/install-content.yml
```

## Configuration

### AWS Credentials Setup

This project uses an AWS CLI profile with `credential_process` to automatically refresh temporary credentials. This prevents token expiration during long-running playbooks (the AAP installer can take 30-40 minutes).

**One-time setup:**

1. Add a profile to `~/.aws/config`:

   ```ini
   [profile aap-deploy]
   credential_process = aws configure export-credentials --profile default --format process
   region = us-east-1
   ```

2. Log in and verify:

   ```bash
   aws login
   aws sts get-caller-identity --profile aap-deploy
   ```

**Daily workflow:**

```bash
aws login              # authenticate once per session (opens browser)
source env-vars.sh     # sets AWS_PROFILE=aap-deploy
ansible-playbook playbooks/deploy-aap.yml
```

> **Note:** If you have `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, or `AWS_SESSION_TOKEN` set in your environment, they will override the profile. The `env-vars.sh` script automatically unsets these variables to prevent conflicts.

### Application Settings

Edit `env-vars.sh` for basic settings:

```bash
# Required - Red Hat registry credentials
export INSTALLER_REGISTRY_USERNAME="your-username"
export INSTALLER_REGISTRY_PASSWORD="your-password"

# Optional - AWS settings
export AWS_REGION="us-east-1"
export AWS_INSTANCE_TYPE="t3.xlarge"
export INSTANCE_NAME="aap-containerized"

# Optional - AAP settings
export INSTALLER_ADMIN_PW="ansible123"
export AAP_INCLUDE_CONTROLLER="true"
export AAP_INCLUDE_EDA_CONTROLLER="false"
export AAP_INCLUDE_AUTOMATION_HUB="false"
export AAP_INCLUDE_LIGHTSPEED="false"
export AAP_INCLUDE_MCP_SERVER="false"

# Optional - Demo content
export INSTALL_PRODUCT_DEMOS="true"
```

### Ansible Lightspeed Configuration

```bash
export AAP_INCLUDE_LIGHTSPEED="true"
export LIGHTSPEED_CHATBOT_MODEL_URL="http://your-llm-endpoint:8000/v1"
export LIGHTSPEED_CHATBOT_MODEL_API_KEY="your-api-key"
export LIGHTSPEED_CHATBOT_MODEL_ID="your-model-id"
```

### Ansible MCP Server Configuration

```bash
export AAP_INCLUDE_MCP_SERVER="true"
export MCP_ALLOW_WRITE_OPERATIONS="true"  # Optional, default is read-only
```

### SSL/TLS with Let's Encrypt

Provision trusted Let's Encrypt certificates automatically during deployment. Two challenge methods are supported:

**DNS-01 via Route53 (recommended — fully automated):**

```bash
export ENABLE_LETSENCRYPT="true"
export LETSENCRYPT_EMAIL="you@example.com"
export ROUTE53_HOSTED_ZONE_ID="Z0123456789"  # enables DNS-01
export INSTALLER_FQDN_HOSTNAME="aap.yourdomain.com"
```

With Route53, the deployment is fully hands-off: infrastructure creation sets up the DNS A record and IAM permissions, then the letsencrypt role obtains the certificate via DNS challenge before AAP installation.

**HTTP-01 (no Route53 — manual DNS step required):**

```bash
export ENABLE_LETSENCRYPT="true"
export LETSENCRYPT_EMAIL="you@example.com"
export INSTALLER_FQDN_HOSTNAME="aap.yourdomain.com"
# ROUTE53_HOSTED_ZONE_ID is not set
```

Without Route53, create the DNS A record manually after infrastructure provisioning (step-by-step deployment required). Port 80 must be reachable.

**Add SSL to an existing instance:**

```bash
ansible-playbook -i inventory/<instance>-hosts.yml \
  playbooks/provision-letsencrypt.yml -e @extra-vars.yml
```

See the [letsencrypt role README](roles/letsencrypt/README.md) for full details on providers, variables, and renewal.

### Optional Configuration

```bash
# Demo content installation
export INSTALL_PRODUCT_DEMOS="true"

# Advanced options
export SKIP_SYSTEM_UPDATE="true"    # Skip system updates
export BUNDLE_INSTALL="true"        # Use bundle install
```

## Access

After deployment:
- **Web Interface:** `https://<public-ip>` (or `https://<fqdn>` if SSL is enabled)
- **Username:** `admin`
- **Password:** Value from `INSTALLER_ADMIN_PW`

Public IP address is displayed in the completion message.

## Cleanup

```bash
ansible-playbook playbooks/aws/teardown_infrastructure.yml
```

## Troubleshooting

**Bundle not found:**
```bash
ls files/ansible-automation-platform-containerized-setup-bundle-*.tar.gz
```

**Registry authentication:**
```bash
podman login registry.redhat.io
```

**AWS permissions:**
```bash
aws sts get-caller-identity
```

**Installation logs:**
```bash
ssh -i files/<instance-name>-key.pem ec2-user@<public-ip>
sudo podman ps -a
``` 