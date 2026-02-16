Complete Step-by-Step Guide: Part 3 (UPDATED) - Separated Dev Environment Structure
🎯 NEW STRUCTURE - Better & More Scalable!
Instead of one big main.tf, we'll separate each resource type into its own folder:
environments/dev/
├── backend.tf                      # Shared backend config
├── catalog/                        # Catalog resources
│   ├── main.tf
│   ├── variables.tf
│   └── outputs.tf
├── storage-container/              # All 3 containers
│   ├── main.tf
│   ├── variables.tf
│   └── outputs.tf
├── storage-credential/             # Credential
│   ├── main.tf
│   ├── variables.tf
│   └── outputs.tf
├── external-location/              # All 3 locations
│   ├── main.tf
│   ├── variables.tf
│   └── outputs.tf
├── schema/                         # All 3 schemas
│   ├── main.tf
│   ├── variables.tf
│   └── outputs.tf
├── volume/                         # All 4 volumes
│   ├── main.tf
│   ├── variables.tf
│   └── outputs.tf
└── grants/                         # All grants
    ├── main.tf
    ├── variables.tf
    └── outputs.tf
Benefits:
✅ Want to add more schemas? Just edit schema/main.tf
✅ Want to deploy only volumes? Just run terraform in volume/ folder
✅ Clear organization
✅ Easy to manage and scale
PHASE 3: CREATE DEV ENVIRONMENT (SEPARATED STRUCTURE)
Step 1: Create Folder Structure
In VS Code, inside environments/dev/, create these folders:
catalog
storage-container
storage-credential
external-location
schema
volume
grants
You should have:
environments/dev/
├── catalog/
├── storage-container/
├── storage-credential/
├── external-location/
├── schema/
├── volume/
└── grants/
FILE 1: SHARED BACKEND
File: environments/dev/backend.tf
# Shared Backend Configuration for Dev Environment
# Used by all resource folders

terraform {
  backend "azurerm" {
    # Configuration provided via GitHub Actions
    # resource_group_name  = "rg-terraform-state"
    # storage_account_name = "sttfstatedatabricks"
    # container_name       = "tfstate"
    # key                  = "dev/<resource-type>.tfstate"
  }
}
RESOURCE 1: CATALOG
File: environments/dev/catalog/main.tf
# Catalog Configuration
# Creates Unity Catalog (if needed)

terraform {
  required_version = ">= 1.5.0"

  required_providers {
    databricks = {
      source  = "databricks/databricks"
      version = "~> 1.35.0"
    }
  }
  
  backend "azurerm" {
    key = "dev/catalog.tfstate"
  }
}

provider "databricks" {
  # Authentication via environment variables
}

# Uncomment if you need to create the catalog
# module "catalog" {
#   source = "../../../modules/catalog"
# 
#   catalog_name   = var.catalog_name
#   comment        = "Unity Catalog for ${var.environment} environment"
#   isolation_mode = "OPEN"
#   
#   properties = {
#     environment = var.environment
#     managed_by  = "terraform"
#   }
# }

# If catalog already exists, just use a data source
data "databricks_catalog" "existing" {
  name = var.catalog_name
}
File: environments/dev/catalog/variables.tf
variable "environment" {
  description = "Environment name"
  type        = string
  default     = "dev"
}

variable "catalog_name" {
  description = "Unity Catalog name"
  type        = string
  default     = "dev_terraform_catalog"
}
File: environments/dev/catalog/outputs.tf
output "catalog_name" {
  description = "Name of the catalog"
  value       = var.catalog_name
}

output "catalog_id" {
  description = "ID of the catalog"
  value       = data.databricks_catalog.existing.id
}
RESOURCE 2: STORAGE CONTAINERS
File: environments/dev/storage-container/main.tf
# Storage Containers Configuration
# Creates bronze, silver, gold containers

terraform {
  required_version = ">= 1.5.0"

  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.85.0"
    }
  }
  
  backend "azurerm" {
    key = "dev/storage-container.tfstate"
  }
}

provider "azurerm" {
  features {}
}

# Bronze Container
module "bronze_container" {
  source = "../../../modules/storage-container"

  container_name       = "bronze"
  storage_account_name = var.storage_account_name

  metadata = {
    layer       = "bronze"
    environment = var.environment
  }
}

# Silver Container
module "silver_container" {
  source = "../../../modules/storage-container"

  container_name       = "silver"
  storage_account_name = var.storage_account_name

  metadata = {
    layer       = "silver"
    environment = var.environment
  }
}

# Gold Container
module "gold_container" {
  source = "../../../modules/storage-container"

