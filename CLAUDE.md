# CLAUDE.md — CCoE Federated IaC Governance Monorepo

## Purpose

This monorepo implements the **Federated Terraform IaC Governance** platform for a CCoE
(Cloud Centre of Excellence). The operating principle is: **Govern centrally, build locally.**

Subsidiaries deploy independently across **Azure, AWS, and GCP**. CCoE provides guardrails, not gates.

---

## ⚠️ Read Standards Before Writing Any Code

Before writing Terraform, policies, catalog entries, or pipeline templates, Claude Code
**must** read the relevant standards files:

| Task | Required reading |
|------|-----------------|
| Any Terraform resource block | `standards/NAMING.md` + `standards/TAGGING.md` |
| `variable "tags"` or tag validation | `standards/TAGGING.md` |
| `local { resource_name }` | `standards/NAMING.md` |
| Compliance policy (Rego) | `standards/NAMING.md` + `standards/TAGGING.md` |
| Catalog `metadata.yaml` | `standards/NAMING.md` |
| Pipeline `default_tags` / label config | `standards/TAGGING.md` |

See `standards/CLAUDE.md` for the full decision matrix.

---

## Monorepo Structure

```
ccoe-iac-governance/
├── standards/                  # ★ GOVERNANCE STANDARDS — read before writing code
│   ├── CLAUDE.md               #   When and how to use these files
│   ├── NAMING.md               #   Universal naming convention (all 3 clouds)
│   └── TAGGING.md              #   Mandatory & recommended tags (all 3 clouds)
│
├── modules/                    # Epic 1 — Terraform Module Registry
│   ├── base/                   #   Story 1.1 — Atomic resource modules
│   │   ├── vm/
│   │   │   ├── azure/          #     Azure: azurerm_linux_virtual_machine
│   │   │   ├── aws/            #     AWS:   aws_instance
│   │   │   └── gcp/            #     GCP:   google_compute_instance
│   │   ├── vnet/
│   │   │   ├── azure/          #     Azure: azurerm_virtual_network
│   │   │   ├── aws/            #     AWS:   aws_vpc
│   │   │   └── gcp/            #     GCP:   google_compute_network
│   │   ├── storage/
│   │   │   ├── azure/          #     Azure: azurerm_storage_account
│   │   │   ├── aws/            #     AWS:   aws_s3_bucket
│   │   │   └── gcp/            #     GCP:   google_storage_bucket
│   │   ├── database/
│   │   │   ├── azure/          #     Azure: azurerm_mssql_server
│   │   │   ├── aws/            #     AWS:   aws_db_instance (RDS)
│   │   │   └── gcp/            #     GCP:   google_sql_database_instance
│   │   └── keyvault/
│   │       ├── azure/          #     Azure: azurerm_key_vault
│   │       ├── aws/            #     AWS:   aws_kms_key + aws_secretsmanager_secret
│   │       └── gcp/            #     GCP:   google_kms_key_ring + google_secret_manager_secret
│   ├── composition/            #   Story 1.2 — Architecture pattern modules
│   │   ├── 3-tier-app/
│   │   │   ├── azure/
│   │   │   ├── aws/
│   │   │   └── gcp/
│   │   ├── secure-platform/
│   │   │   ├── azure/
│   │   │   ├── aws/
│   │   │   └── gcp/
│   │   └── container-stack/
│   │       ├── azure/          #     AKS
│   │       ├── aws/            #     EKS
│   │       └── gcp/            #     GKE
│   └── landing-zone/           #   Story 1.3 — Enterprise blueprint modules
│       ├── regulated-workload/
│       │   ├── azure/
│       │   ├── aws/
│       │   └── gcp/
│       ├── internet-facing-app/
│       │   ├── azure/
│       │   ├── aws/
│       │   └── gcp/
│       └── data-platform/
│           ├── azure/
│           ├── aws/
│           └── gcp/
│
├── policies/                   # Epic 2 — Policy Library
│   ├── security/               #   Story 2.1 — Security policies (multi-cloud)
│   ├── compliance/             #   Story 2.2 — Naming & tagging compliance
│   └── cost/                   #   Story 2.3 — Cost governance
│
├── pipeline-templates/         # Epic 3 — CI/CD Pipeline Template Library
│   ├── github-actions/
│   ├── gitlab-ci/
│   └── jenkins/
│
├── compliance-platform/        # Epic 4 — Compliance Platform
│   ├── schema/
│   ├── collector/
│   ├── api/
│   └── dashboard/
│
├── catalog/                    # Epic 5 — Git-Based Service Catalog
│   ├── networking/
│   ├── compute/
│   ├── database/
│   └── observability/
│
├── brownfield/                 # Epic 6 — Brownfield Modernization
│   ├── registry/
│   ├── scanner/
│   └── exceptions/
│
├── .github/workflows/
├── docs/
└── scripts/
```

