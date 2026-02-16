Complete Step-by-Step Guide: Part 2 - Remaining Modules
MODULE 4: STORAGE-CREDENTIAL
File: modules/storage-credential/main.tf
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
File: modules/storage-credential/variables.tf
variable "credential_name" {
  description = "Name of the storage credential"
  type        = string
  
  validation {
    condition     = can(regex("^[a-zA-Z0-9_-]+$", var.credential_name))
    error_message = "Credential name must contain only alphanumeric characters, underscores, and hyphens."
  }
}

variable "access_connector_id" {
  description = "Azure Databricks Access Connector resource ID"
  type        = string
  
  validation {
    condition     = can(regex("^/subscriptions/.+/resourceGroups/.+/providers/Microsoft.Databricks/accessConnectors/.+$", var.access_connector_id))
    error_message = "Access connector ID must be a valid Azure resource ID."
  }
}

variable "comment" {
  description = "Comment for the storage credential"
  type        = string
  default     = "Created through Terraform"
}

variable "owner" {
  description = "Owner of the storage credential"
  type        = string
  default     = null
}

variable "read_only" {
  description = "Whether the storage credential is read-only"
  type        = bool
  default     = false
}

variable "force_destroy" {
  description = "Allow deletion even if credential is in use"
  type        = bool
  default     = false
}

variable "force_update" {
  description = "Force update even if credential is in use"
  type        = bool
  default     = false
}
File: modules/storage-credential/outputs.tf
output "credential_id" {
  description = "ID of the storage credential"
  value       = databricks_storage_credential.this.id
}

output "credential_name" {
  description = "Name of the storage credential"
  value       = databricks_storage_credential.this.name
}

output "credential" {
  description = "Full storage credential object"
  value       = databricks_storage_credential.this
  sensitive   = true
}
File: modules/storage-credential/README.md
# Storage Credential Module

Creates a Unity Catalog storage credential using Azure Managed Identity.

## Usage

```hcl
module "storage_credential" {
  source = "../../modules/storage-credential"
  
  credential_name     = "dev_storage_credential"
  access_connector_id = "/subscriptions/.../accessConnectors/dac-dev"
  comment             = "Storage credential for dev environment"
}
Inputs
credential_name - Name of the credential (required)
access_connector_id - Azure Access Connector ID (required)
comment - Description
owner - Credential owner
read_only - Read-only access (default: false)
Outputs
credential_id - Credential ID
credential_name - Credential name
---

## MODULE 5: EXTERNAL-LOCATION

### File: `modules/external-location/main.tf`

```hcl
# Databricks External Location Module
# Creates an external location pointing to cloud storage

terraform {
  required_providers {
    databricks = {
      source  = "databricks/databricks"
      version = "~> 1.35.0"
    }
  }
}

resource "databricks_external_location" "this" {
  name            = var.location_name
  url             = var.url
  credential_name = var.credential_name
  comment         = var.comment
  owner           = var.owner
  
  read_only       = var.read_only
  skip_validation = var.skip_validation
  
  force_destroy = var.force_destroy
  force_update  = var.force_update
}
File: modules/external-location/variables.tf
variable "location_name" {
  description = "Name of the external location"
  type        = string
  
  validation {
    condition     = can(regex("^[a-zA-Z0-9_-]+$", var.location_name))
    error_message = "Location name must contain only alphanumeric characters, underscores, and hyphens."
  }
}

variable "url" {
  description = "Cloud storage URL (e.g., abfss://container@storage.dfs.core.windows.net/path/)"
  type        = string
  
  validation {
    condition     = can(regex("^(abfss|s3|gs)://", var.url))
    error_message = "URL must start with abfss://, s3://, or gs://."
  }
}

variable "credential_name" {
  description = "Name of the storage credential to use"
  type        = string
}

variable "comment" {
  description = "Comment for the external location"
  type        = string
  default     = "Created through Terraform"
}

variable "owner" {
  description = "Owner of the external location"
  type        = string
  default     = null
}

variable "read_only" {
  description = "Whether the location is read-only"
  type        = bool
  default     = false
}

variable "skip_validation" {
  description = "Skip validation of the external location"
  type        = bool
  default     = false
}

variable "force_destroy" {
  description = "Allow deletion even if location is in use"
  type        = bool
  default     = false
}

variable "force_update" {
  description = "Force update even if location is in use"
  type        = bool
  default     = false
}
File: modules/external-location/outputs.tf
output "location_id" {
  description = "ID of the external location"
  value       = databricks_external_location.this.id
}

output "location_name" {
  description = "Name of the external location"
  value       = databricks_external_location.this.name
}

