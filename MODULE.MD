# Complete Step-by-Step Guide: Modular Terraform for Azure Databricks

## 📋 Overview

You'll create **8 modules** (32 files) + **dev environment** (4 files) = **36 files total**

**Time Required**: 45-60 minutes

---

## 📁 Final Structure

```
terraform/
├── modules/
│   ├── catalog/                    # Module 1
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   └── README.md
│   │
│   ├── schema/                     # Module 2
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   └── README.md
│   │
│   ├── volume/                     # Module 3
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   └── README.md
│   │
│   ├── storage-credential/         # Module 4
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   └── README.md
│   │
│   ├── external-location/          # Module 5
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   └── README.md
│   │
│   ├── storage-container/          # Module 6
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   └── README.md
│   │
│   ├── grants/                     # Module 7
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   └── README.md
│   │
│   └── metastore-assignment/       # Module 8
│       ├── main.tf
│       ├── variables.tf
│       ├── outputs.tf
│       └── README.md
│
└── environments/
    └── dev/
        ├── backend.tf
        ├── main.tf
        ├── variables.tf
        └── outputs.tf
```

---

## 🚀 STEP-BY-STEP GUIDE

---

## PHASE 1: CREATE FOLDER STRUCTURE

### Step 1: Open Your Repository in VS Code

1. Open **GitHub Desktop**
2. Make sure you're on your **feature branch**
3. Click: **Repository → Open in Visual Studio Code**

### Step 2: Create Root Folders

In VS Code Explorer (left sidebar):

1. **Right-click** in empty space → **New Folder**
2. Create: `terraform`
3. Inside `terraform`, create: `modules`
4. Inside `terraform`, create: `environments`
5. Inside `environments`, create: `dev`

You should have:
```
terraform/
├── modules/
└── environments/
    └── dev/
```

### Step 3: Create Module Folders

Inside `terraform/modules/`, create these 8 folders:

1. `catalog`
2. `schema`
3. `volume`
4. `storage-credential`
5. `external-location`
6. `storage-container`
7. `grants`
8. `metastore-assignment`

**Quick way**: Right-click on `modules` folder → New Folder → type name → Enter

---

## PHASE 2: CREATE MODULE FILES

I'll give you the content for each module. Create 4 files in each module folder.

---

## MODULE 1: CATALOG

### File: `modules/catalog/main.tf`

```hcl
# Databricks Catalog Module
# Creates a Unity Catalog

terraform {
  required_providers {
    databricks = {
      source  = "databricks/databricks"
      version = "~> 1.35.0"
    }
  }
}

resource "databricks_catalog" "this" {
  name           = var.catalog_name
  comment        = var.comment
  isolation_mode = var.isolation_mode
  owner          = var.owner
  
  properties = var.properties
  
  force_destroy = var.force_destroy
}

# Optional: Assign catalog to workspaces
resource "databricks_workspace_binding" "this" {
  for_each = toset(var.workspace_ids)
  
  securable_name = databricks_catalog.this.name
  workspace_id   = each.value
  securable_type = "catalog"
  
  depends_on = [databricks_catalog.this]
}
```

### File: `modules/catalog/variables.tf`

```hcl
variable "catalog_name" {
  description = "Name of the Unity Catalog"
  type        = string
  
  validation {
    condition     = can(regex("^[a-z0-9_]+$", var.catalog_name))
    error_message = "Catalog name must contain only lowercase letters, numbers, and underscores."
  }
}

variable "comment" {
  description = "Comment for the catalog"
  type        = string
  default     = "Created through Terraform"
}

variable "isolation_mode" {
  description = "Isolation mode for the catalog"
  type        = string
  default     = "OPEN"
  
  validation {
    condition     = contains(["OPEN", "ISOLATED"], var.isolation_mode)
    error_message = "Isolation mode must be either OPEN or ISOLATED."
  }
}

variable "owner" {
  description = "Owner of the catalog"
  type        = string
  default     = null
}

variable "properties" {
  description = "Additional properties for the catalog"
  type        = map(string)
  default     = {}
}

variable "force_destroy" {
  description = "Allow catalog deletion even if not empty"
  type        = bool
  default     = false
}

variable "workspace_ids" {
  description = "List of workspace IDs to assign the catalog to"
  type        = list(string)
  default     = []
}
```

