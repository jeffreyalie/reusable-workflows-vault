# Reusable Workflows — Vault (OpenBao)

Shared Gitea Actions reusable workflows for the `Infra` org. This is the **default** workflow repo — all secrets are fetched at runtime from **OpenBao** (Vault-compatible) via AppRole authentication. No infrastructure credentials are stored as Gitea secrets.

---

## Table of Contents

- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Secret Management — OpenBao](#secret-management--openbao)
- [Workflows](#workflows)
- [How to Call These Workflows](#how-to-call-these-workflows)
- [Input Reference](#input-reference)
- [Repository Structure Requirements](#repository-structure-requirements)
- [Troubleshooting](#troubleshooting)

---

## Overview

This repo provides a set of `workflow_call`-triggered Gitea Actions workflows that can be called from any repo in the `Infra` org. They handle the full VM lifecycle:

| Workflow | Trigger | What it does |
|---|---|---|
| `reusable-terraform-check.yml` | `workflow_call` | fmt, validate, tflint — no secrets needed |
| `reusable-terraform-plan.yml` | `workflow_call` | init + plan against MinIO backend |
| `reusable-terraform-apply.yml` | `workflow_call` | init + apply against MinIO backend |
| `reusable-terraform-destroy.yml` | `workflow_call` | init + destroy with confirmation gate |
| `reusable-terraform-init.yml` | `workflow_call` | init only — outputs working_dir and state_key |
| `reusable-ansible-check.yml` | `workflow_call` | lint + syntax-check, installs Galaxy roles |
| `reusable-ansible-deploy.yml` | `workflow_call` | init TF state → get VM IP → SSH wait → deploy |

**Key difference from `reusable-workflows`:** All credentials are read from OpenBao KV at runtime. The only Gitea org secrets required are the three AppRole credentials (`VAULT_ADDR`, `VAULT_ROLE_ID`, `VAULT_SECRET_ID`). Everything else — LXD certs, MinIO keys, SSH keys — lives inside OpenBao.

---

## Prerequisites

| Requirement | Notes |
|---|---|
| OpenBao running | Accessible from the Gitea runner (e.g. `http://10.x.x.x:8200`) |
| AppRole auth enabled | Role must have read access to the KV paths below |
| KV v2 secrets engine | Mounted and populated at `homelab/` path |
| Gitea org secrets | Only three: `VAULT_ADDR`, `VAULT_ROLE_ID`, `VAULT_SECRET_ID` |
| MinIO | S3-compatible state backend, bucket `terraform-state` must exist |
| Gitea Act Runner | Docker-based runner registered to the `Infra` org |

---

## Secret Management — OpenBao

All infrastructure credentials are stored in OpenBao KV v2. The workflows log in via AppRole, fetch secrets, export them to `$GITHUB_ENV`, and use them for all subsequent steps.

### Required Gitea Org Secrets

Set these three once at `Infra` org → Settings → Secrets:

| Secret | Description |
|---|---|
| `VAULT_ADDR` | OpenBao server URL (e.g. `http://10.248.42.x:8200`) |
| `VAULT_ROLE_ID` | AppRole Role ID |
| `VAULT_SECRET_ID` | AppRole Secret ID |

### KV v2 Secret Paths

Populate these paths in OpenBao before running any workflow:

**`homelab/data/lxd`**
| Key | Value |
|---|---|
| `address` | LXD API host IP |
| `client_cert` | Full PEM content of `gitea-runner.crt` |
| `client_key` | Full PEM content of `gitea-runner.key` |
| `trust_password` | LXD trust password *(optional, only if not using certs)* |

**`homelab/data/minio`**
| Key | Value |
|---|---|
| `endpoint` | MinIO S3 API URL (e.g. `http://10.248.42.22:9000`) |
| `access_key` | MinIO access key |
| `secret_key` | MinIO secret key |

**`homelab/data/ansible`**
| Key | Value |
|---|---|
| `ssh_public_key` | Full public key injected into VMs via cloud-init |
| `ssh_private_key` | Full private key used by the runner to SSH into VMs |

### How the AppRole Login Works

Each workflow runs this pattern before any infrastructure step:

```bash
# 1. Exchange AppRole creds for a short-lived token
VAULT_TOKEN=$(curl -sk "${VAULT_ADDR}/v1/auth/approle/login" \
  --data '{"role_id":"...","secret_id":"..."}' \
  | jq -r '.auth.client_token')

# 2. Read KV v2 paths and export to GITHUB_ENV
curl -sk -H "X-Vault-Token: ${VAULT_TOKEN}" \
  "${VAULT_ADDR}/v1/homelab/data/lxd" | jq -r '.data.data'
```

Multiline values (PEM certs, SSH keys) are written using the `<<__EOF__` heredoc syntax to avoid newline corruption in `$GITHUB_ENV`.

### Configuring Custom Vault Paths

The default KV paths (`homelab/data/lxd`, `homelab/data/minio`, `homelab/data/ansible`) can be overridden per-call via `vault_path_*` inputs:

```yaml
jobs:
  plan:
    uses: Infra/reusable-workflows-vault/.gitea/workflows/reusable-terraform-plan.yml@main
    with:
      environment:        "prod"
      vm_name:            "prod01"
      vault_path_lxd:     "production/data/lxd"      # override default
      vault_path_minio:   "production/data/minio"
      vault_path_ansible: "production/data/ansible"
    secrets: inherit
```

---

## Workflows

### `reusable-terraform-check.yml`

Validates Terraform code. Requires no secrets — safe to call on any branch.

Steps: `fmt -check` → `init -backend=false` → `validate` → `tflint`

```yaml
jobs:
  check:
    uses: Infra/reusable-workflows-vault/.gitea/workflows/reusable-terraform-check.yml@main
    with:
      terraform_dir: "terraform"   # optional, default: "terraform"
```

---

### `reusable-terraform-plan.yml`

Fetches secrets from OpenBao, writes LXD certs, inits Terraform against MinIO, validates, and runs plan.

```yaml
jobs:
  plan:
    uses: Infra/reusable-workflows-vault/.gitea/workflows/reusable-terraform-plan.yml@main
    with:
      environment:        ${{ github.event.inputs.environment }}
      vm_name:            ${{ github.event.inputs.vm_name }}
      vault_path_lxd:     "homelab/data/lxd"      # optional
      vault_path_minio:   "homelab/data/minio"     # optional
      vault_path_ansible: "homelab/data/ansible"   # optional
    secrets: inherit
```

---

### `reusable-terraform-apply.yml`

Same flow as plan, runs `terraform apply -auto-approve`.

```yaml
jobs:
  apply:
    uses: Infra/reusable-workflows-vault/.gitea/workflows/reusable-terraform-apply.yml@main
    with:
      environment:        ${{ github.event.inputs.environment }}
      vm_name:            ${{ github.event.inputs.vm_name }}
      vault_path_lxd:     "homelab/data/lxd"
      vault_path_minio:   "homelab/data/minio"
      vault_path_ansible: "homelab/data/ansible"
    secrets: inherit
```

---

### `reusable-terraform-destroy.yml`

Hard-gates on `confirm_destroy == "yes"` before proceeding. Fetches secrets, inits, destroys.

```yaml
jobs:
  destroy:
    uses: Infra/reusable-workflows-vault/.gitea/workflows/reusable-terraform-destroy.yml@main
    with:
      environment:        ${{ github.event.inputs.environment }}
      vm_name:            ${{ github.event.inputs.vm_name }}
      confirm_destroy:    ${{ github.event.inputs.confirm_destroy }}
      vault_path_lxd:     "homelab/data/lxd"
      vault_path_minio:   "homelab/data/minio"
      vault_path_ansible: "homelab/data/ansible"
    secrets: inherit
```

---

### `reusable-ansible-check.yml`

Installs Ansible, installs Galaxy roles from `ansible/requirements.yml` (if present), runs lint and syntax-check.

```yaml
jobs:
  check:
    uses: Infra/reusable-workflows-vault/.gitea/workflows/reusable-ansible-check.yml@main
    with:
      ansible_playbook_path: "ansible/playbook.yml"   # optional
```

---

### `reusable-ansible-deploy.yml`

Fetches secrets from OpenBao → installs Galaxy roles → inits Terraform → reads VM IP from state → waits for SSH → deploys playbook.

```yaml
jobs:
  deploy:
    uses: Infra/reusable-workflows-vault/.gitea/workflows/reusable-ansible-deploy.yml@main
    with:
      environment:              ${{ github.event.inputs.environment }}
      vm_name:                  ${{ github.event.inputs.vm_name }}
      ansible_playbook_path:    "ansible/playbook.yml"   # optional
      ssh_wait_timeout_seconds: 200                       # optional
      vault_path_lxd:           "homelab/data/lxd"
      vault_path_minio:         "homelab/data/minio"
      vault_path_ansible:       "homelab/data/ansible"
    secrets: inherit
```

---

## How to Call These Workflows

### Full Example — All Six Caller Workflows

Create these files in your repo under `.gitea/workflows/`:

**`terraform-check.yml`**
```yaml
name: "Terraform Check"
on:
  pull_request:
  workflow_dispatch:
jobs:
  check:
    uses: Infra/reusable-workflows-vault/.gitea/workflows/reusable-terraform-check.yml@main
    with:
      terraform_dir: "terraform"
```

**`terraform-plan.yml`**
```yaml
name: "Terraform Plan"
on:
  workflow_dispatch:
    inputs:
      environment:
        description: "Environment (dev/prod)"
        required: true
        type: string
      vm_name:
        description: "VM name (e.g. dev01)"
        required: true
        type: string
jobs:
  plan:
    uses: Infra/reusable-workflows-vault/.gitea/workflows/reusable-terraform-plan.yml@main
    with:
      environment:        ${{ github.event.inputs.environment }}
      vm_name:            ${{ github.event.inputs.vm_name }}
      vault_path_lxd:     "homelab/data/lxd"
      vault_path_minio:   "homelab/data/minio"
      vault_path_ansible: "homelab/data/ansible"
    secrets: inherit
```

**`terraform-apply.yml`** — same shape as plan, replace `reusable-terraform-plan` with `reusable-terraform-apply`.

**`terraform-destroy.yml`** — add `confirm_destroy` input and pass to `reusable-terraform-destroy`.

**`ansible-check.yml`**
```yaml
name: "Ansible Check"
on:
  pull_request:
  workflow_dispatch:
jobs:
  check:
    uses: Infra/reusable-workflows-vault/.gitea/workflows/reusable-ansible-check.yml@main
    with:
      ansible_playbook_path: "ansible/playbook.yml"
```

**`ansible-deploy.yml`**
```yaml
name: "Ansible Deploy"
on:
  workflow_dispatch:
    inputs:
      environment:
        required: true
        type: string
      vm_name:
        required: true
        type: string
jobs:
  deploy:
    uses: Infra/reusable-workflows-vault/.gitea/workflows/reusable-ansible-deploy.yml@main
    with:
      environment:        ${{ github.event.inputs.environment }}
      vm_name:            ${{ github.event.inputs.vm_name }}
      vault_path_lxd:     "homelab/data/lxd"
      vault_path_minio:   "homelab/data/minio"
      vault_path_ansible: "homelab/data/ansible"
    secrets: inherit
```

---

## Input Reference

### Terraform Workflows

| Input | Required | Default | Description |
|---|---|---|---|
| `environment` | Yes | — | `dev` or `prod` |
| `vm_name` | Yes | — | Matches `.tfvars` filename (e.g. `dev01`) |
| `terraform_version` | No | latest | Pin a specific Terraform version |
| `terraform_dir` | No | `terraform` | Root dir for fmt/validate (check workflow only) |
| `confirm_destroy` | Yes* | — | Must be `yes` (destroy workflow only) |
| `vault_path_lxd` | No | `homelab/data/lxd` | KV v2 path for LXD secrets |
| `vault_path_minio` | No | `homelab/data/minio` | KV v2 path for MinIO secrets |
| `vault_path_ansible` | No | `homelab/data/ansible` | KV v2 path for Ansible secrets |

### Ansible Workflows

| Input | Required | Default | Description |
|---|---|---|---|
| `environment` | Yes | — | `dev` or `prod` |
| `vm_name` | Yes | — | Target VM name |
| `ansible_playbook_path` | No | `ansible/playbook.yml` | Path to playbook |
| `ssh_wait_timeout_seconds` | No | `200` | Max seconds to wait for SSH |
| `vault_path_*` | No | `homelab/data/*` | Override KV paths (deploy workflow) |

---

## Repository Structure Requirements

Calling repos must follow this layout:

```
your-repo/
├── terraform/
│   ├── env/
│   │   ├── dev/
│   │   │   ├── backend.tf      # terraform { backend "s3" {} }
│   │   │   ├── providers.tf
│   │   │   ├── main.tf
│   │   │   ├── variables.tf
│   │   │   ├── outputs.tf      # must expose vm_ip output
│   │   │   └── dev01.tfvars
│   │   └── prod/
│   │       └── prod01.tfvars
│   └── modules/                # only needed if using local module
└── ansible/
    ├── playbook.yml
    ├── requirements.yml        # Galaxy roles (optional)
    └── roles/                  # local roles (optional)
```

State path convention (auto-derived from inputs):
```
s3://terraform-state/state/<environment>/<vm_name>/terraform.tfstate
```

---

## Troubleshooting

**OpenBao login fails (`Failed to authenticate with OpenBao`)**
- Verify `VAULT_ADDR`, `VAULT_ROLE_ID`, `VAULT_SECRET_ID` are set correctly in Gitea org secrets
- Check OpenBao is reachable from the runner: `curl -sk $VAULT_ADDR/v1/sys/health`
- Confirm the AppRole has read policy on `homelab/data/*`

**Multiline secret corruption (PEM cert / SSH key broken)**
- The workflow uses `<<__EOF__` heredoc syntax in `$GITHUB_ENV` — do not modify this pattern
- Verify the KV value was stored with full PEM including `-----BEGIN/END-----` lines

**`tfvars file not found`**
- The `vm_name` input must exactly match a `.tfvars` filename under `terraform/env/<env>/`
- Example: input `dev01` → expects `terraform/env/dev/dev01.tfvars`

**Terraform init fails (MinIO)**
- Check `homelab/data/minio` keys: `endpoint`, `access_key`, `secret_key`
- Ensure bucket `terraform-state` exists in MinIO

**LXD cert setup skipped**
- The `Setup LXD Certs` step is conditional on `env.LXD_CLIENT_CERT != ''`
- If OpenBao returns an empty `client_cert`, check the KV key name is exactly `client_cert`

---

## Infrastructure Created and Maintained By

**Ali Ahmed**  
Building infrastructure, automation, and DevOps workflows

**Contact**

[![GitHub](https://img.shields.io/badge/GitHub-%20ali%20ahmed-black?style=for-the-badge&logo=github)](https://github.com/jeffreyalie)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-%20ali%20ahmed-blue?style=for-the-badge&logo=linkedin)](https://www.linkedin.com/in/ali-ahmed-261755252/)

