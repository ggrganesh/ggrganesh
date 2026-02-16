Complete Step-by-Step Guide: Part 3 - Dev Environment & Final Steps
PHASE 3: CREATE DEV ENVIRONMENT
Now we'll create the dev environment that CALLS all these modules!
DEV ENVIRONMENT
File: environments/dev/backend.tf
# Terraform Backend Configuration
# Backend details provided via GitHub Actions

terraform {
  backend "azurerm" {
    # Configuration provided via -backend-config flags during terraform init
    # resource_group_name  = provided by workflow
    # storage_account_name = provided by workflow
    # container_name       = provided by workflow
    # key                  = provided by workflow
  }
}
File: environments/dev/variables.tf
variable "environment" {
  description = "Environment name"
  type        = string
  default     = "dev"
}

variable "resource_group_name" {
  description = "Azure resource group name"
  type        = string
}

variable "storage_account_name" {
  description = "Azure storage account name"
  type        = string
}

variable "access_connector_id" {
  description = "Azure Databricks Access Connector resource ID"
  type        = string
}

variable "catalog_name" {
  description = "Unity Catalog name"
  type        = string
  default     = "dev_terraform_catalog"
}

variable "tags" {
  description = "Tags to apply to resources"
  type        = map(string)
  default = {
    Environment = "dev"
    ManagedBy   = "Terraform"
    Project     = "Databricks-Unity-Catalog"
  }
}
File: environments/dev/main.tf
# Dev Environment - Main Configuration
# This file orchestrates all modules

terraform {
  required_version = ">= 1.5.0"

  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.85.0"
    }
    databricks = {
      source  = "databricks/databricks"
      version = "~> 1.35.0"
    }
  }
}

provider "azurerm" {
  features {}
}

provider "databricks" {
  # Authentication via environment variables from GitHub Actions
}

# ============================================
# DATA SOURCES
# ============================================

data "azurerm_resource_group" "this" {
  name = var.resource_group_name
}

# ============================================
# STEP 1: CREATE STORAGE CONTAINERS
# ============================================

module "bronze_container" {
  source = "../../modules/storage-container"

  container_name       = "bronze"
  storage_account_name = var.storage_account_name

  metadata = merge(var.tags, {
    layer = "bronze"
  })
}

module "silver_container" {
  source = "../../modules/storage-container"

  container_name       = "silver"
  storage_account_name = var.storage_account_name

  metadata = merge(var.tags, {
    layer = "silver"
  })
}

module "gold_container" {
  source = "../../modules/storage-container"

  container_name       = "gold"
  storage_account_name = var.storage_account_name

  metadata = merge(var.tags, {
    layer = "gold"
  })
}

# ============================================
# STEP 2: CREATE STORAGE CREDENTIAL
# ============================================

module "storage_credential" {
  source = "../../modules/storage-credential"

  credential_name     = "${var.environment}_storage_credential"
  access_connector_id = var.access_connector_id
  comment             = "Storage credential for ${var.environment} environment - managed by Terraform"

  depends_on = [
    module.bronze_container,
    module.silver_container,
    module.gold_container
  ]
}

# ============================================
# STEP 3: CREATE EXTERNAL LOCATIONS
# ============================================

module "bronze_location" {
  source = "../../modules/external-location"

  location_name   = "${var.environment}_bronze"
  url             = module.bronze_container.container_url
  credential_name = module.storage_credential.credential_name
  comment         = "Bronze layer - raw data ingestion"

  depends_on = [module.storage_credential]
}

module "silver_location" {
  source = "../../modules/external-location"

  location_name   = "${var.environment}_silver"
  url             = module.silver_container.container_url
  credential_name = module.storage_credential.credential_name
  comment         = "Silver layer - cleaned and validated data"

  depends_on = [module.storage_credential]
}

module "gold_location" {
  source = "../../modules/external-location"

  location_name   = "${var.environment}_gold"
  url             = module.gold_container.container_url
  credential_name = module.storage_credential.credential_name
  comment         = "Gold layer - curated business data"

  depends_on = [module.storage_credential]
}

