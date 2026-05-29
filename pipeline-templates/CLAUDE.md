# CLAUDE.md — Epic 3: Pipeline Template Library

## Epic Goal

Provide reusable, CCoE-governed CI/CD pipeline templates for all three supported
platforms (GitHub Actions, GitLab CI, Jenkins). Subsidiaries adopt these templates to
get a compliant Terraform deployment pipeline without building one from scratch.

**Ownership:** CCoE Platform Engineering  
**Consumers:** All subsidiary engineering teams

---

## Pipeline Contract

Regardless of platform, every compliant Terraform pipeline **must** execute these
stages in order:

```
1. fmt              terraform fmt -check
2. validate         terraform validate
3. security-scan    tfsec + checkov
4. policy-validate  conftest against policies/
5. plan             terraform plan (output saved as artifact)
6. approval         Manual gate (prod environments only)
7. apply            terraform apply
8. drift-scan       Post-apply drift detection (scheduled, not on every run)
```

Missing any of stages 1–5 results in a **non-compliant** pipeline score.

---

## Stories

| ID | Story | Directory | Priority |
|----|-------|-----------|----------|
| 3.1 | GitHub Actions Reusable Workflows | `pipeline-templates/github-actions/` | P0 |
| 3.2 | GitLab CI Shared Includes | `pipeline-templates/gitlab-ci/` | P1 |
| 3.3 | Jenkins Shared Library | `pipeline-templates/jenkins/` | P1 |

---

## Story 3.1 — GitHub Actions Reusable Workflows

### File Structure

```
pipeline-templates/github-actions/
├── terraform-plan.yml        # Reusable workflow: stages 1–5
├── terraform-apply.yml       # Reusable workflow: stage 7
├── terraform-drift.yml       # Reusable workflow: stage 8 (scheduled)
├── terraform-full.yml        # Composite: all stages (calls above workflows)
└── README.md
```

### Usage by Subsidiaries

```yaml
# Subsidiary repo: .github/workflows/deploy.yml
jobs:
  terraform:
    uses: org/ccoe-iac-governance/.github/workflows/terraform-full.yml@v2
    with:
      working_directory: ./infra
      environment: prod
      subsidiary_id: acme
    secrets: inherit
```

### Reusable Workflow Inputs

```yaml
# terraform-full.yml inputs
inputs:
  working_directory:
    description: "Path to Terraform root module"
    required: true
    type: string
  environment:
    description: "Target environment: dev | staging | prod"
    required: true
    type: string
  subsidiary_id:
    description: "Subsidiary identifier for compliance metadata"
    required: true
    type: string
  terraform_version:
    description: "Terraform version to use"
    required: false
    default: "1.9.0"
    type: string
  policy_enforcement:
    description: "Policy enforcement mode: observe | warn | enforce"
    required: false
    default: "enforce"
    type: string
```

### Compliance Metadata Emission

Each pipeline run must POST metadata to the compliance platform:

```yaml
- name: Emit compliance metadata
  run: |
    curl -s -X POST "${{ vars.COMPLIANCE_API_URL }}/api/v1/pipeline-runs" \
      -H "Authorization: Bearer ${{ secrets.COMPLIANCE_API_TOKEN }}" \
      -H "Content-Type: application/json" \
      -d '{
        "subsidiary": "${{ inputs.subsidiary_id }}",
        "repo": "${{ github.repository }}",
        "ref": "${{ github.ref }}",
        "workflow": "${{ github.workflow }}",
        "template_version": "v2",
        "template_source": "ccoe-github-actions",
        "stages_passed": ["fmt","validate","security-scan","policy-validate","plan"],
        "environment": "${{ inputs.environment }}",
        "timestamp": "${{ github.event.head_commit.timestamp }}"
      }'
```

### Acceptance Criteria

- [ ] All four workflow files implemented and valid YAML
- [ ] Plan output uploaded as artifact with retention = 30 days
- [ ] Approval gate uses GitHub Environments (required reviewers) for `prod`
- [ ] Security scan step fails the workflow on HIGH/CRITICAL findings
- [ ] Policy validate step respects `policy_enforcement` input
- [ ] Compliance metadata POST fires on every run (fail-safe: pipeline continues even if POST fails)
- [ ] Terraform state backend config injected via secrets (not hardcoded)

---

## Story 3.2 — GitLab CI Shared Includes

### File Structure

```
pipeline-templates/gitlab-ci/
├── terraform-stages.yml      # Hidden jobs: .fmt, .validate, .security, .policy, .plan, .apply, .drift
├── terraform-pipeline.yml    # Extends all stages into a full pipeline
└── README.md
```

