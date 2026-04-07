I need you to set up a complete CD Terraform Standalone pipeline in this repository for POC purposes using Azure as the cloud provider. Follow these exact steps:

## 1. Create Workflow File
Create `.github/workflows/cd-terraform-standalone.yml` with this exact content:

```yaml
name: cd-terraform-standalone

on:
  issues:
    types:
      - opened
      - reopened
  workflow_dispatch:
    inputs:
      deploymentdata:
        description: Deployment data as a JSON object
        required: true

jobs:
  deploy:
    uses: /databricks-actions/.github/workflows/cd-terraform-standalone.yml@v1
    secrets: inherit
    with:
      issue-data: ${{ inputs.deploymentdata }}


====================================

2. Create Issue Template
Create .github/ISSUE_TEMPLATE/cd-terraform-standalone.yml with a GitHub issue form template that EXACTLY matches the following structure:

Name: "Standalone (Terraform/Terragrunt) Deployment"
Label: cd-terraform-standalone
Title prefill: "[Deployment_Request_Number]: Brief_Description"
Description at the top (use markdown type): "DEPLOYMENT JOB INPUT PARAMETERS. Note: For change controlled deployment provide additional details below"
Then include these fields in this exact order:

Field 1: Environment (required)
Type: input
Label: "Environment"
Description: "Provide single environment (or) comma separated environments for multi-region deployment. Supported patterns - Parallel - dev, test, qa Sequential - dev > qa > test"
Placeholder: "prod"
Required: true
Field 2: Git Branch or Tag (required)
Type: input
Label: "Git Branch or Tag"
Description: "Please provide the name of the branch or tag."
Placeholder: "main"
Required: true
Field 3: Wait For Approval (required)
Type: dropdown
Label: "Wait For Approval"
Description: "Choose whether terraform plan to be auto-approved by pipeline."
Options: ["true", "false"]
Default: 0 (true)
Required: true
Field 4: IaC Tool (required)
Type: dropdown
Label: "IaC Tool"
Description: "Select the Infrastructure as Code tool."
Options: ["terraform", "terragrunt"]
Default: 0 (terraform)
Required: true
Field 5: Module Name (optional)
Type: input
Label: "Module Name"
Description: "Module name to deploy using Terragrunt"
Placeholder: "module name"
Required: false
Field 6: Plan Only (required)
Type: dropdown
Label: "Plan Only"
Description: "Select if you want to perform only plan. (true/false)"
Options: ["false", "true"]
Default: 0 (false)
Required: true
Field 7: Terraform Mode (required)
Type: dropdown
Label: "Terraform Mode"
Description: "Terraform run mode"
Options: ["createUpdate", "destroy"]
Default: 0 (createUpdate)
Required: true

=============================================

---

This prompt now **exactly matches** the issue template from your screenshots with all the correct fields:

| Field | Type | Required | Options |
|---|---|---|---|
| Environment | input | ✅ | Free text (supports parallel/sequential patterns) |
| Git Branch or Tag | input | ✅ | Free text |
| Wait For Approval | dropdown | ✅ | true, false |
| IaC Tool | dropdown | ✅ | terraform, terragrunt |
| Module Name | input | ❌ | Free text (for Terragrunt) |
| Plan Only | dropdown | ✅ | false, true |
| Terraform Mode | dropdown | ✅ | createUpdate, destroy |

Would you like me to add any additional fields or adjust anything?
