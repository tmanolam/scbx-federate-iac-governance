# CLAUDE.md — Epic 2: Policy Library

## Epic Goal

Build a reusable, versioned library of governance policies enforced via OPA/Rego and
Checkov. Policies gate the CI/CD pipeline. Subsidiaries cannot deploy infrastructure
that violates an enforced policy.

**Enforcement maturity model:** Observe → Warn → Enforce  
**Ownership:** CCoE Security & Compliance Team

---

## Stories

| ID | Story | Directory | Priority |
|----|-------|-----------|----------|
| 2.1 | Security Policies | `policies/security/` | P0 |
| 2.2 | Compliance Policies | `policies/compliance/` | P0 |
| 2.3 | Cost Policies | `policies/cost/` | P1 |

---

## Policy Framework

### Toolchain

| Tool | Role |
|------|------|
| OPA + Rego | Policy-as-code, evaluated via `conftest` |
| Checkov | Terraform static analysis (built-in policies + custom) |
| tfsec | Supplementary Terraform security scanner |

### Policy File Structure

```
policies/
├── security/
│   ├── no-public-database.rego
│   ├── encryption-mandatory.rego
│   ├── private-endpoint-required.rego
│   └── tests/
│       ├── no-public-database_test.rego
│       └── ...
├── compliance/
│   ├── mandatory-tags.rego
│   ├── naming-convention.rego
│   ├── approved-regions.rego
│   └── tests/
├── cost/
│   ├── vm-sku-restriction.rego
│   ├── budget-guardrails.rego
│   └── tests/
└── policy.yaml             # Policy metadata: severity, enforcement level, owner
```

### Policy Metadata File (`policy.yaml`)

Every policy directory must have a `policy.yaml`:

```yaml
policies:
  - id: SEC-001
    name: no-public-database
    file: no-public-database.rego
    severity: CRITICAL
    enforcement: enforce        # observe | warn | enforce
    owner: security-team@company.com
    description: "Database instances must not have public IP or public access enabled"
    references:
      - "CIS Benchmark 4.3.1"
```

---

## Story 2.1 — Security Policies

### Policies to Implement

| Policy ID | Name | Rule |
|-----------|------|------|
| SEC-001 | no-public-database | Database must not expose public endpoint |
| SEC-002 | encryption-mandatory | All storage and database must have encryption at rest |
| SEC-003 | private-endpoint-required | KeyVault and database must use private endpoints |
| SEC-004 | no-public-storage-blob | Storage containers must not be publicly readable |
| SEC-005 | tls-minimum-version | All services must enforce TLS 1.2+ |

### Rego Implementation Pattern

```rego
# policies/security/no-public-database.rego
package terraform.security

import future.keywords.if
import future.keywords.in

deny[msg] if {
  resource := input.resource_changes[_]
  resource.type == "azurerm_mssql_server"
  resource.change.after.public_network_access_enabled == true
  msg := sprintf(
    "[SEC-001] CRITICAL: Resource '%s' has public network access enabled. Databases must use private endpoints only.",
    [resource.address]
  )
}
```

### Acceptance Criteria

- [ ] All five security policies implemented in Rego
- [ ] Each policy has unit tests in `tests/` with both PASS and FAIL cases
- [ ] `conftest test policies/security/tests/` passes 100%
- [ ] `policy.yaml` populated with severity=CRITICAL and enforcement=enforce for all
- [ ] Policies are evaluated in the `security scan` stage of pipeline templates

---

## Story 2.2 — Compliance Policies

### Policies to Implement

| Policy ID | Name | Rule |
|-----------|------|------|
| CMP-001 | mandatory-tags | All resources must carry the five mandatory tags |
| CMP-002 | naming-convention | Resources must follow `{subsidiary}-{env}-{type}-{name}` |
| CMP-003 | approved-regions | Deployments allowed only in approved region list |
| CMP-004 | managed-by-terraform | Resources must carry `managed-by = "terraform"` tag |

### Approved Regions Configuration

```rego
# policies/compliance/approved-regions.rego
package terraform.compliance

approved_regions := {
  "eastus", "eastus2", "westeurope", "southeastasia"
}

deny[msg] if {
  resource := input.resource_changes[_]
  resource.change.after.location
  not resource.change.after.location in approved_regions
  msg := sprintf(
    "[CMP-003] Resource '%s' is in unapproved region '%s'",
    [resource.address, resource.change.after.location]
  )
}
```

### Tag Enforcement Pattern

```rego
# policies/compliance/mandatory-tags.rego
required_tags := {"environment", "subsidiary", "managed-by", "cost-center", "owner"}

deny[msg] if {
  resource := input.resource_changes[_]
  resource.change.after.tags
  missing := required_tags - {tag | resource.change.after.tags[tag]}
  count(missing) > 0
  msg := sprintf(
    "[CMP-001] Resource '%s' is missing required tags: %v",
    [resource.address, missing]
  )
}
```

