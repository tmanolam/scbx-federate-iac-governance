# CLAUDE.md — standards/

## Purpose

This directory contains the **canonical governance standards** that Claude Code must
read before writing any code that involves resource names, tags, or provider-specific
resource configuration.

These are not guidelines — they are enforced by OPA policies in `policies/compliance/`
and validated in CI on every PR.

---

## When to Read These Files

| You are about to… | Read this file first |
|-------------------|----------------------|
| Write any `resource` or `data` block in Terraform | `NAMING.md` + `TAGGING.md` |
| Write or update a `variable "tags"` block | `TAGGING.md` |
| Write or update a `local { resource_name = … }` | `NAMING.md` |
| Write a policy in `policies/compliance/` | Both |
| Write a catalog entry `metadata.yaml` | `NAMING.md` |
| Write pipeline template that sets provider `default_tags` | `TAGGING.md` |
| Write a brownfield registry `registration.yaml` | `TAGGING.md` |

---

## Files in This Directory

| File | Contents |
|------|----------|
| `NAMING.md` | Universal `{subsidiary}-{env}-{provider}-{type}-{name}` pattern, per-cloud abbreviation tables, special cases (storage accounts, S3, GCP projects) |
| `TAGGING.md` | 5 mandatory tags, recommended tags, per-provider implementation (Azure `tags`, AWS `default_tags`, GCP `labels`), Terraform validation blocks, sandbox expiry rules |

---

## Critical Rules for Claude Code

1. **Never invent a resource abbreviation.** Always use the table in `NAMING.md`. If a
   resource type is missing from the table, add it to `NAMING.md` first, then use it.

2. **Never hardcode tag keys.** Always derive from the mandatory tag list in `TAGGING.md`.

3. **GCP uses `labels`, not `tags`.** The Terraform variable is always named `tags` for
   interface consistency, but it maps to `labels` in GCP resources. See `TAGGING.md §3`.

4. **AWS `Name` tag is mandatory.** Always set `Name = local.resource_name` in AWS
   resource `tags` blocks. See `TAGGING.md §3`.

5. **Sandbox resources need `expiry-date`.** The validation block in `TAGGING.md §5`
   must be present in any module that accepts `environment = "sandbox"`.

6. **Max 63 chars on assembled names.** Include the length validation in every module
   that constructs `local.resource_name`. See `NAMING.md §5`.
