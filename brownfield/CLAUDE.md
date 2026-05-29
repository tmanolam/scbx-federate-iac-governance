# CLAUDE.md — Epic 6: Brownfield Modernization

## Epic Goal

Implement a pragmatic brownfield governance model. Existing resources that predate the
Terraform mandate are **grandfathered** — they are not forcibly rebuilt. Instead, they
are registered, scored, tracked, and gradually modernized over time.

**Principle:** Govern Existing, Modernize Gradually — NOT Rebuild Everything in Terraform

**Ownership:** CCoE Governance Team + Subsidiary Leads

---

## Stories

| ID | Story | Directory | Priority |
|----|-------|-----------|----------|
| 6.1 | Legacy Resource Registry | `brownfield/registry/` | P0 |
| 6.2 | Drift & Compliance Scanner | `brownfield/scanner/` | P1 |
| 6.3 | Waiver & Exception Management | `brownfield/exceptions/` | P1 |

---

## Brownfield Governance Model

### Lifecycle Stages

```
New Workload
  → Must use Terraform (no exception)

Existing Workload
  → Register in brownfield registry
  → Get baseline compliance score
  → Assign modernization tier
  → Track progress quarterly
  → Graduate to Terraform when capacity allows
```

### Modernization Tiers

| Tier | Definition | Expected Action |
|------|------------|-----------------|
| T1 — Compliant | Managed by Terraform, all policies pass | No action needed |
| T2 — Monitored | ClickOps but registered, tags present | Modernize within 12 months |
| T3 — At Risk | ClickOps, missing tags, policy violations | Modernize within 6 months |
| T4 — Critical | Unregistered OR active security violation | Immediate remediation |

---

## Story 6.1 — Legacy Resource Registry

### Purpose

A Git-based registry where subsidiaries declare their existing non-Terraform-managed
resources. Registration is mandatory for all pre-existing workloads. An unregistered
resource discovered by the scanner is escalated as T4-Critical.

### File Structure

```
brownfield/registry/
├── schema/
│   └── registration.schema.json    # JSON Schema for registration files
├── subsidiaries/
│   ├── acme/
│   │   ├── erp-platform.yaml       # One file per registered workload
│   │   └── legacy-api.yaml
│   └── contoso/
│       └── finance-system.yaml
├── scripts/
│   └── validate-registry.py        # CI validation script
└── README.md                       # How to register a workload
```

### Registration File Format

```yaml
# brownfield/registry/subsidiaries/acme/erp-platform.yaml
id: acme-erp-platform
subsidiary: acme
workload_name: "ERP Platform"
workload_description: "SAP S/4HANA ERP deployment on Azure VMs"
environment: prod
cloud_provider: azure
region: eastus
registration_date: "2024-01-15"
registered_by: "platform-team@acme.com"

# Modernization tracking
modernization_tier: T2
modernization_target_date: "2025-06-01"
modernization_owner: "infra-team@acme.com"
terraform_migration_issue: "https://github.com/acme/infra/issues/42"

# Resource inventory (best-effort)
resources:
  - type: azurerm_virtual_machine
    count: 4
    estimated: true
  - type: azurerm_sql_server
    count: 1
    estimated: false

# Compliance status
tags_present: partial              # full | partial | none
encryption_at_rest: true
encryption_in_transit: true
known_violations:
  - policy: CMP-001
    description: "Missing cost-center tag on 2 VMs"
    waiver_id: WVR-2024-001        # Reference to exceptions/

# Approval
approved_by: "coe-governance@company.com"
approval_date: "2024-01-20"
```

### Acceptance Criteria

- [ ] `registration.schema.json` validates all required fields
- [ ] `scripts/validate-registry.py` runs in CI and rejects invalid YAML
- [ ] `README.md` explains the registration process step-by-step
- [ ] CI enforces: new `subsidiaries/*/` directories require at least one `.yaml` file
- [ ] Script generates summary report: total registered workloads per tier per subsidiary

---

## Story 6.2 — Drift & Compliance Scanner

### Purpose

A scheduled scanner that detects:
1. **Configuration drift** — Terraform-managed resources changed outside Terraform
2. **Unregistered resources** — Cloud resources with no registry entry and no Terraform state
3. **Tag violations** — Missing mandatory tags on any resource

### Technology

- Language: Python 3.12+
- Cloud APIs: Azure SDK / AWS Boto3 / GCP SDK (pluggable provider pattern)
- Schedule: Daily (run via GitHub Actions scheduled workflow or cron)
- Output: POST to compliance platform collector (`/api/v1/drift-events`)

### File Structure

```
brownfield/scanner/
├── scanner/
│   ├── main.py                   # Entry point: CLI with --provider, --subsidiary flags
│   ├── providers/
│   │   ├── base.py               # Abstract provider interface
│   │   ├── azure.py              # Azure Resource Graph queries
│   │   └── aws.py                # AWS Config / Resource Explorer
│   ├── checks/
│   │   ├── drift_check.py        # Compare cloud state vs Terraform state
│   │   ├── tag_check.py          # Validate mandatory tags
│   │   └── unregistered_check.py # Find resources not in registry
│   ├── reporters/
│   │   └── compliance_api.py     # POST results to compliance platform
│   └── config.py                 # Scanner configuration
├── tests/
├── Dockerfile
└── requirements.txt
```

### Drift Detection Logic