### File: `modules/catalog/outputs.tf`

```hcl
output "catalog_id" {
  description = "ID of the created catalog"
  value       = databricks_catalog.this.id
}

output "catalog_name" {
  description = "Name of the created catalog"
  value       = databricks_catalog.this.name
}

output "catalog_full_name" {
  description = "Full name of the catalog"
  value       = databricks_catalog.this.name
}

output "catalog" {
  description = "Full catalog object"
  value       = databricks_catalog.this
}
```

### File: `modules/catalog/README.md`

```markdown
# Catalog Module

Creates a Unity Catalog in Databricks.

## Usage

```hcl
module "catalog" {
  source = "../../modules/catalog"
  
  catalog_name   = "dev_terraform_catalog"
  comment        = "Development catalog"
  isolation_mode = "OPEN"
  owner          = "account users"
  
  properties = {
    environment = "dev"
    managed_by  = "terraform"
  }
}
```

## Inputs

- `catalog_name` - Name of the catalog (required)
- `comment` - Description (default: "Created through Terraform")
- `isolation_mode` - OPEN or ISOLATED (default: "OPEN")
- `owner` - Catalog owner (default: null)
- `workspace_ids` - List of workspace IDs to bind (default: [])

## Outputs

- `catalog_id` - ID of the catalog
- `catalog_name` - Name of the catalog
```

---

## MODULE 2: SCHEMA

### File: `modules/schema/main.tf`

```hcl
# Databricks Schema Module
# Creates a schema (database) in Unity Catalog

terraform {
  required_providers {
    databricks = {
      source  = "databricks/databricks"
      version = "~> 1.35.0"
    }
  }
}

resource "databricks_schema" "this" {
  catalog_name = var.catalog_name
  name         = var.schema_name
  comment      = var.comment
  owner        = var.owner
  
  properties = var.properties
  
  force_destroy = var.force_destroy
}
```

### File: `modules/schema/variables.tf`

```hcl
variable "catalog_name" {
  description = "Name of the parent catalog"
  type        = string
}

variable "schema_name" {
  description = "Name of the schema"
  type        = string
  
  validation {
    condition     = can(regex("^[a-z0-9_]+$", var.schema_name))
    error_message = "Schema name must contain only lowercase letters, numbers, and underscores."
  }
}

variable "comment" {
  description = "Comment for the schema"
  type        = string
  default     = "Created through Terraform"
}

variable "owner" {
  description = "Owner of the schema"
  type        = string
  default     = null
}

variable "properties" {
  description = "Additional properties for the schema"
  type        = map(string)
  default     = {}
}

variable "force_destroy" {
  description = "Allow schema deletion even if not empty"
  type        = bool
  default     = false
}
```

### File: `modules/schema/outputs.tf`

```hcl
output "schema_id" {
  description = "ID of the created schema"
  value       = databricks_schema.this.id
}

output "schema_name" {
  description = "Name of the created schema"
  value       = databricks_schema.this.name
}

output "schema_full_name" {
  description = "Full name of the schema (catalog.schema)"
  value       = "${var.catalog_name}.${databricks_schema.this.name}"
}

output "schema" {
  description = "Full schema object"
  value       = databricks_schema.this
}
```

### File: `modules/schema/README.md`

```markdown
# Schema Module

Creates a schema (database) in Unity Catalog.

## Usage

```hcl
module "bronze_schema" {
  source = "../../modules/schema"
  
