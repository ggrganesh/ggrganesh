Create new issue in githu/repo: Terraform Standalone (Terraform/Terragrunt) Deployment
Add a title
*
[Deployment_Request_Number]: Brief_Description
DEPLOYMENT JOB INPUT PARAMETERES. Note: For change controlled deployment provide additional details below.

Environment
*
Provide single environment (or) comma separated environments for multi-region deployment. Supported patterns -

Parallel - dev, test, qa          Sequential - dev > qa > test

prod
Git Branch or Tag
*
Please provide the name of the branch or tag.

Wait For Approval
*
Choose whether terraform plan to be auto-approved by pipeline.

Terraform Command
*
Name of the CLI to use for deployment

Module Name
Module name to deploy using Terragrunt

module name
Plan Only
*
Select if you want to perform only plan. (true/false)

Terraform Mode
*
Terraform run mode