# ============================================
# STEP 4: CREATE CATALOG (if needed)
# ============================================

# Uncomment if you need to create the catalog
# module "catalog" {
#   source = "../../modules/catalog"
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

# ============================================
# STEP 5: CREATE SCHEMAS
# ============================================

module "bronze_schema" {
  source = "../../modules/schema"

  catalog_name = var.catalog_name
  schema_name  = "bronze"
  comment      = "Bronze layer - raw ingested data from source systems"

  properties = {
    layer       = "bronze"
    managed_by  = "terraform"
    environment = var.environment
  }

  # depends_on = [module.catalog]  # If creating catalog
}

module "silver_schema" {
  source = "../../modules/schema"

  catalog_name = var.catalog_name
  schema_name  = "silver"
  comment      = "Silver layer - cleaned, validated, and standardized data"

  properties = {
    layer       = "silver"
    managed_by  = "terraform"
    environment = var.environment
  }

  # depends_on = [module.catalog]
}

module "gold_schema" {
  source = "../../modules/schema"

  catalog_name = var.catalog_name
  schema_name  = "gold"
  comment      = "Gold layer - aggregated business-level data and analytics"

  properties = {
    layer       = "gold"
    managed_by  = "terraform"
    environment = var.environment
  }

  # depends_on = [module.catalog]
}

# ============================================
# STEP 6: CREATE VOLUMES
# ============================================

module "bronze_landing_volume" {
  source = "../../modules/volume"

  volume_name      = "landing_zone"
  catalog_name     = var.catalog_name
  schema_name      = module.bronze_schema.schema_name
  volume_type      = "EXTERNAL"
  storage_location = "${module.bronze_location.location_url}landing/"
  comment          = "Landing zone for incoming raw files"

  depends_on = [module.bronze_location, module.bronze_schema]
}

module "bronze_archive_volume" {
  source = "../../modules/volume"

  volume_name      = "archive"
  catalog_name     = var.catalog_name
  schema_name      = module.bronze_schema.schema_name
  volume_type      = "EXTERNAL"
  storage_location = "${module.bronze_location.location_url}archive/"
  comment          = "Archive for processed raw files"

  depends_on = [module.bronze_location, module.bronze_schema]
}

module "silver_processed_volume" {
  source = "../../modules/volume"

  volume_name      = "processed"
  catalog_name     = var.catalog_name
  schema_name      = module.silver_schema.schema_name
  volume_type      = "EXTERNAL"
  storage_location = "${module.silver_location.location_url}processed/"
  comment          = "Processed and cleaned data files"

  depends_on = [module.silver_location, module.silver_schema]
}

module "gold_exports_volume" {
  source = "../../modules/volume"

  volume_name      = "exports"
  catalog_name     = var.catalog_name
  schema_name      = module.gold_schema.schema_name
  volume_type      = "EXTERNAL"
  storage_location = "${module.gold_location.location_url}exports/"
  comment          = "Curated data exports for business consumption"

  depends_on = [module.gold_location, module.gold_schema]
}

# ============================================
# STEP 7: CREATE GRANTS
# ============================================

# Storage Credential Grants
module "storage_credential_grants" {
  source = "../../modules/grants"

  storage_credential = module.storage_credential.credential_id

  grants = [
    {
      principal  = "account users"
      privileges = ["CREATE_EXTERNAL_TABLE"]
    }
  ]

  depends_on = [module.storage_credential]
}

# External Location Grants
module "bronze_location_grants" {
  source = "../../modules/grants"

  external_location = module.bronze_location.location_id

  grants = [
    {
      principal  = "account users"
      privileges = ["CREATE_EXTERNAL_TABLE", "READ_FILES", "WRITE_FILES"]
    }
  ]

  depends_on = [module.bronze_location]
}

module "silver_location_grants" {
  source = "../../modules/grants"

  external_location = module.silver_location.location_id

  grants = [
    {
      principal  = "account users"
      privileges = ["CREATE_EXTERNAL_TABLE", "READ_FILES", "WRITE_FILES"]
    }
  ]