### Usage by Subsidiaries

```yaml
# Subsidiary repo: .gitlab-ci.yml
include:
  - project: ccoe/ccoe-iac-governance
    ref: v2
    file: pipeline-templates/gitlab-ci/terraform-pipeline.yml

variables:
  TF_WORKING_DIR: ./infra
  SUBSIDIARY_ID: acme
  ENVIRONMENT: prod
```

### Adoption Detection Signal

GitLab pipelines using the include directive are detectable by the compliance platform
scanner. Implement the `include` block as the **only** supported extension mechanism —
do not support copy-paste templates.

### Acceptance Criteria

- [ ] Hidden jobs (`.fmt`, `.validate`, etc.) can be individually overridden by subsidiaries
- [ ] `COMPLIANCE_API_URL` and `COMPLIANCE_API_TOKEN` consumed from CI/CD variables
- [ ] Manual approval job uses `when: manual` with `allow_failure: false` for prod
- [ ] Drift scan implemented as a `schedule`-only job
- [ ] README includes migration guide from legacy GitLab pipelines

---

## Story 3.3 — Jenkins Shared Library

### File Structure

```
pipeline-templates/jenkins/
├── vars/
│   ├── terraformPipeline.groovy    # Main entry point: single-call full pipeline
│   ├── terraformPlan.groovy        # Callable step: plan stages only
│   ├── terraformApply.groovy       # Callable step: apply
│   └── emitComplianceMetadata.groovy
├── src/
│   └── com/ccoe/terraform/
│       ├── PolicyValidator.groovy
│       └── ComplianceReporter.groovy
├── resources/
│   └── scripts/
│       └── drift-check.sh
└── README.md
```

### Usage by Subsidiaries

```groovy
// Subsidiary Jenkinsfile
@Library('ccoe-pipeline@v2') _

terraformPipeline(
  workingDir: './infra',
  environment: 'prod',
  subsidiaryId: 'acme',
  policyEnforcement: 'enforce'
)
```

### Adoption Detection Signal

The `@Library('ccoe-pipeline')` annotation is the detection signal for the compliance
platform. Document this prominently — subsidiaries must not rename the library reference.

### Acceptance Criteria

- [ ] `terraformPipeline` implements all 8 stages
- [ ] Parallel execution of `fmt` + `validate` where safe
- [ ] `emitComplianceMetadata` step runs as `post { always { } }` block
- [ ] Library published to Jenkins internal library server with semantic versioning
- [ ] README includes Jenkinsfile examples for dev, staging, and prod environments

---

## Adoption Scoring CI Check

File: `.github/workflows/pipeline-templates-ci.yml`

Jobs:
1. `validate-github` — lint all `pipeline-templates/github-actions/*.yml` with `actionlint`
2. `validate-gitlab` — lint with `gitlab-ci-lint` API
3. `validate-jenkins` — run Groovy syntax check
4. `integration-test` — dry-run each template against a stub Terraform config

---

## Multi-Cloud Pipeline Notes (added: multi-cloud update)

Pipeline templates must support all three cloud providers. Provider is passed as an
input and used to configure authentication and backend.

### Provider Input

Add to all template inputs:

```yaml
cloud_provider:
  description: "Cloud provider: azure | aws | gcp"
  required: true
  type: string
```

### Authentication Per Provider

| Provider | GitHub Actions | GitLab CI | Jenkins |
|----------|---------------|-----------|---------|
| Azure | `azure/login` OIDC | `AZURE_CLIENT_ID` / OIDC | Azure SP credentials |
| AWS | `aws-actions/configure-aws-credentials` OIDC | `AWS_ROLE_ARN` / OIDC | AWS credentials plugin |
| GCP | `google-github-actions/auth` Workload Identity | `GCP_SERVICE_ACCOUNT` / OIDC | GCP credentials plugin |

OIDC / Workload Identity Federation is preferred over static credentials for all three.

### Backend Config Per Provider

```
Azure  → azurerm backend (Storage Account)
AWS    → s3 backend (S3 + DynamoDB lock)
GCP    → gcs backend (Cloud Storage)
```

Backend config is injected by the pipeline template via `-backend-config` flags — it
must NOT be hardcoded in subsidiary Terraform code.

### Compliance Metadata: Add `cloud_provider` Field

The metadata POST to the compliance platform must include `cloud_provider`:

```json
{
  "subsidiary": "acme",
  "cloud_provider": "azure",
  "repo": "...",
  ...
}
```

Update `compliance-platform/schema/pipeline-run.schema.json` to require this field
(see `compliance-platform/CLAUDE.md`).