### Acceptance Criteria

- [ ] All four compliance policies implemented and tested
- [ ] Approved regions list is externally configurable (YAML input to conftest)
- [ ] Naming convention policy uses regex validation
- [ ] Tag policy distinguishes between WARN (missing optional tags) and DENY (missing mandatory tags)

---

## Story 2.3 — Cost Policies

### Policies to Implement

| Policy ID | Name | Rule | Enforcement |
|-----------|------|------|-------------|
| CST-001 | vm-sku-restriction | Only approved VM SKU sizes allowed | warn initially, enforce after 30 days |
| CST-002 | budget-guardrails | Resource count per deployment cannot exceed limits | warn |
| CST-003 | no-dev-in-prod | Dev-tier SKUs must not appear in prod environments | enforce |

### Approved VM SKU List (configurable)

```yaml
# policies/cost/approved-vm-skus.yaml
approved_skus:
  prod:
    - Standard_D4s_v5
    - Standard_D8s_v5
    - Standard_E4s_v5
  dev:
    - Standard_B2s
    - Standard_B4ms
    - Standard_D2s_v5
```

### Acceptance Criteria

- [ ] SKU list is data-driven (YAML config), not hardcoded in Rego
- [ ] `conftest` can accept `--data policies/cost/approved-vm-skus.yaml`
- [ ] All policies have tests

---

## Policy CI Pipeline

File: `.github/workflows/policies-ci.yml`

Required jobs:
1. `lint` — `opa fmt --diff policies/`
2. `test` — `conftest test policies/*/tests/`
3. `verify-metadata` — script validates `policy.yaml` schema in each subdirectory
4. `checkov-custom` — validate custom Checkov checks compile

---

## Adding a New Policy (Runbook)

1. Create `policies/<category>/<policy-id>-<name>.rego`
2. Create `policies/<category>/tests/<policy-id>-<name>_test.rego` with PASS + FAIL cases
3. Add entry to `policies/<category>/policy.yaml`
4. Start enforcement as `observe`, graduate to `warn`, then `enforce`
5. Open PR — CI must pass before merge

---

## Multi-Cloud Policy Notes (added: multi-cloud update)

Policies evaluate `terraform plan -out` JSON via `conftest`. The resource type names
differ per provider — policies must handle all three:

### Resource Type Mapping for Policy Rules

| Logical Resource | Azure Type | AWS Type | GCP Type |
|-----------------|------------|----------|----------|
| Virtual Machine | `azurerm_linux_virtual_machine` | `aws_instance` | `google_compute_instance` |
| Database | `azurerm_mssql_server` | `aws_db_instance` | `google_sql_database_instance` |
| Storage | `azurerm_storage_account` | `aws_s3_bucket` | `google_storage_bucket` |
| Key Vault / KMS | `azurerm_key_vault` | `aws_kms_key` | `google_kms_crypto_key` |
| Network | `azurerm_virtual_network` | `aws_vpc` | `google_compute_network` |

### Rego Pattern for Multi-Cloud Coverage

Use a set of types rather than a single type check:

```rego
# policies/security/encryption-mandatory.rego
database_types := {
  "azurerm_mssql_server",
  "aws_db_instance",
  "google_sql_database_instance"
}

deny[msg] if {
  resource := input.resource_changes[_]
  resource.type in database_types
  not encryption_enabled(resource)
  msg := sprintf("[SEC-002] %s '%s' does not have encryption enabled", [resource.type, resource.address])
}
```

### Naming Convention Policy (references standards/NAMING.md)

`policies/compliance/naming-convention.rego` must validate the pattern
`{subsidiary}-{env}-{provider}-{type}-{name}` using the abbreviation tables
from `standards/NAMING.md`. The approved provider codes are: `az`, `aws`, `gcp`.

```rego
valid_providers := {"az", "aws", "gcp"}
valid_envs      := {"prod", "staging", "dev", "sandbox"}

# Extract segments and validate
deny[msg] if {
  resource := input.resource_changes[_]
  name := resource.change.after.name
  parts := split(name, "-")
  count(parts) >= 5
  not parts[2] in valid_providers
  msg := sprintf("[CMP-002] Resource '%s' has invalid provider code '%s'. Must be az|aws|gcp", [resource.address, parts[2]])
}
```

### Tagging Policy (references standards/TAGGING.md)

`policies/compliance/mandatory-tags.rego` enforces the five mandatory tags.
GCP resources use `labels` not `tags` — the policy must check both fields:

```rego
# Check tags (Azure/AWS) OR labels (GCP)
get_tags(resource) := tags if {
  tags := resource.change.after.tags
} else := tags if {
  tags := resource.change.after.labels
} else := {}
```