```python
# Pseudocode: drift detection workflow
def detect_drift(subsidiary_id: str, provider: CloudProvider):
    # 1. Query cloud provider for all resources tagged subsidiary=<id>
    cloud_resources = provider.list_resources(tag_filter={"subsidiary": subsidiary_id})
    
    # 2. Load Terraform state files for this subsidiary
    tf_resources = load_terraform_state(subsidiary_id)
    
    # 3. Find resources in cloud but not in Terraform state (ClickOps or unregistered)
    unmanaged = cloud_resources - tf_resources
    
    # 4. Find resources in state but deleted from cloud (state drift)
    missing = tf_resources - cloud_resources
    
    # 5. POST events to compliance platform
    for resource in unmanaged:
        post_drift_event(subsidiary_id, resource, drift_type="unmanaged")
```

### Acceptance Criteria

- [ ] Scanner runs end-to-end against a mock Azure provider in tests
- [ ] `--dry-run` flag prints findings without POSTing to compliance API
- [ ] `--subsidiary` flag scopes scan to one subsidiary
- [ ] Findings include resource ID, type, subsidiary, violation type, timestamp
- [ ] Scanner exits with code 1 if T4-Critical findings are detected (CI integration)
- [ ] Docker image deployable as scheduled job

---

## Story 6.3 — Waiver & Exception Management

### Purpose

A Git-based workflow for subsidiaries to request time-limited policy exceptions
(waivers). All waivers are version-controlled, require CCoE approval, and auto-expire.

**Principle:** Exceptions are allowed. Untracked exceptions are not.

### File Structure

```
brownfield/exceptions/
├── schema/
│   └── waiver.schema.yaml         # Waiver file schema
├── waivers/
│   ├── active/
│   │   └── WVR-2024-001.yaml      # Active waivers
│   └── expired/
│       └── WVR-2023-001.yaml      # Auto-archived expired waivers
├── scripts/
│   ├── validate-waivers.py        # Schema validation
│   └── expire-waivers.py          # Move expired waivers to expired/
└── README.md                      # Waiver request process
```

### Waiver File Format

```yaml
# brownfield/exceptions/waivers/active/WVR-2024-001.yaml
id: WVR-2024-001
workload_id: acme-erp-platform       # Must match brownfield/registry entry
subsidiary: acme
policy_id: CMP-001                   # Policy being waived
policy_name: "mandatory-tags"

reason: |
  Legacy SAP deployment predates tagging standard. Tag remediation in progress.
  VMs are managed by vendor and require change window for tag application.

exception_scope:
  resources:
    - "azurerm_virtual_machine:acme-prod-vm-sap01"
    - "azurerm_virtual_machine:acme-prod-vm-sap02"

timeline:
  requested_date: "2024-01-15"
  approved_date: "2024-01-20"
  expiry_date: "2024-07-15"          # Maximum 6 months
  remediation_target: "2024-07-01"

approvals:
  - approver: "coe-governance@company.com"
    role: "CCoE Governance Lead"
    date: "2024-01-20"
  - approver: "ciso@company.com"
    role: "CISO"
    date: "2024-01-21"

status: active                        # active | expired | revoked
escalation_if_overdue: "ciso@company.com"
```

### Waiver Request Process (via Pull Request)

```
1. Subsidiary creates: brownfield/exceptions/waivers/active/WVR-{YEAR}-{NNN}.yaml
2. Opens PR with label: waiver-request
3. CI validates schema
4. CCoE reviewer approves in PR (required approvers enforced via CODEOWNERS)
5. Merge = approved waiver
6. Nightly job checks expiry dates and moves expired to expired/
```

### CODEOWNERS Entry

```
brownfield/exceptions/waivers/    @ccoe-governance-team @ciso-team
```

### Acceptance Criteria

- [ ] Waiver schema validation enforces: id, workload_id, policy_id, expiry_date, approvals
- [ ] Expiry date cannot exceed 6 months from requested_date (enforced in CI)
- [ ] `scripts/expire-waivers.py` runs daily in CI and creates PR to archive expired waivers
- [ ] Compliance platform API returns active waivers when queried (`GET /api/v1/waivers/{subsidiary}`)
- [ ] CODEOWNERS enforces CCoE approval on all waiver changes
- [ ] `README.md` documents the full waiver lifecycle with examples

---

## CI Pipeline

File: `.github/workflows/brownfield-ci.yml`

Jobs:
1. `validate-registry` — Schema validation of all `brownfield/registry/subsidiaries/**/*.yaml`
2. `validate-waivers` — Schema validation + expiry date check
3. `expire-check` — Warn on waivers expiring within 30 days
4. `scanner-test` — Pytest on `brownfield/scanner/tests/`
5. `generate-brownfield-report` — Summary of registered workloads by tier (comment on PR)

---

## Multi-Cloud Scanner Notes (added: multi-cloud update)

The drift scanner (`brownfield/scanner/`) must support all three cloud providers.
`standards/NAMING.md` defines the expected name pattern — the scanner uses it to
identify CCoE-managed resources vs unmanaged ones.

### Provider Detection in Registry

Add `cloud_provider` to `brownfield/registry/schema/registration.schema.yaml`:

```yaml
cloud_provider:
  type: string
  enum: [azure, aws, gcp]
  required: true
```

### Scanner Resource Queries Per Provider

| Provider | Query Mechanism | Tag/Label Filter |
|----------|----------------|-----------------|
| Azure | Azure Resource Graph (`az graph query`) | `tags.subsidiary` |
| AWS | AWS Resource Explorer / Config | `tags.subsidiary` |
| GCP | Cloud Asset Inventory | `labels.subsidiary` |

Note: GCP uses `labels`, not `tags`. The scanner must query `labels.subsidiary`
on GCP resources. See `standards/TAGGING.md §3`.
