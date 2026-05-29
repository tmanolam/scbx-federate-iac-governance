# ROADMAP.md

# Federated Terraform IaC Governance Roadmap

**Version:** 1.0  
**Target Model:** Federated Governance (Option B)  
**IaC Standard:** Terraform only  
**Deployment Model:** Decentralized execution with centralized governance  
**CI/CD Platform:** Agnostic (No Azure DevOps dependency)

---

# 1. Executive Summary

This roadmap defines a federated Infrastructure-as-Code (IaC) governance model for enterprise subsidiaries operating independently across cloud environments while maintaining centralized governance, security, compliance, and visibility.

The proposed model enables:

- Central governance without centralized deployment bottleneck
- Subsidiary autonomy
- Terraform standardization
- Policy enforcement
- Progressive brownfield modernization
- Git-native service catalog
- Multi-CI/CD support
- Enterprise-scale compliance visibility

This roadmap intentionally avoids:

- Central management portal ownership
- Azure DevOps dependency
- Big-bang cloud migration
- Forced Terraform rebuild of legacy environments
- Centralized deployment ownership

The operating principle is:

> Govern centrally, build locally

---

# 2. Problem Statement

Most cloud governance initiatives fail due to one of two extremes.

## Model A — Over-centralized Governance

```text
CCoE owns everything
↓
All deployments routed to central team
↓
Delivery bottleneck
↓
Subsidiary frustration
↓
Shadow IT
```

### Problems

- Slow delivery
- Platform bottleneck
- Engineering resistance
- Low adoption
- Unscalable operating model

---

## Model B — No Governance

```text
Subsidiaries operate independently
↓
Inconsistent tooling
↓
No visibility
↓
Security drift
↓
Compliance failures
```

### Problems

- Uncontrolled cloud sprawl
- Security inconsistency
- Tagging issues
- Cost explosion
- Audit risk

---

## Target State

A federated governance model.

```text
CCoE defines guardrails
Subsidiaries deploy independently
```

---

# 3. Strategic Objectives

## 3.1 Terraform Standardization

All **new infrastructure** must be provisioned through Terraform.

Objective:

- Eliminate click-ops
- Improve repeatability
- Improve auditability
- Reduce configuration drift

---

## 3.2 Federated Autonomy

Subsidiaries remain responsible for:

- Workloads
- Deployment execution
- CI/CD ownership
- Repository ownership
- Environment operation

---

## 3.3 Central Governance

CCoE must gain:

- Compliance visibility
- Security governance
- Policy enforcement
- Adoption tracking
- Risk reporting

Without becoming a bottleneck.

---

## 3.4 Brownfield Modernization

Existing environments must be governed progressively.

Principle:

> Govern Existing, Modernize Gradually

NOT:

> Rebuild everything in Terraform

---

# 4. Guiding Principles

## Principle 1 — Terraform Mandatory for New Resources

All new cloud infrastructure must use Terraform.

Forbidden:

- Manual production provisioning
- Portal-based provisioning
- ClickOps

---

## Principle 2 — Existing Resources Are Grandfathered

Existing workloads are temporarily allowed.

Requirements:

- Registration
- Visibility
- Compliance scoring
- Modernization tracking

---

## Principle 3 — Guardrails, Not Gates

CCoE should provide:

```text
Golden path
```

Instead of:

```text
Approval bottleneck
```

---

## Principle 4 — Git as Source of Truth

No management portal required.

Everything must be Git-based:

- Modules
- Catalog
- Policies
- Exceptions
- Pipelines
- Documentation

---

## Principle 5 — Open CI/CD Ecosystem

Supported:

- Jenkins
- GitHub Actions
- GitLab CI

No mandatory platform.

---

## Principle 6 — Progressive Enforcement

Enforcement maturity:

```text
Observe
→ Warn
→ Enforce
```

---

# 5. Target Operating Model

## Option B — Federated Governance

### CCoE Responsibilities

CCoE owns:

### Governance

- Security standards
- Policy framework
- Tagging standards
- Compliance baseline

### Platform Assets

- Terraform modules
- Pipeline templates
- Policy library
- Golden patterns

### Visibility

- Compliance dashboard
- Adoption tracking
- Drift monitoring
- Executive reporting

### Exception Management

- Waivers
- Legacy approvals
- Modernization oversight

---

### Subsidiary Responsibilities

Subsidiaries own:

### Delivery

- Application delivery
- Infrastructure deployment
- Repo ownership

### Terraform Execution

- terraform plan
- terraform apply
- State ownership

### Operations

- Support
- Incident management
- SLA ownership
- Patching

---

## Ownership Boundary

```text
CCoE defines HOW

Subsidiaries decide WHAT
```

Example:

CCoE:

```text
Approved networking standard
```

Subsidiary:

```text
Deploy ERP platform
```

---

# 6. High-Level Architecture

