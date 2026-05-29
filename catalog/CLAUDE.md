# CLAUDE.md — Epic 5: Git-Based Service Catalog

## Epic Goal

Implement a Git-native service catalog that helps subsidiaries discover, understand,
and consume CCoE-approved Terraform modules. No custom portal. No web app to maintain.
Git IS the catalog.

**Ownership:** CCoE Platform Team  
**Principle:** Git as source of truth — everything discoverable via README and directory structure

---

## Stories

| ID | Story | Directory | Priority |
|----|-------|-----------|----------|
| 5.1 | Catalog Structure & Module Entries | `catalog/` | P0 |
| 5.2 | Catalog Automation & Index Generation | `scripts/` + CI | P1 |

---

## Catalog Directory Structure

```
catalog/
├── CATALOG.md                      # Auto-generated index of all catalog entries
├── networking/
│   ├── vnet-standard/
│   │   ├── README.md               # Human-authored description
│   │   ├── metadata.yaml           # Machine-readable catalog metadata
│   │   └── example/
│   │       └── main.tf             # Copy-paste ready example
│   └── hub-spoke/
│       ├── README.md
│       ├── metadata.yaml
│       └── example/
│           └── main.tf
├── compute/
│   ├── metadata.yaml               # Category-level metadata
│   └── README.md
├── database/
│   ├── metadata.yaml
│   └── README.md
└── observability/
    ├── metadata.yaml
    └── README.md
```

---

## Story 5.1 — Catalog Structure & Module Entries

### Purpose

Populate the catalog with entries for every module in `modules/`. Each catalog entry is
the "shop window" for a module — it tells a subsidiary engineer what the module does,
what it costs, what security controls it enforces, and how to use it.

### Catalog Entry File: `metadata.yaml`

```yaml
# catalog/networking/vnet-standard/metadata.yaml
id: net-vnet-standard
name: "Standard Virtual Network"
category: networking
module_source: "ccoe/ccoe-iac-governance//modules/base/vnet"
module_version: "v1.2.0"
status: stable                  # stable | beta | deprecated
tier: base                      # base | composition | landing-zone
owner: "ccoe-platform@company.com"
description: |
  Deploys a virtual network with CCoE-standard subnets, NSG baseline rules,
  and DDoS protection. Encryption in transit enforced by default.
use_cases:
  - "Application workload networking"
  - "Hub-spoke spoke VNet"
security_controls:
  - "Private DNS zones"
  - "NSG baseline rules applied"
  - "No public subnets by default"
cost_guidance: "~$15/month for a 3-subnet VNet in East US"
compliance_profiles:
  - pci-dss-ready
  - hipaa-ready
tags:
  - networking
  - vnet
  - baseline
links:
  documentation: "modules/base/vnet/README.md"
  security: "modules/base/vnet/security.md"
  cost: "modules/base/vnet/cost.md"
  example: "catalog/networking/vnet-standard/example/main.tf"
```

### Entry README Template

```markdown
# {Module Name}

{One-line description}

## When to Use This

{2-3 sentences on the right use case}

## What It Provisions

- {Resource 1}
- {Resource 2}

## Security Controls Included

| Control | Default |
|---------|---------|
| Encryption in transit | Enforced |
| Public endpoints | Disabled |

## Quick Start

```hcl
module "vnet" {
  source  = "ccoe/ccoe-iac-governance//modules/base/vnet"
  version = "~> 1.2"
  
  # ... example variables
}
```

## Inputs & Outputs

<!-- terraform-docs: auto-generated below -->
```

### Entries to Create

| Category | Entry | Module Reference |
|----------|-------|-----------------|
| networking | vnet-standard | `modules/base/vnet` |
| networking | hub-spoke | `modules/composition/secure-platform` |
| compute | vm-standard | `modules/base/vm` |
| compute | container-stack | `modules/composition/container-stack` |
| database | managed-db | `modules/base/database` |
| observability | monitoring-stack | `modules/base/monitoring` (if exists) |

### Acceptance Criteria

- [ ] All catalog entries have `metadata.yaml` conforming to schema
- [ ] All entries have hand-authored `README.md`
- [ ] All entries have working `example/main.tf`
- [ ] `metadata.yaml` schema validated in CI via `ajv` or Python `jsonschema`

---

## Story 5.2 — Catalog Automation & Index Generation

### Purpose

Auto-generate `catalog/CATALOG.md` — a human-readable index of all modules — so
subsidiaries can browse the catalog in GitHub/GitLab without navigating directories.

### Script: `scripts/generate-catalog-index.py`

```python
# Reads all catalog/*/metadata.yaml files
# Outputs catalog/CATALOG.md with:
#   - Table of all modules grouped by category
#   - Status badges (stable / beta / deprecated)
#   - Links to each module's README and example
```

### Generated `CATALOG.md` Format

```markdown
# CCoE Terraform Service Catalog

> Last updated: {timestamp} | Modules: {count}

## Networking

| Module | Description | Status | Version | Example |
|--------|-------------|--------|---------|---------|
| [vnet-standard](networking/vnet-standard/) | Standard Virtual Network | ✅ stable | v1.2.0 | [example](networking/vnet-standard/example/main.tf) |
| [hub-spoke](networking/hub-spoke/) | Hub-Spoke topology | ✅ stable | v1.0.0 | [example](networking/hub-spoke/example/main.tf) |

## Compute

...
```

### Acceptance Criteria

- [ ] `scripts/generate-catalog-index.py` runs in CI on every catalog change
- [ ] CI fails if `CATALOG.md` is out of date (git diff check)
- [ ] Script has `--dry-run` flag that prints diff without writing
- [ ] Deprecated modules shown with ⚠️ badge and migration link

---

## CI Pipeline

File: `.github/workflows/catalog-ci.yml`

Jobs:
1. `validate-metadata` — JSON Schema validation of all `metadata.yaml` files
2. `validate-examples` — `terraform init` + `terraform validate` on all `example/` dirs
3. `generate-index` — Run `scripts/generate-catalog-index.py` and check for diff
4. `link-check` — Verify all `links.*` fields in `metadata.yaml` resolve to real files

---

## Multi-Cloud Catalog Updates (added: multi-cloud update)

Each catalog entry now has a `providers` array. A module with all three providers
gets three separate catalog entries (one per provider) OR a single entry with a
`providers` list — use the single-entry approach for discoverability:

```yaml
# metadata.yaml addition
providers:
  - code: azure
    module_source: "ccoe/ccoe-iac-governance//modules/base/vnet/azure"
    module_version: "v1.2.0"
    status: stable
  - code: aws
    module_source: "ccoe/ccoe-iac-governance//modules/base/vnet/aws"
    module_version: "v1.0.0"
    status: stable
  - code: gcp
    module_source: "ccoe/ccoe-iac-governance//modules/base/vnet/gcp"
    module_version: "v1.0.0"
    status: beta
```

The `scripts/generate-catalog-index.py` must render provider availability badges
in the generated `CATALOG.md`:

```
| vnet-standard | Standard Virtual Network | ✅ Azure · ✅ AWS · 🔶 GCP (beta) |
```

Naming and tagging examples in each `example/main.tf` must use the correct cloud
provider pattern from `standards/NAMING.md` and `standards/TAGGING.md`.
