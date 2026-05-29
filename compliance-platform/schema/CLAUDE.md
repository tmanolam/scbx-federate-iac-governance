# CLAUDE.md — Story 4.1: Metadata Schema & API Contract

Parent epic: `compliance-platform/CLAUDE.md`

## Task

Define all data contracts before any service code is written. The schema is the
single source of truth for what pipeline templates send and what the collector accepts.

## Deliverables Checklist

- [ ] `pipeline-run.schema.json`
- [ ] `drift-event.schema.json`
- [ ] `repo-scan-result.schema.json`
- [ ] `compliance-score.schema.json`
- [ ] `openapi.yaml` (OpenAPI 3.0)
- [ ] `tests/validate-schemas.test.js` using AJV

## Compliance Score Levels

Implement as a pure function — given a pipeline run event, return a score level:

```
Input: { template_source, stages_passed[] }

Gold   → template_source IN ["ccoe-github-actions","ccoe-gitlab-ci","ccoe-jenkins"]
         AND stages_passed contains all of: fmt, validate, security-scan, policy-validate, plan, apply

Silver → template_source = "custom"
         AND stages_passed contains all of: fmt, validate, security-scan, policy-validate, plan

Bronze → template_source = "custom"
         AND stages_passed contains: fmt, validate, plan
         AND (missing security-scan OR policy-validate)

Red    → stages_passed does NOT contain security-scan OR policy-validate
```

## Key Design Decisions to Document

Add a `DECISIONS.md` in this directory covering:
- Why JSON Schema over Protobuf (answer: Git-friendly, no codegen required)
- Why `schemaVersion` field (answer: backward compat when collector ingests v1+v2)
- Why pipeline runs are append-only (answer: auditability)

---

## Multi-Cloud Schema Updates (added: multi-cloud update)

Add `cloud_provider` as a required field to `pipeline-run.schema.json`:

```json
"cloud_provider": {
  "type": "string",
  "enum": ["azure", "aws", "gcp"]
}
```

Add `cloud_provider` to `drift-event.schema.json` and `repo-scan-result.schema.json`.

The compliance score algorithm is provider-agnostic — score is based on pipeline
template adoption, not which cloud is being deployed to.
