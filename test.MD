# Databricks Storage Credential Module
# Creates a storage credential using Azure Managed Identity

terraform {
  required_providers {
    databricks = {
      source  = "databricks/databricks"
      version = "~> 1.35.0"
    }
  }
}

resource "databricks_storage_credential" "this" {
  name    = var.credential_name
  comment = var.comment
  owner   = var.owner
  
  azure_managed_identity {
    access_connector_id = var.access_connector_id
  }
  
  read_only = var.read_only
  
  force_destroy = var.force_destroy
  force_update  = var.force_update
}