---

## Epics & Stories Index

| Epic | Area | CLAUDE.md |
|------|------|-----------|
| Standards | Naming & Tagging | `standards/CLAUDE.md` |
| Epic 1 | Terraform Module Registry (3-cloud) | `modules/CLAUDE.md` |
| Epic 2 | Policy Library | `policies/CLAUDE.md` |
| Epic 3 | Pipeline Template Library | `pipeline-templates/CLAUDE.md` |
| Epic 4 | Compliance Platform | `compliance-platform/CLAUDE.md` |
| Epic 5 | Git-Based Service Catalog | `catalog/CLAUDE.md` |
| Epic 6 | Brownfield Modernization | `brownfield/CLAUDE.md` |

---

## Cloud Provider Reference

| Cloud | Terraform Provider | Provider Code (names) | Tags Mechanism |
|-------|-------------------|----------------------|----------------|
| Azure | `hashicorp/azurerm ~> 3.100` | `az` | `tags = {}` block |
| AWS | `hashicorp/aws ~> 5.50` | `aws` | `tags = {}` + `provider default_tags` |
| GCP | `hashicorp/google ~> 5.30` | `gcp` | `labels = {}` (NOT tags — see TAGGING.md) |

---

## Global Conventions

### Terraform
- Minimum version: `>= 1.6.0`
- Required files per module: `main.tf`, `variables.tf`, `outputs.tf`, `versions.tf`, `README.md`
- Module interface: every module exposes `var.tags map(string)` — even on GCP (maps to labels internally)
- Docs: `terraform-docs` auto-generates README variable/output tables

### Naming → `standards/NAMING.md`
Pattern: `{subsidiary}-{env}-{provider}-{resource-type}-{name}`  
Example: `acme-prod-az-vnet-hub`, `contoso-staging-aws-db-postgres-01`  
**Do not invent abbreviations.** Use the tables in `standards/NAMING.md`.

### Tagging → `standards/TAGGING.md`
Five mandatory tags: `environment`, `subsidiary`, `managed-by`, `cost-center`, `owner`  
**Copy the validation block from `standards/TAGGING.md §1` verbatim into every module.**

### Policy Engine
- OPA + Rego via `conftest` for policy-as-code
- Checkov for Terraform static analysis
- tfsec as secondary scanner
- All policy rules reference `standards/NAMING.md` and `standards/TAGGING.md` as source of truth

### Testing
- `terratest` for module integration tests (Go)
- `conftest test` for OPA policy unit tests
- Tests in `tests/` subdirectory inside each module

### Git Workflow
- Branch: `feat/<epic>/<story-description>`
- PR gates: fmt → validate → tfsec → checkov → terratest
- Module release tags: `modules/<level>/<name>/<provider>/v<semver>` e.g. `modules/base/vnet/azure/v1.2.0`

---

## Multi-Cloud Module Interface Contract

All three provider implementations of the same logical module must share an identical
external interface (same variable names, same output names). Only the internal
implementation differs.

```hcl
# Same interface on azure/, aws/, and gcp/:
variable "name"                { type = string }
variable "environment"         { type = string }
variable "subsidiary"          { type = string }
variable "tags"                { type = map(string) }  # GCP maps to labels internally

output "id"                    { }  # Resource ID (ARN on AWS, self_link on GCP, id on Azure)
output "name"                  { }  # Resolved resource name
output "governance_metadata"   { }  # Struct for compliance platform
```

This allows composition modules to swap providers without changing their variable blocks.

---

## Non-Goals (Do NOT Implement)

- Azure DevOps pipelines
- Centralized deployment ownership (subsidiaries own `terraform apply`)
- Management portal (everything is Git-native)
- Forced migration of existing resources to Terraform
- Single-cloud-only implementations (every module needs all 3 providers)