  container_name       = "gold"
  storage_account_name = var.storage_account_name

  metadata = {
    layer       = "gold"
    environment = var.environment
  }
}
File: environments/dev/storage-container/variables.tf
variable "environment" {
  description = "Environment name"
  type        = string
  default     = "dev"
}

variable "storage_account_name" {
  description = "Azure storage account name"
  type        = string
}

variable "resource_group_name" {
  description = "Azure resource group name"
  type        = string
}
File: environments/dev/storage-container/outputs.tf
output "bronze_container" {
  description = "Bronze container details"
  value = {
    name = module.bronze_container.container_name
    url  = module.bronze_container.container_url
  }
}

output "silver_container" {
  description = "Silver container details"
  value = {
    name = module.silver_container.container_name
    url  = module.silver_container.container_url
  }
}

output "gold_container" {
  description = "Gold container details"
  value = {
    name = module.gold_container.container_name
    url  = module.gold_container.container_url
  }
}
RESOURCE 3: STORAGE CREDENTIAL
File: environments/dev/storage-credential/main.tf
# Storage Credential Configuration

terraform {
  required_version = ">= 1.5.0"

  required_providers {
    databricks = {
      source  = "databricks/databricks"
      version = "~> 1.35.0"
    }
  }
  
  backend "azurerm" {
    key = "dev/storage-credential.tfstate"
  }
}

provider "databricks" {}

module "storage_credential" {
  source = "../../../modules/storage-credential"

  credential_name     = "${var.environment}_storage_credential"
  access_connector_id = var.access_connector_id
  comment             = "Storage credential for ${var.environment} environment - managed by Terraform"
}
File: environments/dev/storage-credential/variables.tf
variable "environment" {
  description = "Environment name"
  type        = string
  default     = "dev"
}

variable "access_connector_id" {
  description = "Azure Databricks Access Connector resource ID"
  type        = string
}
File: environments/dev/storage-credential/outputs.tf
output "credential_name" {
  description = "Storage credential name"
  value       = module.storage_credential.credential_name
}

output "credential_id" {
  description = "Storage credential ID"
  value       = module.storage_credential.credential_id
}
RESOURCE 4: EXTERNAL LOCATIONS
File: environments/dev/external-location/main.tf
# External Locations Configuration
# Creates bronze, silver, gold external locations

terraform {
  required_version = ">= 1.5.0"

  required_providers {
    databricks = {
      source  = "databricks/databricks"
      version = "~> 1.35.0"
    }
  }
  
  backend "azurerm" {
    key = "dev/external-location.tfstate"
  }
}

provider "databricks" {}

# Bronze External Location
module "bronze_location" {
  source = "../../../modules/external-location"

  location_name   = "${var.environment}_bronze"
  url             = var.bronze_container_url
  credential_name = var.credential_name
  comment         = "Bronze layer - raw data ingestion"
}

# Silver External Location
module "silver_location" {
  source = "../../../modules/external-location"

  location_name   = "${var.environment}_silver"
  url             = var.silver_container_url
  credential_name = var.credential_name
  comment         = "Silver layer - cleaned and validated data"
}

# Gold External Location
module "gold_location" {
  source = "../../../modules/external-location"

  location_name   = "${var.environment}_gold"
  url             = var.gold_container_url
  credential_name = var.credential_name
  comment         = "Gold layer - curated business data"
}
File: environments/dev/external-location/variables.tf
variable "environment" {
  description = "Environment name"
  type        = string
  default     = "dev"
}

variable "credential_name" {
  description = "Storage credential name to use"
  type        = string
}

variable "bronze_container_url" {
  description = "Bronze container URL"
  type        = string
}

variable "silver_container_url" {
  description = "Silver container URL"
  type        = string
}

variable "gold_container_url" {
  description = "Gold container URL"
  type        = string
}
File: environments/dev/external-location/outputs.tf
output "bronze_location" {
  description = "Bronze external location details"
  value = {
    name = module.bronze_location.location_name
    url  = module.bronze_location.location_url
    id   = module.bronze_location.location_id
  }
}

output "silver_location" {
  description = "Silver external location details"
  value = {
    name = module.silver_location.location_name
    url  = module.silver_location.location_url
    id   = module.silver_location.location_id
  }
}

output "gold_location" {
  description = "Gold external location details"
  value = {
    name = module.gold_location.location_name
    url  = module.gold_location.location_url
    id   = module.gold_location.location_id
  }
}
RESOURCE 5: SCHEMAS
File: environments/dev/schema/main.tf
# Schemas Configuration
# Creates bronze, silver, gold schemas