  depends_on = [module.silver_location]
}

module "gold_location_grants" {
  source = "../../modules/grants"

  external_location = module.gold_location.location_id

  grants = [
    {
      principal  = "account users"
      privileges = ["CREATE_EXTERNAL_TABLE", "READ_FILES", "WRITE_FILES"]
    }
  ]

  depends_on = [module.gold_location]
}

# Schema Grants
module "bronze_schema_grants" {
  source = "../../modules/grants"

  schema = module.bronze_schema.schema_full_name

  grants = [
    {
      principal  = "account users"
      privileges = ["USE_SCHEMA", "SELECT", "MODIFY", "CREATE_TABLE", "CREATE_VOLUME"]
    }
  ]

  depends_on = [module.bronze_schema]
}

module "silver_schema_grants" {
  source = "../../modules/grants"

  schema = module.silver_schema.schema_full_name

  grants = [
    {
      principal  = "account users"
      privileges = ["USE_SCHEMA", "SELECT", "MODIFY", "CREATE_TABLE", "CREATE_VOLUME"]
    }
  ]

  depends_on = [module.silver_schema]
}

module "gold_schema_grants" {
  source = "../../modules/grants"

  schema = module.gold_schema.schema_full_name

  grants = [
    {
      principal  = "account users"
      privileges = ["USE_SCHEMA", "SELECT", "CREATE_TABLE", "CREATE_VOLUME"]
    }
  ]

  depends_on = [module.gold_schema]
}

# Volume Grants
module "bronze_landing_volume_grants" {
  source = "../../modules/grants"

  volume = module.bronze_landing_volume.volume_id

  grants = [
    {
      principal  = "account users"
      privileges = ["READ_VOLUME", "WRITE_VOLUME"]
    }
  ]

  depends_on = [module.bronze_landing_volume]
}

module "bronze_archive_volume_grants" {
  source = "../../modules/grants"

  volume = module.bronze_archive_volume.volume_id

  grants = [
    {
      principal  = "account users"
      privileges = ["READ_VOLUME", "WRITE_VOLUME"]
    }
  ]

  depends_on = [module.bronze_archive_volume]
}

module "silver_processed_volume_grants" {
  source = "../../modules/grants"

  volume = module.silver_processed_volume.volume_id

  grants = [
    {
      principal  = "account users"
      privileges = ["READ_VOLUME", "WRITE_VOLUME"]
    }
  ]

  depends_on = [module.silver_processed_volume]
}

module "gold_exports_volume_grants" {
  source = "../../modules/grants"

  volume = module.gold_exports_volume.volume_id

  grants = [
    {
      principal  = "account users"
      privileges = ["READ_VOLUME", "WRITE_VOLUME"]
    }
  ]

  depends_on = [module.gold_exports_volume]
}
File: environments/dev/outputs.tf
# Dev Environment Outputs

output "storage_containers" {
  description = "Created storage containers"
  value = {
    bronze = module.bronze_container.container_name
    silver = module.silver_container.container_name
    gold   = module.gold_container.container_name
  }
}

output "storage_credential" {
  description = "Storage credential details"
  value = {
    name = module.storage_credential.credential_name
    id   = module.storage_credential.credential_id
  }
}

output "external_locations" {
  description = "External locations"
  value = {
    bronze = {
      name = module.bronze_location.location_name
      url  = module.bronze_location.location_url
    }
    silver = {
      name = module.silver_location.location_name
      url  = module.silver_location.location_url
    }
    gold = {
      name = module.gold_location.location_name
      url  = module.gold_location.location_url
    }
  }
}

output "schemas" {
  description = "Unity Catalog schemas"
  value = {
    bronze = module.bronze_schema.schema_full_name
    silver = module.silver_schema.schema_full_name
    gold   = module.gold_schema.schema_full_name
  }
}

output "volumes" {
  description = "Unity Catalog volumes"
  value = {
    bronze_landing   = module.bronze_landing_volume.volume_full_name
    bronze_archive   = module.bronze_archive_volume.volume_full_name
    silver_processed = module.silver_processed_volume.volume_full_name
    gold_exports     = module.gold_exports_volume.volume_full_name
  }
}

