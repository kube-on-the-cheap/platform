# Terraform scaffolding for self-hosted Zitadel

Date: 2026-06-05
Status: Approved (scaffolding only — no Zitadel resources yet)

## Goal

Stand up a Terraform/OpenTofu harness in the platform repo to manage the
self-hosted Zitadel instance running on the freeloader cluster as code.
The scaffolding must be ready for follow-up specs to drop in OIDC clients
(starting with Grimmory on understairs) as ~30-line `.tf` files, without
re-litigating provider, backend, or secrets handling.

## Non-goals

- Defining any actual Zitadel resources (projects, applications, users, roles).
  Those land in follow-up specs.
- CI integration. First iteration runs from a developer laptop.
- `tofu-controller` / GitOps reconciliation of Terraform state.
- Importing existing Zitadel state (no clients exist yet — fresh start).
- Bootstrapping the Terraform service user via Terraform itself
  (chicken-and-egg).

## Context

- Platform repo (`~/Repositories/platform/`) is Flux + Kustomize GitOps.
  No Terraform exists today, though `README.md` already references it
  conceptually for cluster bootstrap (Cilium).
- Zitadel is deployed on the freeloader cluster via the sibling apps repo
  at `~/Repositories/apps/apps/zitadel/overlays/freeloader/`. It is reachable
  at `https://auth.cloud.blacksd.tech`.
- CNPG-backed Postgres for Zitadel uses Barman S3 backups to an Oracle Cloud
  Object Storage bucket named `pgzitadel`, endpoint
  `https://froaw0vigiem.compat.objectstorage.eu-frankfurt-1.oraclecloud.com`.
- SOPS is the established secret-handling pattern, with cluster-specific age
  recipients defined in `.sops.yaml`. Files matching
  `.*/overlays/freeloader/.*\.sops(?:\.yaml)?$` are encrypted with the
  freeloader age key, Marco's PGP key, and Marco's age backup key.
- `devbox.json` is the toolchain manifest at repo root.

## Decisions

### 1. OpenTofu over Terraform

Default to OpenTofu (`opentofu` Nixpkgs / devbox package). Aligns with the
FOSS-leaning posture of the rest of the toolchain (Helm, Flux, Kustomize,
SOPS, age) and avoids HashiCorp BSL concerns. State files and provider
behavior are wire-compatible with Terraform; can switch back if needed.

### 2. State backend: S3 on existing Oracle Cloud bucket

Reuse the `pgzitadel` bucket under a dedicated prefix:

```
s3://pgzitadel/terraform/zitadel/terraform.tfstate
```

Rationale:
- No new infrastructure to provision.
- Same credentials/endpoint as the existing Barman setup, so the operator
  workstation already has working S3 access.
- Prefix isolation keeps Terraform state from colliding with Barman backups
  under `s3://pgzitadel/backups/`.

Required Oracle Cloud S3 compatibility flags in the backend block:
- `endpoints = { s3 = "https://froaw0vigiem.compat.objectstorage.eu-frankfurt-1.oraclecloud.com" }`
- `region = "eu-frankfurt-1"` (required by the AWS SDK; ignored server-side)
- `skip_credentials_validation = true`
- `skip_metadata_api_check = true`
- `skip_region_validation = true`
- `skip_requesting_account_id = true`
- `use_path_style = true`