terraform {
  required_version = ">= 1.5.0"

  required_providers {
    databricks = {
      source  = "databricks/databricks"
      version = "~> 1.35.0"
    }
  }
  
  backend "azurerm" {
    key = "dev/schema.tfstate"
  }
}

provider "databricks" {}

# Bronze Schema
module "bronze_schema" {
  source = "../../../modules/schema"

  catalog_name = var.catalog_name
  schema_name  = "bronze"
  comment      = "Bronze layer - raw ingested data from source systems"

  properties = {
    layer       = "bronze"
    managed_by  = "terraform"
    environment = var.environment
  }
}

# Silver Schema
module "silver_schema" {
  source = "../../../modules/schema"

  catalog_name = var.catalog_name
  schema_name  = "silver"
  comment      = "Silver layer - cleaned, validated, and standardized data"

  properties = {
    layer       = "silver"
    managed_by  = "terraform"
    environment = var.environment
  }
}

# Gold Schema
module "gold_schema" {
  source = "../../../modules/schema"

  catalog_name = var.catalog_name
  schema_name  = "gold"
  comment      = "Gold layer - aggregated business-level data and analytics"

  properties = {
    layer       = "gold"
    managed_by  = "terraform"
    environment = var.environment
  }
}

# Add more schemas here as needed
# module "analytics_schema" {
#   source = "../../../modules/schema"
#   
#   catalog_name = var.catalog_name
#   schema_name  = "analytics"
#   comment      = "Analytics data"
# }
File: environments/dev/schema/variables.tf
variable "environment" {
  description = "Environment name"
  type        = string
  default     = "dev"
}

variable "catalog_name" {
  description = "Unity Catalog name"
  type        = string
  default     = "dev_terraform_catalog"
}
File: environments/dev/schema/outputs.tf
output "bronze_schema" {
  description = "Bronze schema details"
  value = {
    name      = module.bronze_schema.schema_name
    full_name = module.bronze_schema.schema_full_name
    id        = module.bronze_schema.schema_id
  }
}

output "silver_schema" {
  description = "Silver schema details"
  value = {
    name      = module.silver_schema.schema_name
    full_name = module.silver_schema.schema_full_name
    id        = module.silver_schema.schema_id
  }
}

output "gold_schema" {
  description = "Gold schema details"
  value = {
    name      = module.gold_schema.schema_name
    full_name = module.gold_schema.schema_full_name
    id        = module.gold_schema.schema_id
  }
}
RESOURCE 6: VOLUMES
File: environments/dev/volume/main.tf
# Volumes Configuration
# Creates all external volumes

terraform {
  required_version = ">= 1.5.0"

  required_providers {
    databricks = {
      source  = "databricks/databricks"
      version = "~> 1.35.0"
    }
  }
  
  backend "azurerm" {
    key = "dev/volume.tfstate"
  }
}

provider "databricks" {}

# Bronze Landing Volume
module "bronze_landing_volume" {
  source = "../../../modules/volume"

  volume_name      = "landing_zone"
  catalog_name     = var.catalog_name
  schema_name      = "bronze"
  volume_type      = "EXTERNAL"
  storage_location = "${var.bronze_location_url}landing/"
  comment          = "Landing zone for incoming raw files"
}

# Bronze Archive Volume
module "bronze_archive_volume" {
  source = "../../../modules/volume"

  volume_name      = "archive"
  catalog_name     = var.catalog_name
  schema_name      = "bronze"
  volume_type      = "EXTERNAL"
  storage_location = "${var.bronze_location_url}archive/"
  comment          = "Archive for processed raw files"
}

# Silver Processed Volume
module "silver_processed_volume" {
  source = "../../../modules/volume"

  volume_name      = "processed"
  catalog_name     = var.catalog_name
  schema_name      = "silver"
  volume_type      = "EXTERNAL"
  storage_location = "${var.silver_location_url}processed/"
  comment          = "Processed and cleaned data files"
}

# Gold Exports Volume
module "gold_exports_volume" {
  source = "../../../modules/volume"

  volume_name      = "exports"
  catalog_name     = var.catalog_name
  schema_name      = "gold"
  volume_type      = "EXTERNAL"
  storage_location = "${var.gold_location_url}exports/"
  comment          = "Curated data exports for business consumption"
}

# Add more volumes here as needed
# module "bronze_quarantine_volume" {
#   source = "../../../modules/volume"
#   
#   volume_name      = "quarantine"
#   catalog_name     = var.catalog_name
#   schema_name      = "bronze"
#   volume_type      = "EXTERNAL"
#   storage_location = "${var.bronze_location_url}quarantine/"
# }
File: environments/dev/volume/variables.tf
variable "catalog_name" {
  description = "Unity Catalog name"
  type        = string
  default     = "dev_terraform_catalog"
}

