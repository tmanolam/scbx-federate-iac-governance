# CLAUDE.md — Story 1.1: Base Terraform Modules (Multi-Cloud)

Parent epic: `modules/CLAUDE.md`

## ⚠️ Read First

```
standards/NAMING.md     — name pattern, abbreviation tables for all 3 clouds
standards/TAGGING.md    — mandatory tags, GCP labels caveat, AWS Name tag requirement
```

## Build Order (respect provider dependencies)

Build each module across all 3 providers before moving to the next:

1. `vnet` — foundation; all other modules depend on it
2. `keyvault` — standalone
3. `storage` — standalone
4. `database` — depends on vnet (private endpoint)
5. `vm` — depends on vnet

For each module, implement: `azure/` → `aws/` → `gcp/`

## Per-Provider Checklists

### vm

| File | Azure | AWS | GCP |
|------|-------|-----|-----|
| Provider resource | `azurerm_linux_virtual_machine` | `aws_instance` | `google_compute_instance` |
| Key security default | `disable_password_authentication = true` | `ebs_optimized = true` + root encrypted | `shielded_instance_config` enabled |
| Name local | `${sub}-${env}-az-vm-${name}` | `${sub}-${env}-aws-vm-${name}` | `${sub}-${env}-gcp-vm-${name}` |
| Tags | `tags = var.tags` | `tags = merge(var.tags, {Name=local.resource_name})` | `labels = local.gcp_labels` |

### vnet

| File | Azure | AWS | GCP |
|------|-------|-----|-----|
| Resources | `azurerm_virtual_network` + `azurerm_subnet` + `azurerm_network_security_group` | `aws_vpc` + `aws_subnet` + `aws_security_group` + `aws_flow_log` | `google_compute_network` + `google_compute_subnetwork` |
| Key security default | NSG attached to every subnet | `enable_dns_hostnames = true`; no IGW attached | `private_ip_google_access = true` |

### storage

| File | Azure | AWS | GCP |
|------|-------|-----|-----|
| Resources | `azurerm_storage_account` | `aws_s3_bucket` + `aws_s3_bucket_public_access_block` + `aws_s3_bucket_versioning` + `aws_s3_bucket_server_side_encryption_configuration` | `google_storage_bucket` |
| Key security default | `allow_blob_public_access = false`; `min_tls_version = "TLS1_2"` | All four public-access-block booleans = true | `uniform_bucket_level_access = true` |

### database

| File | Azure | AWS | GCP |
|------|-------|-----|-----|
| Resources | `azurerm_mssql_server` + `azurerm_mssql_database` | `aws_db_instance` (engine = postgres) + `aws_db_subnet_group` | `google_sql_database_instance` + `google_sql_user` |
| Key security default | `public_network_access_enabled = false` | `publicly_accessible = false`; `storage_encrypted = true` | `ipv4_enabled = false`; private IP via `google_compute_global_address` |

### keyvault

| File | Azure | AWS | GCP |
|------|-------|-----|-----|
| Resources | `azurerm_key_vault` | `aws_kms_key` + `aws_secretsmanager_secret` | `google_kms_key_ring` + `google_kms_crypto_key` + `google_secret_manager_secret` |
| Key security default | `purge_protection_enabled = true`; default network deny | `enable_key_rotation = true`; `deletion_window_in_days = 30` | `rotation_period` set; `prevent_destroy = true` on key ring |

## Terratest Pattern (same structure for all providers)

```go
// tests/main_test.go
func TestVnetModule_Azure(t *testing.T) {
    opts := &terraform.Options{
        TerraformDir: ".",
        Vars: map[string]interface{}{
            "name":        "test",
            "environment": "dev",
            "subsidiary":  "test",
            "location":    "eastus",
            "tags": map[string]string{
                "environment":  "dev",
                "subsidiary":   "test",
                "managed-by":   "terraform",
                "cost-center":  "CC-0000",
                "owner":        "test@test.com",
            },
        },
    }
    defer terraform.Destroy(t, opts)
    terraform.InitAndApply(t, opts)

    // Assert name conforms to NAMING.md
    name := terraform.Output(t, opts, "name")
    assert.Regexp(t, `^test-dev-az-vnet-test$`, name)

    // Assert mandatory tag present on actual cloud resource
    // ... provider-specific SDK call here
}
```