output "volume_paths" {
  description = "Volume access paths"
  value = {
    bronze_landing   = module.bronze_landing_volume.volume_path
    bronze_archive   = module.bronze_archive_volume.volume_path
    silver_processed = module.silver_processed_volume.volume_path
    gold_exports     = module.gold_exports_volume.volume_path
  }
}

output "deployment_summary" {
  description = "Complete deployment summary"
  value = {
    environment       = var.environment
    catalog           = var.catalog_name
    containers        = 3
    storage_credential = 1
    external_locations = 3
    schemas           = 3
    volumes           = 4
    grants            = 11
  }
}
PHASE 4: CREATE ROOT FILES
File: .gitignore (root of repository)
# Terraform
**/.terraform/*
*.tfstate
*.tfstate.*
*.tfvars
!*.tfvars.example
.terraform.lock.hcl
*.tfplan
crash.log
crash.*.log

# Sensitive files
*.pem
*.key
*.env
.env.*
secrets.yml
secrets.yaml

# IDE / Editor
.vscode/
.idea/
*.swp
*.swo
*.swn
.DS_Store
Thumbs.db

# Python
__pycache__/
*.py[cod]
*$py.class
.Python
venv/
env/

# Logs
*.log
logs/
*.tmp

# Backup files
*.backup
*.bak
*~
File: README.md (root of repository)
# Databricks Unity Catalog - Modular Terraform Infrastructure

Production-grade modular Terraform configuration for Azure Databricks Unity Catalog.

## 🏗️ Architecture

### Modular Structure

This repository uses **8 reusable modules** for maximum flexibility:

1. **catalog** - Creates Unity Catalogs
2. **schema** - Creates schemas/databases
3. **volume** - Creates external/managed volumes
4. **storage-credential** - Manages storage credentials
5. **external-location** - Creates external locations
6. **storage-container** - Creates Azure storage containers
7. **grants** - Manages permissions
8. **metastore-assignment** - Assigns metastore to workspace

### Why Modular?

✅ **Reusable**: Same modules for dev, staging, prod  
✅ **Testable**: Test each module independently  
✅ **Maintainable**: Update one module without affecting others  
✅ **Scalable**: Easy to add new environments or resources  
✅ **Industry Standard**: Follows Terraform best practices  

## 📁 Structure
terraform/
├── modules/              # Reusable modules (8 modules, 32 files)
│   ├── catalog/
│   ├── schema/
│   ├── volume/
│   ├── storage-credential/
│   ├── external-location/
│   ├── storage-container/
│   ├── grants/
│   └── metastore-assignment/
│
└── environments/         # Environment-specific configs
└── dev/
├── backend.tf
├── main.tf      # Calls all modules
├── variables.tf
└── outputs.tf
## 📋 What Gets Created

Per environment (dev):

- **3** Azure Storage Containers (bronze, silver, gold)
- **1** Storage Credential
- **3** External Locations
- **3** Schemas (bronze, silver, gold)
- **4** Volumes (landing_zone, archive, processed, exports)
- **11** Grant resources (permissions)

**Total: 25 resources**

## 🔐 Required GitHub Secrets

### 1. AZURE_AND_TFSTATE
```json
{
  "client_id": "xxxxx",
  "client_secret": "xxxxx",
  "tenant_id": "xxxxx",
  "subscription_id": "xxxxx",
  "state_resource_group": "rg-terraform-state",
  "state_storage_account": "sttfstatedatabricks",
  "state_container": "tfstate"
}
2. DBX_DEV
{
  "workspace_url": "https://adb-xxxxx.azuredatabricks.net",
  "token": "dapixxxxx"
}
3. DEV_ENV_DETAILS
{
  "resource_group": "rg-databricks-dev",
  "storage_account": "stdevdatalake001",
  "access_connector_id": "/subscriptions/.../accessConnectors/dac-dev"
}
🚀 Usage
Module Example
# Create a schema
module "analytics_schema" {
  source = "../../modules/schema"
  
  catalog_name = "dev_terraform_catalog"
  schema_name  = "analytics"
  comment      = "Analytics data"
  
  properties = {
    team = "analytics"
  }
}

# Create a volume
module "analytics_volume" {
  source = "../../modules/volume"
  
  volume_name      = "reports"
  catalog_name     = "dev_terraform_catalog"
  schema_name      = "analytics"
  volume_type      = "EXTERNAL"
  storage_location = "abfss://gold@storage.../reports/"
}
🎯 Deployment Workflow
Create Feature Branch
Make Changes (add/modify modules or environment configs)
Create Pull Request → Terraform plan runs automatically
Review Plan in PR comments
Merge → Auto-deploys to dev
✅ Verification
After deployment:
Databricks UI:
Data → Unity Catalog → dev_terraform_catalog
Verify schemas: bronze, silver, gold
Verify volumes
Azure Portal:
Storage Account → Containers
Verify: bronze, silver, gold
Test Access (Databricks notebook):
dbutils.fs.ls("/Volumes/dev_terraform_catalog/bronze/landing_zone/")
📚 Module Documentation
Each module has its own README with:
Usage examples
Input variables
Outputs
Requirements
See: terraform/modules/<module-name>/README.md
🔧 Customization
Adding New Schema
module "new_schema" {
  source = "../../modules/schema"
  
  catalog_name = var.catalog_name
  schema_name  = "my_new_schema"
  comment      = "My description"
}
Custom Grants
module "custom_grants" {
  source = "../../modules/grants"
  
  schema = "dev_terraform_catalog.bronze"
  
  grants = [
    {
      principal  = "data_engineers"
      privileges = ["ALL_PRIVILEGES"]
    }
  ]
}
🐛 Troubleshooting
Module Not Found
Error: Module not found: ../../modules/schema

Solution: Ensure you're running terraform from environments/dev/ directory
cd terraform/environments/dev
terraform init
Variable Not Defined
Error: Variable not declared in module

Solution: Check module's variables.tf for required variables
📞 Support
Terraform Issues: Check module README
Databricks Issues: Unity Catalog documentation
Access Issues: Contact admin
Architecture: Modular
Provider: Azure Databricks
Catalog: dev_terraform_catalog
Maintained by: Platform Team
Last Updated: February 2026
---

## ✅ FILES COMPLETE!

You now have **ALL 36 files**!

---

## PHASE 5: COMMIT AND PUSH

### Step 1: Save All Files

Make sure all files are saved in VS Code (Ctrl+S or Cmd+S)

### Step 2: Open GitHub Desktop

1. Switch to GitHub Desktop
2. You should see **36 new files** in the Changes list

### Step 3: Review Changes

Scroll through and verify you see:
- ✅ terraform/modules/ (8 folders with 4 files each)
- ✅ terraform/environments/dev/ (4 files)
- ✅ .gitignore
- ✅ README.md

### Step 4: Commit

1. **Summary**: `Add modular Terraform structure for Unity Catalog`
2. **Description**:
Added 8 reusable modules (catalog, schema, volume, etc.)
Added dev environment configuration
Using catalog: dev_terraform_catalog
Follows organization's module pattern
Production-grade architecture
3. Click **Commit to [your-feature-branch]**

### Step 5: Push

Click **Push origin** (top right)

---

## PHASE 6: VERIFY ON GITHUB

1. Go to your GitHub repository
2. Switch to your feature branch
3. Navigate to `terraform/modules/`
4. You should see all 8 module folders!

---

## 🎉 CONGRATULATIONS!

You've created a **production-grade modular Terraform structure**!

### What You Have:

✅ 8 reusable modules  
✅ Clean separation of concerns  
✅ Dev environment ready to deploy  
✅ Easy to add prod environment  
✅ Industry best practices  
✅ Matches your organization's pattern  

### Next Steps:

1. **Create Pull Request**
2. **Review Terraform Plan**
3. **Merge to Deploy**
4. **Verify in Databricks**

---

**Ready to create that Pull Request?** 🚀