variable "bronze_location_url" {
  description = "Bronze external location URL"
  type        = string
}

variable "silver_location_url" {
  description = "Silver external location URL"
  type        = string
}

variable "gold_location_url" {
  description = "Gold external location URL"
  type        = string
}
File: environments/dev/volume/outputs.tf
output "bronze_landing_volume" {
  description = "Bronze landing volume details"
  value = {
    name      = module.bronze_landing_volume.volume_name
    full_name = module.bronze_landing_volume.volume_full_name
    path      = module.bronze_landing_volume.volume_path
    id        = module.bronze_landing_volume.volume_id
  }
}

output "bronze_archive_volume" {
  description = "Bronze archive volume details"
  value = {
    name      = module.bronze_archive_volume.volume_name
    full_name = module.bronze_archive_volume.volume_full_name
    path      = module.bronze_archive_volume.volume_path
    id        = module.bronze_archive_volume.volume_id
  }
}

output "silver_processed_volume" {
  description = "Silver processed volume details"
  value = {
    name      = module.silver_processed_volume.volume_name
    full_name = module.silver_processed_volume.volume_full_name
    path      = module.silver_processed_volume.volume_path
    id        = module.silver_processed_volume.volume_id
  }
}

output "gold_exports_volume" {
  description = "Gold exports volume details"
  value = {
    name      = module.gold_exports_volume.volume_name
    full_name = module.gold_exports_volume.volume_full_name
    path      = module.gold_exports_volume.volume_path
    id        = module.gold_exports_volume.volume_id
  }
}
RESOURCE 7: GRANTS
File: environments/dev/grants/main.tf
# Grants Configuration
# Manages all permissions

terraform {
  required_version = ">= 1.5.0"

  required_providers {
    databricks = {
      source  = "databricks/databricks"
      version = "~> 1.35.0"
    }
  }
  
  backend "azurerm" {
    key = "dev/grants.tfstate"
  }
}

provider "databricks" {}

# Storage Credential Grants
module "storage_credential_grants" {
  source = "../../../modules/grants"

  storage_credential = var.storage_credential_id

  grants = [
    {
      principal  = "account users"
      privileges = ["CREATE_EXTERNAL_TABLE"]
    }
  ]
}

# External Location Grants - Bronze
module "bronze_location_grants" {
  source = "../../../modules/grants"

  external_location = var.bronze_location_id

  grants = [
    {
      principal  = "account users"
      privileges = ["CREATE_EXTERNAL_TABLE", "READ_FILES", "WRITE_FILES"]
    }
  ]
}

# External Location Grants - Silver
module "silver_location_grants" {
  source = "../../../modules/grants"

  external_location = var.silver_location_id

  grants = [
    {
      principal  = "account users"
      privileges = ["CREATE_EXTERNAL_TABLE", "READ_FILES", "WRITE_FILES"]
    }
  ]
}

# External Location Grants - Gold
module "gold_location_grants" {
  source = "../../../modules/grants"

  external_location = var.gold_location_id

  grants = [
    {
      principal  = "account users"
      privileges = ["CREATE_EXTERNAL_TABLE", "READ_FILES", "WRITE_FILES"]
    }
  ]
}

# Schema Grants - Bronze
module "bronze_schema_grants" {
  source = "../../../modules/grants"

  schema = var.bronze_schema_full_name

  grants = [
    {
      principal  = "account users"
      privileges = ["USE_SCHEMA", "SELECT", "MODIFY", "CREATE_TABLE", "CREATE_VOLUME"]
    }
  ]
}

# Schema Grants - Silver
module "silver_schema_grants" {
  source = "../../../modules/grants"

  schema = var.silver_schema_full_name

  grants = [
    {
      principal  = "account users"
      privileges = ["USE_SCHEMA", "SELECT", "MODIFY", "CREATE_TABLE", "CREATE_VOLUME"]
    }
  ]
}

# Schema Grants - Gold
module "gold_schema_grants" {
  source = "../../../modules/grants"

  schema = var.gold_schema_full_name

  grants = [
    {
      principal  = "account users"
      privileges = ["USE_SCHEMA", "SELECT", "CREATE_TABLE", "CREATE_VOLUME"]
    }
  ]
}

# Volume Grants - Bronze Landing
module "bronze_landing_volume_grants" {
  source = "../../../modules/grants"

  volume = var.bronze_landing_volume_id

  grants = [
    {
      principal  = "account users"
      privileges = ["READ_VOLUME", "WRITE_VOLUME"]
    }
  ]
}

