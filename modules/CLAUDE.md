# CLAUDE.md — Epic 1: Terraform Module Registry (Multi-Cloud)

## ⚠️ Read Before Writing Any Module Code

```
standards/NAMING.md    — resource name patterns per cloud
standards/TAGGING.md   — mandatory tags, provider-specific implementation
```

## Epic Goal

Build a versioned, security-reviewed Terraform module registry for **Azure, AWS, and GCP**.
Every logical module (vm, vnet, storage, database, keyvault) has three provider
implementations sharing an identical external interface.

**Principle:** Same variable names and output names across all three clouds. Only
`main.tf` and `versions.tf` differ between providers.

---

## Stories

| ID | Story | Priority |
|----|-------|----------|
| 1.1 | Base Modules — all 3 providers | P0 |
| 1.2 | Composition Modules — all 3 providers | P1 |
| 1.3 | Landing Zone Modules — all 3 providers | P2 |

---

## Module Directory Layout

```
modules/base/<module-name>/
├── CLAUDE.md             ← story-level instructions
├── azure/
│   ├── main.tf
│   ├── variables.tf      ← identical interface to aws/ and gcp/
│   ├── outputs.tf        ← identical interface to aws/ and gcp/
│   ├── versions.tf
│   ├── README.md
│   ├── security.md
│   ├── cost.md
│   └── tests/
│       └── main_test.go
├── aws/
│   └── (same files)
└── gcp/
    └── (same files)
```

---

## Story 1.1 — Base Modules

### Modules × Providers Matrix

| Module | Azure Resource | AWS Resource | GCP Resource |
|--------|---------------|--------------|--------------|
| `vm` | `azurerm_linux_virtual_machine` | `aws_instance` | `google_compute_instance` |
| `vnet` | `azurerm_virtual_network` | `aws_vpc` + `aws_subnet` | `google_compute_network` + `google_compute_subnetwork` |
| `storage` | `azurerm_storage_account` | `aws_s3_bucket` | `google_storage_bucket` |
| `database` | `azurerm_mssql_server` + `azurerm_mssql_database` | `aws_db_instance` (RDS PostgreSQL) | `google_sql_database_instance` |
| `keyvault` | `azurerm_key_vault` | `aws_kms_key` + `aws_secretsmanager_secret` | `google_kms_key_ring` + `google_secret_manager_secret` |

### Shared Interface (copy into every provider's variables.tf)

```hcl
# Shared interface — identical across azure/, aws/, gcp/
variable "name"        { type = string; description = "Short descriptor (kebab-case). Combined with subsidiary/env/provider to form full name per NAMING.md." }
variable "environment" { type = string; description = "prod | staging | dev | sandbox" }
variable "subsidiary"  { type = string; description = "Subsidiary slug (max 8 chars)" }
variable "location"    { type = string; description = "Cloud region (azure: eastus, aws: us-east-1, gcp: us-central1)" }
variable "tags"        { type = map(string); description = "See standards/TAGGING.md. Copy validation block verbatim." }
```

```hcl
# Shared outputs — identical names across azure/, aws/, gcp/
output "id"   { description = "Resource ID (Azure id / AWS ARN / GCP self_link)" }
output "name" { description = "Resolved resource name per NAMING.md" }
output "governance_metadata" {
  description = "Emitted to compliance platform"
  value = {
    module_path    = "<level>/<name>/<provider>"
    module_version = local.module_version
    subsidiary     = var.subsidiary
    environment    = var.environment
    provider       = "<azure|aws|gcp>"
  }
}
```

### Security Defaults Per Module × Provider