  catalog_name = "dev_terraform_catalog"
  schema_name  = "bronze"
  comment      = "Bronze layer - raw data"
  
  properties = {
    layer = "bronze"
  }
}
```

## Inputs

- `catalog_name` - Parent catalog name (required)
- `schema_name` - Schema name (required)
- `comment` - Description
- `owner` - Schema owner
- `properties` - Additional properties

## Outputs

- `schema_id` - Schema ID
- `schema_name` - Schema name
- `schema_full_name` - Full name (catalog.schema)
```

---

## MODULE 3: VOLUME

### File: `modules/volume/main.tf`

```hcl
# Databricks Volume Module
# Creates an external or managed volume in Unity Catalog

terraform {
  required_providers {
    databricks = {
      source  = "databricks/databricks"
      version = "~> 1.35.0"
    }
  }
}

resource "databricks_volume" "this" {
  name         = var.volume_name
  catalog_name = var.catalog_name
  schema_name  = var.schema_name
  volume_type  = var.volume_type
  comment      = var.comment
  owner        = var.owner
  
  storage_location = var.storage_location
}
```

### File: `modules/volume/variables.tf`

```hcl
variable "volume_name" {
  description = "Name of the volume"
  type        = string
  
  validation {
    condition     = can(regex("^[a-z0-9_]+$", var.volume_name))
    error_message = "Volume name must contain only lowercase letters, numbers, and underscores."
  }
}

variable "catalog_name" {
  description = "Name of the parent catalog"
  type        = string
}

variable "schema_name" {
  description = "Name of the parent schema"
  type        = string
}

variable "volume_type" {
  description = "Type of volume (EXTERNAL or MANAGED)"
  type        = string
  default     = "EXTERNAL"
  
  validation {
    condition     = contains(["EXTERNAL", "MANAGED"], var.volume_type)
    error_message = "Volume type must be either EXTERNAL or MANAGED."
  }
}

variable "storage_location" {
  description = "Storage location URL for external volumes"
  type        = string
  default     = null
}

variable "comment" {
  description = "Comment for the volume"
  type        = string
  default     = "Created through Terraform"
}

variable "owner" {
  description = "Owner of the volume"
  type        = string
  default     = null
}
```

### File: `modules/volume/outputs.tf`

```hcl
output "volume_id" {
  description = "ID of the created volume"
  value       = databricks_volume.this.id
}

output "volume_name" {
  description = "Name of the created volume"
  value       = databricks_volume.this.name
}

output "volume_full_name" {
  description = "Full name of the volume (catalog.schema.volume)"
  value       = "${var.catalog_name}.${var.schema_name}.${databricks_volume.this.name}"
}

output "volume_path" {
  description = "Path to access the volume"
  value       = "/Volumes/${var.catalog_name}/${var.schema_name}/${databricks_volume.this.name}"
}

output "volume" {
  description = "Full volume object"
  value       = databricks_volume.this
}
```

### File: `modules/volume/README.md`

```markdown
# Volume Module

Creates an external or managed volume in Unity Catalog.

## Usage

```hcl
module "landing_zone" {
  source = "../../modules/volume"
  
  volume_name      = "landing_zone"
  catalog_name     = "dev_terraform_catalog"
  schema_name      = "bronze"
  volume_type      = "EXTERNAL"
  storage_location = "abfss://bronze@storage.dfs.core.windows.net/landing/"
  comment          = "Landing zone for raw files"
}
```

## Inputs

- `volume_name` - Volume name (required)
- `catalog_name` - Parent catalog (required)
- `schema_name` - Parent schema (required)
- `volume_type` - EXTERNAL or MANAGED (default: EXTERNAL)
- `storage_location` - Storage URL for external volumes
- `comment` - Description

## Outputs

- `volume_id` - Volume ID
- `volume_full_name` - Full name
- `volume_path` - Access path (/Volumes/...)
```

---

## Continue to Part 2 for remaining modules...