# Volume Grants - Bronze Archive
module "bronze_archive_volume_grants" {
  source = "../../../modules/grants"

  volume = var.bronze_archive_volume_id

  grants = [
    {
      principal  = "account users"
      privileges = ["READ_VOLUME", "WRITE_VOLUME"]
    }
  ]
}

# Volume Grants - Silver Processed
module "silver_processed_volume_grants" {
  source = "../../../modules/grants"

  volume = var.silver_processed_volume_id

  grants = [
    {
      principal  = "account users"
      privileges = ["READ_VOLUME", "WRITE_VOLUME"]
    }
  ]
}

# Volume Grants - Gold Exports
module "gold_exports_volume_grants" {
  source = "../../../modules/grants"

  volume = var.gold_exports_volume_id

  grants = [
    {
      principal  = "account users"
      privileges = ["READ_VOLUME", "WRITE_VOLUME"]
    }
  ]
}
File: environments/dev/grants/variables.tf
variable "storage_credential_id" {
  description = "Storage credential ID"
  type        = string
}

variable "bronze_location_id" {
  description = "Bronze external location ID"
  type        = string
}

variable "silver_location_id" {
  description = "Silver external location ID"
  type        = string
}

variable "gold_location_id" {
  description = "Gold external location ID"
  type        = string
}

variable "bronze_schema_full_name" {
  description = "Bronze schema full name (catalog.schema)"
  type        = string
}

variable "silver_schema_full_name" {
  description = "Silver schema full name (catalog.schema)"
  type        = string
}

variable "gold_schema_full_name" {
  description = "Gold schema full name (catalog.schema)"
  type        = string
}

variable "bronze_landing_volume_id" {
  description = "Bronze landing volume ID"
  type        = string
}

variable "bronze_archive_volume_id" {
  description = "Bronze archive volume ID"
  type        = string
}

variable "silver_processed_volume_id" {
  description = "Silver processed volume ID"
  type        = string
}

variable "gold_exports_volume_id" {
  description = "Gold exports volume ID"
  type        = string
}
File: environments/dev/grants/outputs.tf
output "grants_summary" {
  description = "Summary of applied grants"
  value = {
    storage_credential_grants = 1
    external_location_grants  = 3
    schema_grants            = 3
    volume_grants            = 4
    total                    = 11
  }
}
📝 HOW TO DEPLOY (With Separated Structure)
Option 1: Deploy Each Resource Separately
# 1. Deploy storage containers first
cd environments/dev/storage-container
terraform init -backend-config="..."
terraform apply

# 2. Deploy storage credential
cd ../storage-credential
terraform init -backend-config="..."
terraform apply

# 3. Deploy external locations (needs containers + credential)
cd ../external-location
terraform apply -var="bronze_container_url=..." -var="credential_name=..."

# And so on...
Option 2: Deploy All at Once (Via GitHub Actions)
Update workflows to deploy each folder sequentially:
# In terraform-apply-dev.yml

- name: Deploy Storage Containers
  working-directory: terraform/environments/dev/storage-container
  run: terraform apply

- name: Deploy Storage Credential
  working-directory: terraform/environments/dev/storage-credential
  run: terraform apply

- name: Deploy External Locations
  working-directory: terraform/environments/dev/external-location
  run: terraform apply
  
# etc...
✅ BENEFITS OF THIS STRUCTURE
When You Want to Add More Schemas:
# Just edit ONE file:
cd environments/dev/schema
# Edit main.tf, add:
module "analytics_schema" {
  source = "../../../modules/schema"
  catalog_name = var.catalog_name
  schema_name  = "analytics"
}
When You Want to Add More Volumes:
# Just edit ONE file:
cd environments/dev/volume
# Edit main.tf, add:
module "new_volume" {
  source = "../../../modules/volume"
  volume_name = "my_new_volume"
  # ...
}
Independent Deployment:
# Only want to update volumes? 
cd environments/dev/volume
terraform apply

# Only storage containers?
cd environments/dev/storage-container
terraform apply
🎯 SUMMARY
Old Structure (Part 3 Original):
environments/dev/
├── main.tf (200+ lines, everything mixed)
├── variables.tf
└── outputs.tf
New Structure (Part 3 Updated):
environments/dev/
├── backend.tf (shared)
├── catalog/ (3 files)
├── storage-container/ (3 files)
├── storage-credential/ (3 files)
├── external-location/ (3 files)
├── schema/ (3 files)
├── volume/ (3 files)
└── grants/ (3 files)
Total: 22 files (instead of 4), but much better organized!
Ready to create this improved structure? 🚀