No DynamoDB locking (Oracle isn't AWS). Single-operator usage only.
Document the limitation.

### 3. Repository location: `terraform/zitadel/`

New top-level directory, peer to `apps/`, `clusters/`, `infrastructure/`.
The `terraform/` parent leaves room for future Terraform-managed
subsystems (e.g., DNS, other SaaS providers) without restructuring.

### 4. Secret storage: SOPS, freeloader-keyed

The Zitadel machine-user PAT lives in
`terraform/zitadel/secrets.sops.yaml`, encrypted via the existing freeloader
SOPS rules. To make the file match the existing path regex, extend
`.sops.yaml` with a new rule that covers `terraform/zitadel/.*\.sops\.yaml$`
(otherwise the file would not be picked up).

Operator reads the PAT with `sops -d` and exports it as `TF_VAR_zitadel_token`
before running `tofu plan` / `tofu apply`.

S3 backend credentials are sourced from the operator's environment
(`AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY`) — same pattern as `kubectl`
access today. Not committed to the repo.

### 5. Service user creation: documented manual step

Operator creates a Zitadel service user (machine account) and PAT in the
Zitadel Console as a one-time bootstrap, then encrypts the PAT into
`secrets.sops.yaml`. README walks through this step by step.

## Directory layout

```
terraform/
  zitadel/
    versions.tf              # tofu + provider version pins
    backend.tf               # S3 backend pointed at Oracle Cloud bucket
    provider.tf              # zitadel/zitadel provider config
    variables.tf             # zitadel_domain, zitadel_token, zitadel_org_id
    main.tf                  # currently empty (locals block only)
    terraform.tfvars.example # template — operator copies to terraform.tfvars
    secrets.sops.yaml        # encrypted Zitadel PAT
    .gitignore               # ignore tfvars, plan files, .terraform/
    clients/
      .gitkeep               # placeholder for future OIDC client modules
    README.md                # operator runbook
```

The `.terraform.lock.hcl` file is committed once `tofu init` runs (standard
convention).

## File contents

### `versions.tf`

```hcl
terraform {
  required_version = ">= 1.8.0"

  required_providers {
    zitadel = {
      source  = "zitadel/zitadel"
      version = "~> 1.2"
    }
  }
}
```

### `backend.tf`

```hcl
terraform {
  backend "s3" {
    bucket = "pgzitadel"
    key    = "terraform/zitadel/terraform.tfstate"
    region = "eu-frankfurt-1"

    endpoints = {
      s3 = "https://froaw0vigiem.compat.objectstorage.eu-frankfurt-1.oraclecloud.com"
    }

    use_path_style              = true
    skip_credentials_validation = true
    skip_metadata_api_check     = true
    skip_region_validation      = true
    skip_requesting_account_id  = true
  }
}
```

### `provider.tf`

```hcl
provider "zitadel" {
  domain = var.zitadel_domain
  token  = var.zitadel_token
}
```

### `variables.tf`

```hcl
variable "zitadel_domain" {
  description = "Hostname of the self-hosted Zitadel instance."
  type        = string
  default     = "auth.cloud.blacksd.tech"
}

variable "zitadel_token" {
  description = "Personal access token for the Terraform service user. Provide via TF_VAR_zitadel_token after decrypting secrets.sops.yaml."
  type        = string
  sensitive   = true
}

variable "zitadel_org_id" {
  description = "ID of the Zitadel organization that owns the resources managed by this configuration."
  type        = string
}
```

### `terraform.tfvars.example`

```hcl
# Copy to terraform.tfvars and fill in real values. terraform.tfvars is gitignored.
# zitadel_token is preferably set via TF_VAR_zitadel_token from sops, not stored here.

zitadel_domain = "auth.cloud.blacksd.tech"
zitadel_org_id = "REPLACE_ME"
```

### `secrets.sops.yaml` (plaintext skeleton before encryption)

```yaml
# Encrypted Zitadel Terraform credentials. Decrypt with sops -d.
# The zitadel_token field is the Personal Access Token of the Terraform
# machine user created manually in the Zitadel Console.
zitadel_token: REPLACE_ME_PAT_FROM_ZITADEL_CONSOLE
```

After populating, encrypt with `sops --encrypt --in-place secrets.sops.yaml`.
The `.sops.yaml` rule (below) ensures the right recipients are used.

### `.sops.yaml` addition (at repo root)

Add a new rule alongside the existing freeloader rule:

```yaml
  - path_regex: ^terraform/zitadel/.*\.sops\.yaml$
    mac_only_encrypted: true
    key_groups:
      - pgp:
          - *Marco_pgp
        age:
          - *freeloader
          - *Marco_age_backup
```

Note no `encrypted_regex` — the Terraform secrets file is a flat YAML, not
a Kubernetes Secret with `data`/`stringData`. SOPS will encrypt all leaf
values.

### `main.tf`

```hcl
# Placeholder. Zitadel resources land in clients/*.tf in follow-up specs.

locals {
  managed_by = "terraform/zitadel (platform repo)"
}
```

### `clients/.gitkeep`

Empty file. Future OIDC client modules go here (e.g.,
`clients/grimmory.tf`).

### `.gitignore` (in `terraform/zitadel/`)

```
*.tfvars
!terraform.tfvars.example
*.tfplan
.terraform/
crash.log
crash.*.log
```

`.terraform.lock.hcl` is intentionally **not** ignored — commit it.

## `devbox.json` change

Add `opentofu@latest` to the `packages` list. Renovate's devbox manager
will keep it current.

## README outline

`terraform/zitadel/README.md` covers:

1. **What this manages** — OIDC clients and related Zitadel resources on
   the freeloader-hosted Zitadel at `auth.cloud.blacksd.tech`.
2. **One-time bootstrap**:
   - Create a Zitadel service user (machine account, `IAM_OWNER` role at
     the org level) via the Console.
   - Generate a Personal Access Token for that user.
   - Look up the organization ID in the Console.
   - Write the PAT into `secrets.sops.yaml` and encrypt:
     `sops --encrypt --in-place secrets.sops.yaml`.
   - Copy `terraform.tfvars.example` → `terraform.tfvars`, fill in
     `zitadel_org_id`.
3. **S3 state backend credentials** — export
   `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` for the `pgzitadel`
   bucket (same credentials used for Barman; available in 1Password or
   wherever the operator keeps them).
4. **Day-to-day workflow**:
   ```
   devbox shell
   export TF_VAR_zitadel_token=$(sops -d --extract '["zitadel_token"]' secrets.sops.yaml)
   tofu init    # first run only
   tofu plan
   tofu apply
   ```
5. **Limitations** — single-operator only, no state locking, no CI.
6. **Adding a new OIDC client** — pointer to where future per-client `.tf`
   files live (`clients/`) and a note that each gets its own spec.

## Verification

After scaffolding:

- `devbox shell` succeeds and `tofu version` reports a recent OpenTofu.
- `tofu init` in `terraform/zitadel/` succeeds (requires bootstrap PAT and
  S3 credentials present).
- `tofu plan` returns "No changes" — confirms provider auth + state backend
  work without yet defining any resources.
- `sops -d secrets.sops.yaml` round-trips the PAT.

These checks are operator-side. The PR for this scaffolding does not run
them in CI.

## Follow-up specs (out of scope here)

- `2026-MM-DD-zitadel-grimmory-oidc-client-design.md` — adds
  `clients/grimmory.tf` with an OIDC application for the Grimmory rollout
  on the understairs cluster.
