# Tagging Standard

**Version:** 1.0  
**Owner:** CCoE Platform Team  
**Status:** Enforced — policy `CMP-001` in `policies/compliance/mandatory-tags.rego`

> This file is the authoritative source for all resource tagging.
> Claude Code must read this file before generating any `tags` variable, tag validation,
> or policy rule related to tagging.

---

## 1. Mandatory Tags

These five tags are **required on every cloud resource** across Azure, AWS, and GCP.
Missing any of these causes policy `CMP-001` to DENY the deployment.

| Tag Key | Description | Allowed Values | Example |
|---------|-------------|----------------|---------|
| `environment` | Deployment environment | `prod` \| `staging` \| `dev` \| `sandbox` | `prod` |
| `subsidiary` | Owning subsidiary identifier | Lowercase slug, max 8 chars | `acme` |
| `managed-by` | Provisioning method | `terraform` \| `manual` (manual triggers alert) | `terraform` |
| `cost-center` | Finance cost allocation code | Alphanumeric, format `CC-{4digits}` | `CC-1042` |
| `owner` | Team email responsible for the resource | Valid email address | `platform@acme.com` |

### Terraform Validation Block (copy into every module)

```hcl
variable "tags" {
  description = "Mandatory resource tags. Must include: environment, subsidiary, managed-by, cost-center, owner."
  type        = map(string)

  validation {
    condition = alltrue([
      for k in ["environment", "subsidiary", "managed-by", "cost-center", "owner"] :
      contains(keys(var.tags), k)
    ])
    error_message = "tags must contain all mandatory keys: environment, subsidiary, managed-by, cost-center, owner."
  }

  validation {
    condition     = contains(["prod", "staging", "dev", "sandbox"], var.tags["environment"])
    error_message = "tags.environment must be one of: prod, staging, dev, sandbox."
  }

  validation {
    condition     = var.tags["managed-by"] == "terraform"
    error_message = "tags.managed-by must be 'terraform'. Manual resources must use the brownfield registry."
  }

  validation {
    condition     = can(regex("^CC-[0-9]{4}$", var.tags["cost-center"]))
    error_message = "tags.cost-center must match format CC-NNNN (e.g. CC-1042)."
  }

  validation {
    condition     = can(regex("^[^@]+@[^@]+\\.[^@]+$", var.tags["owner"]))
    error_message = "tags.owner must be a valid email address."
  }
}
```

---

## 2. Recommended Tags

These tags are **strongly recommended**. Missing them generates a WARNING in the
compliance platform but does not block deployment.

| Tag Key | Description | Example |
|---------|-------------|---------|
| `project` | Project or product name | `erp-migration` |
| `team` | Engineering team name | `platform-eng` |
| `repo` | Source repository URL | `github.com/acme/infra` |
| `tf-module` | Terraform module that created the resource | `ccoe/vnet/v1.2.0` |
| `created-date` | ISO 8601 creation date | `2024-01-15` |
| `expiry-date` | For sandbox/temp resources: auto-expiry date | `2024-03-01` |
| `data-classification` | Data sensitivity level | `confidential` \| `internal` \| `public` |
| `backup-policy` | Backup tier | `daily` \| `weekly` \| `none` |
| `dr-tier` | Disaster recovery tier | `tier1` \| `tier2` \| `none` |

---

## 3. Provider-Specific Implementation

### Azure

Azure uses the `tags` block natively.

```hcl
resource "azurerm_resource_group" "this" {
  name     = local.resource_name
  location = var.location
  tags     = var.tags
}
```

Tag key constraints: max 512 chars key, max 256 chars value, max 50 tags per resource.

---

### AWS

AWS uses the `tags` block. The `default_tags` pattern on the provider is recommended
so mandatory tags propagate automatically:

```hcl
# In provider configuration (pipeline template injects this)
provider "aws" {
  default_tags {
    tags = {
      environment  = var.environment
      subsidiary   = var.subsidiary
      "managed-by" = "terraform"
    }
  }
}

# Module-level tags merged on top
resource "aws_vpc" "this" {
  cidr_block = var.cidr_block
  tags       = merge(var.tags, { Name = local.resource_name })
}
```

AWS tag key constraints: max 128 chars key, max 256 chars value, max 50 tags per resource.
AWS requires the `Name` tag to appear in the console — always set it to `local.resource_name`.

---

### GCP

GCP uses `labels` (not `tags` — GCP tags are network firewall tags, different concept).

```hcl
# In modules, the variable is still called "tags" for interface consistency
# but it maps to GCP labels internally

variable "tags" {
  # same mandatory validation as above
}

locals {
  # GCP labels must be lowercase, max 63 chars, letters/numbers/hyphens/underscores only
  gcp_labels = {
    for k, v in var.tags :
    replace(lower(k), "/[^a-z0-9_-]/", "-") => replace(lower(v), "/[^a-z0-9_-]/", "-")
  }
}

resource "google_compute_network" "this" {
  name   = local.resource_name
  labels = local.gcp_labels
}
```

GCP label constraints: max 64 labels, key max 63 chars, value max 63 chars, lowercase only.

> **Important:** GCP network tags (firewall) are separate from labels. Never confuse them.
> Modules use `var.tags` → GCP labels. Firewall tags are a separate `var.network_tags` input.

---

## 4. Tag Inheritance Model

```
Landing Zone module sets: environment, subsidiary, managed-by, cost-center, owner
      ↓
Composition module merges: project, team, tf-module
      ↓
Base module merges: resource-specific tags
      ↓
Final resource: all tags merged, module tags cannot override mandatory tags
```

Merge pattern in modules:

```hcl
locals {
  # Mandatory tags from caller + module-injected tags
  # Caller tags take precedence for everything except managed-by (always "terraform")
  final_tags = merge(
    var.tags,
    {
      "managed-by" = "terraform"
      "tf-module"  = "ccoe/vnet/v${local.module_version}"
    }
  )
}
```

---

## 5. Sandbox Auto-Expiry

Resources tagged `environment = "sandbox"` **must** carry an `expiry-date` tag.
The compliance scanner (`brownfield/scanner/`) flags sandbox resources with an
`expiry-date` in the past and triggers automatic remediation.

```hcl
validation {
  condition = (
    var.tags["environment"] != "sandbox" ||
    can(regex("^[0-9]{4}-[0-9]{2}-[0-9]{2}$", lookup(var.tags, "expiry-date", "")))
  )
  error_message = "Sandbox resources must carry an expiry-date tag in YYYY-MM-DD format."
}
```

---

## 6. OPA Policy Reference

Tag enforcement is implemented in:

```
policies/compliance/mandatory-tags.rego   ← DENY on missing mandatory tags
policies/compliance/tag-values.rego       ← DENY on invalid tag values  
policies/compliance/sandbox-expiry.rego   ← WARN on missing sandbox expiry-date
```

---

## 7. Tag Quick Reference Card

```
MANDATORY (DENY if missing):
  environment      = prod | staging | dev | sandbox
  subsidiary       = <slug max 8 chars>
  managed-by       = terraform
  cost-center      = CC-NNNN
  owner            = team@company.com

RECOMMENDED (WARN if missing):
  project          = <project-name>
  team             = <team-name>
  repo             = <git-url>
  tf-module        = <module-path/version>
  data-classification = confidential | internal | public

SANDBOX ONLY (DENY if missing on sandbox):
  expiry-date      = YYYY-MM-DD
```
