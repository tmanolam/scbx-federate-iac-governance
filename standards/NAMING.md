# Naming Convention Standard

**Version:** 1.0  
**Owner:** CCoE Platform Team  
**Status:** Enforced — policy `CMP-002` in `policies/compliance/naming-convention.rego`

> This file is the authoritative source. All Terraform modules, pipeline templates,
> and policy rules must derive from the patterns defined here.
> Claude Code must read this file before generating any resource names or name variables.

---

## 1. Universal Pattern

```
{subsidiary}-{env}-{provider}-{resource-type}-{name}[-{index}]
```

| Segment | Required | Values | Max Length |
|---------|----------|--------|------------|
| `subsidiary` | Yes | Lowercase alphanumeric slug, e.g. `acme`, `contoso` | 8 chars |
| `env` | Yes | `prod` \| `staging` \| `dev` \| `sandbox` | — |
| `provider` | Yes | `az` \| `aws` \| `gcp` | — |
| `resource-type` | Yes | See Resource Type Abbreviations below | — |
| `name` | Yes | Short descriptor, kebab-case | 20 chars |
| `index` | No | Zero-padded integer `01`, `02` | — |

**Full pattern max length: 63 characters** (DNS-safe, works across all three clouds)

### Examples

```
acme-prod-az-vnet-hub
acme-prod-aws-vpc-main
contoso-staging-gcp-vpc-shared
acme-prod-az-vm-sapapp-01
acme-dev-aws-rds-postgres-01
contoso-prod-gcp-gke-platform
```

---

## 2. Resource Type Abbreviations

### Networking

| Resource | Azure | AWS | GCP | Abbreviation |
|----------|-------|-----|-----|--------------|
| Virtual Network / VPC | `vnet` | `vpc` | `vpc` | `vnet` |
| Subnet | `snet` | `snet` | `snet` | `snet` |
| Network Security Group | `nsg` | `sg` | `fw` | `nsg` |
| Load Balancer | `lb` | `alb` \| `nlb` | `lb` | `lb` |
| Public IP | `pip` | `eip` | `pip` | `pip` |
| Private Endpoint | `pe` | `vpce` | `psc` | `pe` |
| DNS Zone | `dns` | `dns` | `dns` | `dns` |
| VPN Gateway | `vpng` | `vpng` | `vpng` | `vpng` |

### Compute

| Resource | Azure | AWS | GCP | Abbreviation |
|----------|-------|-----|-----|--------------|
| Virtual Machine | `vm` | `ec2` | `gce` | `vm` |
| VM Scale Set / ASG | `vmss` | `asg` | `mig` | `vmss` |
| Kubernetes Cluster | `aks` | `eks` | `gke` | `k8s` |
| Container Registry | `acr` | `ecr` | `gcr` | `cr` |
| Function / Lambda | `func` | `lambda` | `func` | `func` |

### Storage

| Resource | Azure | AWS | GCP | Abbreviation |
|----------|-------|-----|-----|--------------|
| Object Storage | `st` | `s3` | `gcs` | `st` |
| File Share | `fs` | `efs` | `fs` | `fs` |
| Managed Disk | `disk` | `ebs` | `disk` | `disk` |

### Database

| Resource | Azure | AWS | GCP | Abbreviation |
|----------|-------|-----|-----|--------------|
| Relational DB | `sql` | `rds` | `sql` | `db` |
| NoSQL / Document | `cosmos` | `ddb` | `fstore` | `nosql` |
| Cache | `redis` | `elc` | `redis` | `cache` |
| Data Warehouse | `syn` | `rs` | `bq` | `dw` |

### Security & Identity

| Resource | Azure | AWS | GCP | Abbreviation |
|----------|-------|-----|-----|--------------|
| Key Vault / KMS | `kv` | `kms` | `kms` | `kv` |
| Secrets Manager | `kv` | `sm` | `sm` | `sec` |
| Identity / IAM Role | `id` | `role` | `sa` | `id` |
| Managed Identity / SA | `mi` | `role` | `sa` | `mi` |

### Monitoring & Observability

| Resource | Azure | AWS | GCP | Abbreviation |
|----------|-------|-----|-----|--------------|
| Log Analytics / CloudWatch | `law` | `cw` | `log` | `log` |
| Monitor / CloudTrail | `mon` | `ct` | `mon` | `mon` |
| Alert | `alert` | `alarm` | `alert` | `alert` |

### Resource Containers

| Resource | Azure | AWS | GCP | Abbreviation |
|----------|-------|-----|-----|--------------|
| Resource Group / Account | `rg` | `acc` | `proj` | `rg` |
| Subscription / OU | `sub` | `ou` | `folder` | `sub` |

---

## 3. Environment Values

| Value | Meaning | Notes |
|-------|---------|-------|
| `prod` | Production | Highest policy enforcement |
| `staging` | Pre-production / UAT | Near-prod policy enforcement |
| `dev` | Development | Relaxed SKU restrictions |
| `sandbox` | Experimental / spike | Minimal enforcement, auto-expiry tagging required |

---

## 4. Provider Codes

| Cloud | Code | Used in resource names |
|-------|------|----------------------|
| Microsoft Azure | `az` | Yes |
| Amazon Web Services | `aws` | Yes |
| Google Cloud Platform | `gcp` | Yes |

---

## 5. Terraform Variable Convention

All modules must expose naming via a `name` variable — never construct names internally
from multiple variables unless behind a `local` with a clearly named local value:

```hcl
locals {
  resource_name = "${var.subsidiary}-${var.environment}-${var.provider_code}-${var.resource_type}-${var.name}"
}
```

Modules must validate the assembled name length:

```hcl
validation {
  condition     = length(local.resource_name) <= 63
  error_message = "Assembled resource name exceeds 63 characters. Shorten subsidiary, name, or descriptor segments."
}
```

---

## 6. Special Cases

### Azure Storage Accounts
Storage account names must be globally unique, lowercase alphanumeric, 3–24 chars.
Strip hyphens and truncate:
```
{subsidiary}{env}st{name}  →  acmeprodstbackup
```

### AWS S3 Buckets
Globally unique, 3–63 chars, lowercase. Use full pattern without stripping hyphens:
```
acme-prod-aws-st-assets
```

### GCP Project IDs
6–30 chars, lowercase, hyphens allowed, must start with letter:
```
acme-prod-gcp-{name}  (max 30 chars)
```

### Kubernetes Namespaces
Follow the universal pattern, no provider segment:
```
{subsidiary}-{env}-{name}  →  acme-prod-payments
```

---

## 7. Policy Enforcement

The naming convention is enforced by:

- `policies/compliance/naming-convention.rego` — OPA policy, blocks non-conforming names at plan time
- CI stage: `policy-validate` in all pipeline templates

Subsidiaries requiring a name deviation must open a waiver via `brownfield/exceptions/`.