```text
                    ┌──────────────────────┐
                    │         CCoE         │
                    ├──────────────────────┤
                    │ Terraform Modules    │
                    │ Policy Library       │
                    │ Pipeline Templates   │
                    │ Compliance Platform  │
                    │ Governance Rules     │
                    └──────────┬───────────┘
                               │
              ┌────────────────┼────────────────┐
              │                │                │
              ▼                ▼                ▼
      Subsidiary A      Subsidiary B      Subsidiary C
      ┌────────────┐    ┌────────────┐    ┌────────────┐
      │ Git Repo   │    │ Git Repo   │    │ Git Repo   │
      │ Terraform  │    │ Terraform  │    │ Terraform  │
      │ CI/CD      │    │ CI/CD      │    │ CI/CD      │
      └─────┬──────┘    └─────┬──────┘    └─────┬──────┘
            │                  │                  │
            ▼                  ▼                  ▼
       Cloud Account      Cloud Account     Cloud Account
```

Key principle:

Subsidiaries deploy independently.

CCoE governs centrally.

---

# 7. Governance Layers

## Layer 1 — Preventive Controls

Prevent insecure deployment.

Examples:

- Mandatory tags
- Encryption enforcement
- Approved regions
- Approved SKUs
- Network restrictions

---

## Layer 2 — Detective Controls

Detect violations.

Examples:

- Drift detection
- Manual changes
- Missing tags
- Unmanaged resources

---

## Layer 3 — Corrective Controls

Remediation workflow.

```text
Violation detected
→ notify owner
→ remediation SLA
→ escalation
```

---

## Layer 4 — Reporting Controls

Executive visibility.

Metrics:

- Terraform adoption
- Policy compliance
- Drift percentage
- Exception count
- Cloud maturity

---

# 8. CCoE-Owned Components

## 8.1 Terraform Module Registry

Reusable modules.

Examples:

```text
network
identity
storage
database
monitoring
kubernetes
landing-zone
```

Requirements:

- Versioned
- Security reviewed
- Documented
- Backward compatible

---

## 8.2 Policy Library

Reusable governance policies.

### Security

```text
No public database
Encryption mandatory
Private endpoint required
```

### Compliance

```text
Mandatory tags
Naming convention
Approved regions
```

### Cost

```text
VM SKU restriction
Budget guardrails
```

---

## 8.3 Pipeline Template Library

Reusable CI/CD templates.

Supported:

- Jenkins shared library
- GitHub reusable workflow
- GitLab shared include

---

## 8.4 Compliance Platform

Central governance visibility.

Collect:

- Terraform adoption
- Policy violations
- Drift detection
- Runtime metadata
- Exception tracking
- Repo compliance

---

# 9. Git-Based Service Catalog (No Portal)

## Objective

Avoid centralized management portal ownership.

Use Git as service catalog.

Benefits:

- Low maintenance
- Developer-friendly
- No custom portal
- Scalable

---

## Repository Structure

```text
terraform-catalog/
├── networking/
│   ├── vnet-standard/
│   └── hub-spoke/
├── compute/
├── database/
└── observability/
```

Each module includes:

```text
README.md
examples/
variables.tf
security.md
cost.md
```

---

# 10. Terraform Module Strategy

## Level 1 — Base Modules

Atomic resources.

Examples:

```text
vm
vnet
storage
database
keyvault
```

---

## Level 2 — Composition Modules

Reusable architecture patterns.

Examples:

```text
3-tier-app
secure-platform
container-stack
```

---

## Level 3 — Landing Zone Modules

Opinionated enterprise blueprints.

Examples:

```text
regulated-workload
internet-facing-app
data-platform
```

---

## Consumption Policy

Priority:

1. Enterprise approved module
2. Custom Terraform with compliance
3. Manual provisioning (forbidden)

---

# 11. CI/CD Governance Model

Subsidiaries continue using:

- Jenkins
- GitHub Actions
- GitLab CI

CCoE governs through:

> Pipeline contract

Required stages:

```text
fmt
validate
security scan
policy validation
plan
approval
apply
drift scan
```

---

# 12. Pipeline Adoption Detection

CCoE must know whether subsidiaries follow standards.

## Signal 1 — Template Detection

Example:

Jenkins

```groovy
@Library('ccoe-pipeline')
```

GitHub

```yaml
uses: org/terraform-template
```

GitLab

```yaml
include:
  - project: ccoe/pipeline
```

## Signal 2 — Required Stages

Detect missing:

```text
policy validation
security scan
```

## Signal 3 — Runtime Metadata

Pipeline sends metadata:

```json
{
  "subsidiary": "companyA",
  "repo": "erp-platform",
  "templateVersion": "v2.1"
}
```

## Compliance Score

| Level | Meaning |
|-------|---------|
| Gold | Official template |
| Silver | Equivalent controls |
| Bronze | Partial adoption |
| Red | Non-compliant |

---