output "location_url" {
  description = "URL of the external location"
  value       = databricks_external_location.this.url
}

output "location" {
  description = "Full external location object"
  value       = databricks_external_location.this
}
File: modules/external-location/README.md
# External Location Module

Creates an external location in Unity Catalog.

## Usage

```hcl
module "bronze_location" {
  source = "../../modules/external-location"
  
  location_name   = "dev_bronze"
  url             = "abfss://bronze@storage.dfs.core.windows.net/"
  credential_name = "dev_storage_credential"
  comment         = "Bronze layer external location"
}
Inputs
location_name - Location name (required)
url - Storage URL (required)
credential_name - Storage credential name (required)
comment - Description
skip_validation - Skip validation (default: false)
Outputs
location_id - Location ID
location_name - Location name
location_url - Storage URL
---

## MODULE 6: STORAGE-CONTAINER

### File: `modules/storage-container/main.tf`

```hcl
# Azure Storage Container Module
# Creates a storage container in an existing storage account

terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.85.0"
    }
  }
}

resource "azurerm_storage_container" "this" {
  name                  = var.container_name
  storage_account_name  = var.storage_account_name
  container_access_type = var.container_access_type
  
  metadata = var.metadata
}
File: modules/storage-container/variables.tf
variable "container_name" {
  description = "Name of the storage container"
  type        = string
  
  validation {
    condition     = can(regex("^[a-z0-9-]+$", var.container_name)) && length(var.container_name) >= 3 && length(var.container_name) <= 63
    error_message = "Container name must be 3-63 characters, lowercase letters, numbers, and hyphens only."
  }
}

variable "storage_account_name" {
  description = "Name of the parent storage account"
  type        = string
}

variable "container_access_type" {
  description = "Access level for the container"
  type        = string
  default     = "private"
  
  validation {
    condition     = contains(["private", "blob", "container"], var.container_access_type)
    error_message = "Access type must be private, blob, or container."
  }
}

variable "metadata" {
  description = "Metadata for the container"
  type        = map(string)
  default     = {}
}
File: modules/storage-container/outputs.tf
output "container_name" {
  description = "Name of the created container"
  value       = azurerm_storage_container.this.name
}

output "container_id" {
  description = "ID of the created container"
  value       = azurerm_storage_container.this.id
}

output "container_url" {
  description = "ABFSS URL of the container"
  value       = "abfss://${azurerm_storage_container.this.name}@${var.storage_account_name}.dfs.core.windows.net/"
}

output "container" {
  description = "Full container object"
  value       = azurerm_storage_container.this
}
File: modules/storage-container/README.md
# Storage Container Module

Creates an Azure Storage container.

## Usage

```hcl
module "bronze_container" {
  source = "../../modules/storage-container"
  
  container_name       = "bronze"
  storage_account_name = "stdevdatalake001"
  
  metadata = {
    layer = "bronze"
    environment = "dev"
  }
}
Inputs
container_name - Container name (required, 3-63 chars, lowercase)
storage_account_name - Storage account name (required)
container_access_type - Access level (default: private)
metadata - Container metadata
Outputs
container_name - Container name
container_url - ABFSS URL
---

## MODULE 7: GRANTS

### File: `modules/grants/main.tf`

```hcl
# Databricks Grants Module
# Manages permissions on Unity Catalog objects

terraform {
  required_providers {
    databricks = {
      source  = "databricks/databricks"
      version = "~> 1.35.0"
    }
  }
}

# Grants can be applied to different securable types
# This module supports: catalog, schema, table, volume, storage_credential, external_location

# Catalog grants
resource "databricks_grants" "catalog" {
  count   = var.catalog != null ? 1 : 0
  catalog = var.catalog
  
  dynamic "grant" {
    for_each = var.grants
    content {
      principal  = grant.value.principal
      privileges = grant.value.privileges
    }
  }
}

# Schema grants
resource "databricks_grants" "schema" {
  count  = var.schema != null ? 1 : 0
  schema = var.schema
  
  dynamic "grant" {
    for_each = var.grants
    content {
      principal  = grant.value.principal
      privileges = grant.value.privileges
    }
  }
}

# Volume grants
resource "databricks_grants" "volume" {
  count  = var.volume != null ? 1 : 0
  volume = var.volume
  
  dynamic "grant" {
    for_each = var.grants
    content {
      principal  = grant.value.principal
      privileges = grant.value.privileges
    }
  }
}

# Storage credential grants
resource "databricks_grants" "storage_credential" {
  count              = var.storage_credential != null ? 1 : 0
  storage_credential = var.storage_credential
  
  dynamic "grant" {
    for_each = var.grants
    content {
      principal  = grant.value.principal
      privileges = grant.value.privileges
    }
  }
}

