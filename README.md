# CCoE Federated IaC Governance

> **Govern centrally. Build locally. Deploy anywhere.**

Multi-cloud (Azure · AWS · GCP) federated Terraform governance platform.
Subsidiaries deploy independently; CCoE provides guardrails, not gates.

## Quick Navigation

| What you need | Where to go |
|---------------|-------------|
| **Naming & tagging standards** | [`standards/`](standards/) |
| Terraform modules to use | [`catalog/CATALOG.md`](catalog/) |
| Module source code | [`modules/`](modules/) |
| Governance policies | [`policies/`](policies/) |
| Pipeline templates | [`pipeline-templates/`](pipeline-templates/) |
| Register an existing workload | [`brownfield/registry/`](brownfield/registry/) |
| Request a policy exception | [`brownfield/exceptions/`](brownfield/exceptions/) |

## For CCoE Developers — Start Here

Read [`CLAUDE.md`](CLAUDE.md) for full architecture, conventions, and epic index.
Read [`standards/CLAUDE.md`](standards/CLAUDE.md) for when to consult naming/tagging standards.

## Supported Cloud Providers

| Cloud | Provider Code | Terraform Provider |
|-------|-------------|-------------------|
| Azure | `az` | `hashicorp/azurerm ~> 3.100` |
| AWS | `aws` | `hashicorp/aws ~> 5.50` |
| GCP | `gcp` | `hashicorp/google ~> 5.30` |

## Epics

| # | Epic | Status |
|---|------|--------|
| — | Standards (Naming & Tagging) | ✅ Defined |
| 1 | Terraform Module Registry (3-cloud) | 🚧 In Progress |
| 2 | Policy Library | 🚧 In Progress |
| 3 | Pipeline Template Library | 📋 Planned |
| 4 | Compliance Platform | 📋 Planned |
| 5 | Git-Based Service Catalog | 📋 Planned |
| 6 | Brownfield Modernization | 📋 Planned |
