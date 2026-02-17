terraform {
  required_version = ">= 1.5.0"

  required_providers {
    databricks = {
      source  = "databricks/databricks"
      version = "~> 1.35.0"
    }
  }
}

provider "databricks" {
  # Auth via DATABRICKS_HOST and DATABRICKS_TOKEN environment variables
}

module "bronze_schema_grants" {
  source = "../../../modules/grants"

  schema = var.bronze_schema_full_name

  grants = [
    {
      principal  = "account users"    # ← REPLACE with your group/user if needed
      privileges = ["USE_SCHEMA", "SELECT", "MODIFY", "CREATE_TABLE"]
    }
  ]
}