# External location grants
resource "databricks_grants" "external_location" {
  count             = var.external_location != null ? 1 : 0
  external_location = var.external_location
  
  dynamic "grant" {
    for_each = var.grants
    content {
      principal  = grant.value.principal
      privileges = grant.value.privileges
    }
  }
}
File: modules/grants/variables.tf
variable "catalog" {
  description = "Catalog name to apply grants to"
  type        = string
  default     = null
}

variable "schema" {
  description = "Schema full name (catalog.schema) to apply grants to"
  type        = string
  default     = null
}

variable "volume" {
  description = "Volume ID to apply grants to"
  type        = string
  default     = null
}

variable "storage_credential" {
  description = "Storage credential ID to apply grants to"
  type        = string
  default     = null
}

variable "external_location" {
  description = "External location ID to apply grants to"
  type        = string
  default     = null
}

variable "grants" {
  description = "List of grants to apply"
  type = list(object({
    principal  = string
    privileges = list(string)
  }))
  
  validation {
    condition     = length(var.grants) > 0
    error_message = "At least one grant must be specified."
  }
}
File: modules/grants/outputs.tf
output "grants_applied" {
  description = "Number of grants applied"
  value       = length(var.grants)
}

output "securable_type" {
  description = "Type of securable object"
  value = (
    var.catalog != null ? "catalog" :
    var.schema != null ? "schema" :
    var.volume != null ? "volume" :
    var.storage_credential != null ? "storage_credential" :
    var.external_location != null ? "external_location" :
    "unknown"
  )
}
File: modules/grants/README.md
# Grants Module

Manages permissions on Unity Catalog objects.

## Usage

```hcl
# Grant on schema
module "bronze_schema_grants" {
  source = "../../modules/grants"
  
  schema = "dev_terraform_catalog.bronze"
  
  grants = [
    {
      principal  = "account users"
      privileges = ["USE_SCHEMA", "SELECT", "MODIFY"]
    },
    {
      principal  = "data_engineers"
      privileges = ["ALL_PRIVILEGES"]
    }
  ]
}

# Grant on external location
module "bronze_location_grants" {
  source = "../../modules/grants"
  
  external_location = databricks_external_location.bronze.id
  
  grants = [
    {
      principal  = "account users"
      privileges = ["CREATE_EXTERNAL_TABLE", "READ_FILES"]
    }
  ]
}
Inputs
ONE of: catalog, schema, volume, storage_credential, external_location
grants - List of {principal, privileges} (required)
Outputs
grants_applied - Count of grants
securable_type - Type of object
---

## MODULE 8: METASTORE-ASSIGNMENT (Optional)

### File: `modules/metastore-assignment/main.tf`

```hcl
# Databricks Metastore Assignment Module
# Assigns a metastore to a workspace
# NOTE: Requires account-level permissions

terraform {
  required_providers {
    databricks = {
      source  = "databricks/databricks"
      version = "~> 1.35.0"
    }
  }
}

resource "databricks_metastore_assignment" "this" {
  metastore_id         = var.metastore_id
  workspace_id         = var.workspace_id
  default_catalog_name = var.default_catalog_name
}
File: modules/metastore-assignment/variables.tf
variable "metastore_id" {
  description = "ID of the metastore to assign"
  type        = string
}

variable "workspace_id" {
  description = "ID of the workspace to assign to"
  type        = string
}

variable "default_catalog_name" {
  description = "Default catalog name for the workspace"
  type        = string
  default     = "main"
}
File: modules/metastore-assignment/outputs.tf
output "metastore_id" {
  description = "ID of the assigned metastore"
  value       = databricks_metastore_assignment.this.metastore_id
}

output "workspace_id" {
  description = "ID of the workspace"
  value       = databricks_metastore_assignment.this.workspace_id
}
File: modules/metastore-assignment/README.md
# Metastore Assignment Module

Assigns a metastore to a workspace.

**NOTE**: Requires Databricks Account Admin permissions.

## Usage

```hcl
module "metastore_assignment" {
  source = "../../modules/metastore-assignment"
  
  metastore_id         = "xxxxx-xxxxx-xxxxx"
  workspace_id         = "12345"
  default_catalog_name = "main"
}
Inputs
metastore_id - Metastore ID (required)
workspace_id - Workspace ID (required)
default_catalog_name - Default catalog (default: "main")
---

## ✅ MODULES COMPLETE!

You now have **8 complete modules** (32 files)!

Next: Continue to Part 3 for the Dev Environment configuration...