| Module | Azure Default | AWS Default | GCP Default |
|--------|--------------|-------------|-------------|
| `vm` | `disable_password_authentication=true`; OS disk encrypted | `ebs_optimized=true`; root volume encrypted | `enable_integrity_monitoring=true`; disk encrypted |
| `vnet` | NSG on every subnet; no public subnets by default | Flow logs enabled; no IGW by default | Private Google Access enabled |
| `storage` | `https_only=true`; `min_tls=TLS1_2`; `public_blob=false` | `block_public_acls=true`; versioning enabled; SSE-S3 | `uniform_bucket_level_access=true`; CMEK optional |
| `database` | `public_network_access=false`; private endpoint var | `publicly_accessible=false`; encryption at rest | `ipv4_enabled=false`; private IP only |
| `keyvault` | `purge_protection=true`; network_acls default deny | Key rotation enabled; deletion window=30 | `prevent_destroy=true`; CMEK |

### Name Construction (read NAMING.md first)

```hcl
# Place in each provider's main.tf
locals {
  module_version = "1.0.0"
  provider_code  = "az"   # or "aws" or "gcp"
  resource_name  = "${var.subsidiary}-${var.environment}-${local.provider_code}-<type>-${var.name}"
}
```

### Tags / Labels (read TAGGING.md first)

**Azure & AWS** — use `var.tags` directly on `tags = var.tags`.
**GCP** — map `var.tags` to `labels`. Copy the `locals { gcp_labels }` block from `standards/TAGGING.md §3`.
**AWS** — add `Name = local.resource_name` to every resource's tags block (console visibility).

### Acceptance Criteria (per module × provider = 15 total)

- [ ] `terraform validate` passes
- [ ] `terraform fmt -check` passes
- [ ] `tfsec` — zero HIGH/CRITICAL
- [ ] `checkov` — zero FAILED (or suppressed with justification)
- [ ] Terratest happy-path test passes
- [ ] Terratest validates mandatory tags are present on created resource
- [ ] Name conforms to NAMING.md pattern (assert in test)
- [ ] Security defaults enforced (cannot be disabled without explicit override variable)

---

## Story 1.2 — Composition Modules

Composition modules call base modules. They must NOT directly reference provider
resources — only `../base/<module>/<provider>`.

### Modules × Providers

| Module | Azure | AWS | GCP |
|--------|-------|-----|-----|
| `3-tier-app` | vnet + vm(web) + vm(app) + database | vpc + ec2(web) + ec2(app) + rds | vpc + gce(web) + gce(app) + cloud-sql |
| `secure-platform` | vnet + vm + keyvault + nsg | vpc + ec2 + kms + sg | vpc + gce + kms + fw |
| `container-stack` | AKS + acr + keyvault | EKS + ecr + kms | GKE + gcr + kms |

### Additional Files

```
modules/composition/<name>/<provider>/
└── examples/
    └── basic/
        └── main.tf     # Working copy-paste example
```

---

## Story 1.3 — Landing Zone Modules

Landing zones call composition modules and add enterprise-grade opinionated defaults.

### Additional Requirement

Every landing zone module must output a `compliance_profile` listing satisfied controls:

```hcl
output "compliance_profile" {
  value = {
    controls_satisfied = ["encryption-at-rest", "private-endpoints", "mandatory-tags", "tls-1-2"]
    provider           = "<azure|aws|gcp>"
    landing_zone_type  = "regulated-workload"
  }
}
```

---

## Provider Version Pins (use in all versions.tf)

```hcl
# azure/versions.tf
terraform {
  required_version = ">= 1.6.0"
  required_providers {
    azurerm = { source = "hashicorp/azurerm"; version = "~> 3.100" }
  }
}

# aws/versions.tf
terraform {
  required_version = ">= 1.6.0"
  required_providers {
    aws = { source = "hashicorp/aws"; version = "~> 5.50" }
  }
}

# gcp/versions.tf
terraform {
  required_version = ">= 1.6.0"
  required_providers {
    google = { source = "hashicorp/google"; version = "~> 5.30" }
  }
}
```

---

## CI Pipeline

File: `.github/workflows/modules-ci.yml`

Matrix strategy — run all jobs across `[azure, aws, gcp]` × `[base, composition, landing-zone]`:

```yaml
strategy:
  matrix:
    provider: [azure, aws, gcp]
    level: [base, composition, landing-zone]
```

Jobs: `fmt` → `validate` → `security` → `test` → `docs-diff`
