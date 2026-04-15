# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Ansible automation framework for deploying Red Hat Ansible Automation Platform (AAP) 2.6 containerized on AWS. Provisions AWS infrastructure (VPC, EC2, networking) and installs AAP with optional components (EDA, Automation Hub, Lightspeed, MCP Server) and demo content.

## Common Commands

```bash
# Install required collections
ansible-galaxy collection install -r requirements.yml

# Deploy AAP (infrastructure + install)
source env-vars.sh
ansible-playbook playbooks/deploy-aap.yml

# Deploy AAP with Product Demos content
source env-vars.sh
ansible-playbook playbooks/deploy-aap-with-content.yml

# Run individual steps
ansible-playbook playbooks/aws/create_infrastructure.yml
ansible-playbook -i inventory/hosts.yml playbooks/aap/install.yml
ansible-playbook playbooks/aap/install-content.yml

# Teardown all AWS resources
ansible-playbook playbooks/aws/teardown_infrastructure.yml
```

All playbooks require `source env-vars.sh` first. Extra vars are passed automatically via `extra-vars.yml` which reads from environment variables with defaults.

## Architecture

### Deployment Flow

```
deploy-aap.yml (orchestrator)
  -> aws/create_infrastructure.yml  (localhost: VPC, subnet, IGW, SG, EC2)
     -> generates inventory/hosts.yml dynamically
  -> aap/install.yml                (aap_nodes: pre-install, run installer, post-install)
     -> tasks/pre-install.yml       (packages, bundle extraction, inventory template)
     -> containerized installer     (async, 40-min timeout)
     -> tasks/post-install.yml      (service checks, API verification, cleanup)

deploy-aap-with-content.yml adds:
  -> aap/install-content.yml        (clones product-demos, runs via ansible-navigator + EE)
```

### Configuration Chain

Environment variables (`env-vars.sh`) -> `extra-vars.yml` (Jinja2 lookups with defaults) -> playbooks. All configurable values flow through `extra-vars.yml`.

### Key Files

- `env-vars.sample` - Reference template for all environment variables (copy to `env-vars.sh`)
- `extra-vars.yml` - Variable definitions with env lookups and defaults
- `playbooks/aap/templates/inventory.j2` - Dynamic AAP installer inventory (Jinja2 conditionals for component toggles)
- `files/` - AAP setup bundle (tar.gz) and generated SSH keys (gitignored)
- `inventory/hosts.yml` - Generated at runtime by infrastructure playbook

### AAP Component Toggles

Controlled via env vars in `env-vars.sh`, consumed through `extra-vars.yml`:
- `AAP_INCLUDE_CONTROLLER` (always true)
- `AAP_INCLUDE_EDA_CONTROLLER`
- `AAP_INCLUDE_AUTOMATION_HUB`
- `AAP_INCLUDE_LIGHTSPEED` (requires LLM endpoint config)
- `AAP_INCLUDE_MCP_SERVER`

## Design Patterns

- **Dynamic inventory generation**: `create_infrastructure.yml` writes `inventory/hosts.yml` with EC2 public IP and SSH key path, consumed by subsequent plays.
- **Bundle auto-detection**: Pre-install tasks glob for exactly one `ansible-automation-platform-*.tar.gz` in `files/` and fail if count != 1.
- **Async installer**: AAP containerized installer runs async with 40-min timeout and 30-sec polling to avoid SSH timeouts.
- **EE workaround**: `install-content.yml` uses `ansible-navigator` with an execution environment to work around Ansible 2.14/2.15 collection compatibility issues.

## Ansible Conventions

- Uses `amazon.aws` collection with fully qualified module names.
- All playbooks use `extra-vars.yml` via `ansible-playbook -e @extra-vars.yml` (embedded in orchestrator playbooks via `vars_files`).
- SSH access to deployed instances: `ssh -i files/<instance-name>-key.pem ec2-user@<public-ip>`.
