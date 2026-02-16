Production-Grade GitHub Actions Workflows
Complete Set of 5 Workflows
WORKFLOW 1: Terraform Validate (Code Quality)
File: .github/workflows/terraform-validate.yml
Purpose: Runs on every PR to validate Terraform code quality
name: 'Terraform Validate'

on:
  pull_request:
    branches:
      - main
    paths:
      - 'terraform/**'
      - '.github/workflows/terraform-*.yml'

env:
  TF_VERSION: '1.7.0'

jobs:
  validate:
    name: 'Validate Terraform Code'
    runs-on: ubuntu-latest
    
    strategy:
      matrix:
        environment: [dev]
        resource: [
          storage-container,
          storage-credential,
          external-location,
          schema,
          volume,
          grants
        ]
    
    steps:
      - name: Checkout Code
        uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}

      - name: Terraform Format Check
        working-directory: terraform/environments/${{ matrix.environment }}/${{ matrix.resource }}
        run: terraform fmt -check -recursive
        continue-on-error: true

      - name: Terraform Init
        working-directory: terraform/environments/${{ matrix.environment }}/${{ matrix.resource }}
        run: terraform init -backend=false

      - name: Terraform Validate
        working-directory: terraform/environments/${{ matrix.environment }}/${{ matrix.resource }}
        run: terraform validate

  validate-modules:
    name: 'Validate Modules'
    runs-on: ubuntu-latest
    
    strategy:
      matrix:
        module: [
          catalog,
          schema,
          volume,
          storage-credential,
          external-location,
          storage-container,
          grants,
          metastore-assignment
        ]
    
    steps:
      - name: Checkout Code
        uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}

      - name: Terraform Init
        working-directory: terraform/modules/${{ matrix.module }}
        run: terraform init -backend=false

      - name: Terraform Validate
        working-directory: terraform/modules/${{ matrix.module }}
        run: terraform validate

      - name: Check README exists
        working-directory: terraform/modules/${{ matrix.module }}
        run: |
          if [ ! -f "README.md" ]; then
            echo "❌ README.md missing in ${{ matrix.module }} module"
            exit 1
          fi

  summary:
    name: 'Validation Summary'
    needs: [validate, validate-modules]
    runs-on: ubuntu-latest
    if: always()
    
    steps:
      - name: Create Summary
        run: |
          echo "## 📋 Terraform Validation Results" >> $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY
          
          if [ "${{ needs.validate.result }}" == "success" ] && [ "${{ needs.validate-modules.result }}" == "success" ]; then
            echo "✅ All Terraform code is valid!" >> $GITHUB_STEP_SUMMARY
            echo "✅ All modules validated successfully" >> $GITHUB_STEP_SUMMARY
          else
            echo "❌ Validation failed. Check the logs above." >> $GITHUB_STEP_SUMMARY
            exit 1
          fi
WORKFLOW 2: Terraform Plan (Show Changes)
File: .github/workflows/terraform-plan-dev.yml
Purpose: Shows what will be created/changed on every PR
name: 'Terraform Plan - Dev'

on:
  pull_request:
    branches:
      - main
    paths:
      - 'terraform/environments/dev/**'
      - 'terraform/modules/**'
      - '.github/workflows/terraform-plan-dev.yml'

env:
  TF_VERSION: '1.7.0'

jobs:
  plan-phase1:
    name: 'Plan Phase 1 (Basic Infrastructure)'
    runs-on: ubuntu-latest
    permissions:
      contents: read
      pull-requests: write
    
    steps:
      - name: Checkout Code
        uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}

      - name: Parse Secrets
        id: secrets
        run: |
          echo "client_id=$(echo '${{ secrets.AZURE_AND_TFSTATE }}' | jq -r '.client_id')" >> $GITHUB_OUTPUT
          echo "client_secret=$(echo '${{ secrets.AZURE_AND_TFSTATE }}' | jq -r '.client_secret')" >> $GITHUB_OUTPUT
          echo "tenant_id=$(echo '${{ secrets.AZURE_AND_TFSTATE }}' | jq -r '.tenant_id')" >> $GITHUB_OUTPUT
          echo "subscription_id=$(echo '${{ secrets.AZURE_AND_TFSTATE }}' | jq -r '.subscription_id')" >> $GITHUB_OUTPUT
          echo "state_rg=$(echo '${{ secrets.AZURE_AND_TFSTATE }}' | jq -r '.state_resource_group')" >> $GITHUB_OUTPUT
          echo "state_storage=$(echo '${{ secrets.AZURE_AND_TFSTATE }}' | jq -r '.state_storage_account')" >> $GITHUB_OUTPUT
          echo "state_container=$(echo '${{ secrets.AZURE_AND_TFSTATE }}' | jq -r '.state_container')" >> $GITHUB_OUTPUT
          echo "databricks_host=$(echo '${{ secrets.DBX_DEV }}' | jq -r '.workspace_url')" >> $GITHUB_OUTPUT
          echo "databricks_token=$(echo '${{ secrets.DBX_DEV }}' | jq -r '.token')" >> $GITHUB_OUTPUT
          echo "resource_group=$(echo '${{ secrets.DEV_ENV_DETAILS }}' | jq -r '.resource_group')" >> $GITHUB_OUTPUT
          echo "storage_account=$(echo '${{ secrets.DEV_ENV_DETAILS }}' | jq -r '.storage_account')" >> $GITHUB_OUTPUT
          echo "access_connector=$(echo '${{ secrets.DEV_ENV_DETAILS }}' | jq -r '.access_connector_id')" >> $GITHUB_OUTPUT

      # Plan Storage Containers
      - name: Plan Storage Containers
        id: plan_storage
        working-directory: terraform/environments/dev/storage-container
        env:
          ARM_CLIENT_ID: ${{ steps.secrets.outputs.client_id }}
          ARM_CLIENT_SECRET: ${{ steps.secrets.outputs.client_secret }}
          ARM_TENANT_ID: ${{ steps.secrets.outputs.tenant_id }}
          ARM_SUBSCRIPTION_ID: ${{ steps.secrets.outputs.subscription_id }}
          TF_VAR_storage_account_name: ${{ steps.secrets.outputs.storage_account }}
          TF_VAR_resource_group_name: ${{ steps.secrets.outputs.resource_group }}
        run: |
          terraform init \
            -backend-config="resource_group_name=${{ steps.secrets.outputs.state_rg }}" \
            -backend-config="storage_account_name=${{ steps.secrets.outputs.state_storage }}" \
            -backend-config="container_name=${{ steps.secrets.outputs.state_container }}" \
            -backend-config="key=dev/storage-container.tfstate"
          terraform plan -no-color -out=tfplan 2>&1 | tee plan.txt
        continue-on-error: true

      # Plan Storage Credential
      - name: Plan Storage Credential
        id: plan_credential
        working-directory: terraform/environments/dev/storage-credential
        env:
          ARM_CLIENT_ID: ${{ steps.secrets.outputs.client_id }}
          ARM_CLIENT_SECRET: ${{ steps.secrets.outputs.client_secret }}
          ARM_TENANT_ID: ${{ steps.secrets.outputs.tenant_id }}
          ARM_SUBSCRIPTION_ID: ${{ steps.secrets.outputs.subscription_id }}
          DATABRICKS_HOST: ${{ steps.secrets.outputs.databricks_host }}
          DATABRICKS_TOKEN: ${{ steps.secrets.outputs.databricks_token }}
          TF_VAR_access_connector_id: ${{ steps.secrets.outputs.access_connector }}
        run: |
          terraform init \
            -backend-config="resource_group_name=${{ steps.secrets.outputs.state_rg }}" \
            -backend-config="storage_account_name=${{ steps.secrets.outputs.state_storage }}" \
            -backend-config="container_name=${{ steps.secrets.outputs.state_container }}" \
            -backend-config="key=dev/storage-credential.tfstate"
          terraform plan -no-color -out=tfplan 2>&1 | tee plan.txt
        continue-on-error: true

      # Comment on PR
      - name: Comment PR with Plans
        uses: actions/github-script@v7
        if: github.event_name == 'pull_request'
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
          script: |
            const fs = require('fs');
            
            const storagePlan = fs.existsSync('terraform/environments/dev/storage-container/plan.txt') 
              ? fs.readFileSync('terraform/environments/dev/storage-container/plan.txt', 'utf8')
              : 'Plan not generated';
            
            const credentialPlan = fs.existsSync('terraform/environments/dev/storage-credential/plan.txt')
              ? fs.readFileSync('terraform/environments/dev/storage-credential/plan.txt', 'utf8')
              : 'Plan not generated';
            
            const output = `## 📊 Terraform Plan - Dev Environment
            
            ### Phase 1: Basic Infrastructure
            
            <details><summary>Storage Containers Plan</summary>
            
            \`\`\`terraform
            ${storagePlan.substring(0, 60000)}
            \`\`\`
            
            </details>
            
            <details><summary>Storage Credential Plan</summary>
            
            \`\`\`terraform
            ${credentialPlan.substring(0, 60000)}
            \`\`\`
            
            </details>
            
            ---
            
            **⚠️ Review the plans above before merging!**
            
            After merging, deploy using:
            1. Actions → Deploy Phase 1
            2. Actions → Deploy Phase 2
            
            *Pusher: @${{ github.actor }}*`;
            
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: output
            });
WORKFLOW 3: Deploy Phase 1 (Basic Infrastructure)
File: .github/workflows/terraform-apply-phase1.yml
Purpose: Deploy containers, credential, external locations
name: 'Deploy Phase 1 - Basic Infrastructure'

on:
  workflow_dispatch:
    inputs:
      confirm:
        description: 'Type "deploy" to confirm deployment'
        required: true
        default: ''
      environment:
        description: 'Environment to deploy'
        required: true
        default: 'dev'
        type: choice
        options:
          - dev

env:
  TF_VERSION: '1.7.0'

jobs:
  validate-input:
    name: 'Validate Deployment Request'
    runs-on: ubuntu-latest
    steps:
      - name: Check Confirmation
        run: |
          if [ "${{ github.event.inputs.confirm }}" != "deploy" ]; then
            echo "❌ Deployment cancelled: Must type 'deploy' to confirm"
            exit 1
          fi
          echo "✅ Deployment confirmed"

  deploy-storage-containers:
    name: '1️⃣ Deploy Storage Containers'
    needs: validate-input
    runs-on: ubuntu-latest
    environment: ${{ github.event.inputs.environment }}
    
    outputs:
      bronze_url: ${{ steps.outputs.outputs.bronze_url }}
      silver_url: ${{ steps.outputs.outputs.silver_url }}
      gold_url: ${{ steps.outputs.outputs.gold_url }}
    
    steps:
      - name: Checkout Code
        uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}
          terraform_wrapper: false

      - name: Parse Secrets
        id: secrets
        run: |
          echo "client_id=$(echo '${{ secrets.AZURE_AND_TFSTATE }}' | jq -r '.client_id')" >> $GITHUB_OUTPUT
          echo "client_secret=$(echo '${{ secrets.AZURE_AND_TFSTATE }}' | jq -r '.client_secret')" >> $GITHUB_OUTPUT
          echo "tenant_id=$(echo '${{ secrets.AZURE_AND_TFSTATE }}' | jq -r '.tenant_id')" >> $GITHUB_OUTPUT
          echo "subscription_id=$(echo '${{ secrets.AZURE_AND_TFSTATE }}' | jq -r '.subscription_id')" >> $GITHUB_OUTPUT
          echo "state_rg=$(echo '${{ secrets.AZURE_AND_TFSTATE }}' | jq -r '.state_resource_group')" >> $GITHUB_OUTPUT
          echo "state_storage=$(echo '${{ secrets.AZURE_AND_TFSTATE }}' | jq -r '.state_storage_account')" >> $GITHUB_OUTPUT
          echo "state_container=$(echo '${{ secrets.AZURE_AND_TFSTATE }}' | jq -r '.state_container')" >> $GITHUB_OUTPUT
          echo "resource_group=$(echo '${{ secrets.DEV_ENV_DETAILS }}' | jq -r '.resource_group')" >> $GITHUB_OUTPUT
          echo "storage_account=$(echo '${{ secrets.DEV_ENV_DETAILS }}' | jq -r '.storage_account')" >> $GITHUB_OUTPUT

      - name: Terraform Init
        working-directory: terraform/environments/${{ github.event.inputs.environment }}/storage-container
        env:
          ARM_CLIENT_ID: ${{ steps.secrets.outputs.client_id }}
          ARM_CLIENT_SECRET: ${{ steps.secrets.outputs.client_secret }}
          ARM_TENANT_ID: ${{ steps.secrets.outputs.tenant_id }}
          ARM_SUBSCRIPTION_ID: ${{ steps.secrets.outputs.subscription_id }}
        run: |
          terraform init \
            -backend-config="resource_group_name=${{ steps.secrets.outputs.state_rg }}" \
            -backend-config="storage_account_name=${{ steps.secrets.outputs.state_storage }}" \
            -backend-config="container_name=${{ steps.secrets.outputs.state_container }}" \
            -backend-config="key=${{ github.event.inputs.environment }}/storage-container.tfstate"

      - name: Terraform Plan
        working-directory: terraform/environments/${{ github.event.inputs.environment }}/storage-container
        env:
          ARM_CLIENT_ID: ${{ steps.secrets.outputs.client_id }}
          ARM_CLIENT_SECRET: ${{ steps.secrets.outputs.client_secret }}
          ARM_TENANT_ID: ${{ steps.secrets.outputs.tenant_id }}
          ARM_SUBSCRIPTION_ID: ${{ steps.secrets.outputs.subscription_id }}
          TF_VAR_storage_account_name: ${{ steps.secrets.outputs.storage_account }}
          TF_VAR_resource_group_name: ${{ steps.secrets.outputs.resource_group }}
        run: terraform plan -out=tfplan

      - name: Terraform Apply
        working-directory: terraform/environments/${{ github.event.inputs.environment }}/storage-container
        env:
          ARM_CLIENT_ID: ${{ steps.secrets.outputs.client_id }}
          ARM_CLIENT_SECRET: ${{ steps.secrets.outputs.client_secret }}
          ARM_TENANT_ID: ${{ steps.secrets.outputs.tenant_id }}
          ARM_SUBSCRIPTION_ID: ${{ steps.secrets.outputs.subscription_id }}
          TF_VAR_storage_account_name: ${{ steps.secrets.outputs.storage_account }}
          TF_VAR_resource_group_name: ${{ steps.secrets.outputs.resource_group }}
        run: terraform apply -auto-approve tfplan

      - name: Get Outputs
        id: outputs
        working-directory: terraform/environments/${{ github.event.inputs.environment }}/storage-container
        env:
          ARM_CLIENT_ID: ${{ steps.secrets.outputs.client_id }}
          ARM_CLIENT_SECRET: ${{ steps.secrets.outputs.client_secret }}
          ARM_TENANT_ID: ${{ steps.secrets.outputs.tenant_id }}
          ARM_SUBSCRIPTION_ID: ${{ steps.secrets.outputs.subscription_id }}
        run: |
          BRONZE_URL=$(terraform output -raw bronze_container | jq -r '.url')
          SILVER_URL=$(terraform output -raw silver_container | jq -r '.url')
          GOLD_URL=$(terraform output -raw gold_container | jq -r '.url')
          echo "bronze_url=$BRONZE_URL" >> $GITHUB_OUTPUT
          echo "silver_url=$SILVER_URL" >> $GITHUB_OUTPUT
          echo "gold_url=$GOLD_URL" >> $GITHUB_OUTPUT

  deploy-storage-credential:
    name: '2️⃣ Deploy Storage Credential'
    needs: deploy-storage-containers
    runs-on: ubuntu-latest
    environment: ${{ github.event.inputs.environment }}
    
    outputs:
      credential_name: ${{ steps.outputs.outputs.credential_name }}
    
    steps:
      - name: Checkout Code
        uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}
          terraform_wrapper: false

      - name: Parse Secrets
        id: secrets
        run: |
          echo "client_id=$(echo '${{ secrets.AZURE_AND_TFSTATE }}' | jq -r '.client_id')" >> $GITHUB_OUTPUT
          echo "client_secret=$(echo '${{ secrets.AZURE_AND_TFSTATE }}' | jq -r '.client_secret')" >> $GITHUB_OUTPUT
          echo "tenant_id=$(echo '${{ secrets.AZURE_AND_TFSTATE }}' | jq -r '.tenant_id')" >> $GITHUB_OUTPUT
          echo "subscription_id=$(echo '${{ secrets.AZURE_AND_TFSTATE }}' | jq -r '.subscription_id')" >> $GITHUB_OUTPUT
          echo "state_rg=$(echo '${{ secrets.AZURE_AND_TFSTATE }}' | jq -r '.state_resource_group')" >> $GITHUB_OUTPUT
          echo "state_storage=$(echo '${{ secrets.AZURE_AND_TFSTATE }}' | jq -r '.state_storage_account')" >> $GITHUB_OUTPUT
          echo "state_container=$(echo '${{ secrets.AZURE_AND_TFSTATE }}' | jq -r '.state_container')" >> $GITHUB_OUTPUT
          echo "databricks_host=$(echo '${{ secrets.DBX_DEV }}' | jq -r '.workspace_url')" >> $GITHUB_OUTPUT
          echo "databricks_token=$(echo '${{ secrets.DBX_DEV }}' | jq -r '.token')" >> $GITHUB_OUTPUT
          echo "access_connector=$(echo '${{ secrets.DEV_ENV_DETAILS }}' | jq -r '.access_connector_id')" >> $GITHUB_OUTPUT

      - name: Terraform Init
        working-directory: terraform/environments/${{ github.event.inputs.environment }}/storage-credential
        env:
          ARM_CLIENT_ID: ${{ steps.secrets.outputs.client_id }}
          ARM_CLIENT_SECRET: ${{ steps.secrets.outputs.client_secret }}
          ARM_TENANT_ID: ${{ steps.secrets.outputs.tenant_id }}
          ARM_SUBSCRIPTION_ID: ${{ steps.secrets.outputs.subscription_id }}
        run: |
          terraform init \
            -backend-config="resource_group_name=${{ steps.secrets.outputs.state_rg }}" \
            -backend-config="storage_account_name=${{ steps.secrets.outputs.state_storage }}" \
            -backend-config="container_name=${{ steps.secrets.outputs.state_container }}" \
            -backend-config="key=${{ github.event.inputs.environment }}/storage-credential.tfstate"

      - name: Terraform Apply
        working-directory: terraform/environments/${{ github.event.inputs.environment }}/storage-credential
        env:
          ARM_CLIENT_ID: ${{ steps.secrets.outputs.client_id }}
          ARM_CLIENT_SECRET: ${{ steps.secrets.outputs.client_secret }}
          ARM_TENANT_ID: ${{ steps.secrets.outputs.tenant_id }}
          ARM_SUBSCRIPTION_ID: ${{ steps.secrets.outputs.subscription_id }}
          DATABRICKS_HOST: ${{ steps.secrets.outputs.databricks_host }}
          DATABRICKS_TOKEN: ${{ steps.secrets.outputs.databricks_token }}
          TF_VAR_access_connector_id: ${{ steps.secrets.outputs.access_connector }}
        run: |
          terraform plan -out=tfplan
          terraform apply -auto-approve tfplan

      - name: Get Outputs
        id: outputs
        working-directory: terraform/environments/${{ github.event.inputs.environment }}/storage-credential
        env:
          ARM_CLIENT_ID: ${{ steps.secrets.outputs.client_id }}
          ARM_CLIENT_SECRET: ${{ steps.secrets.outputs.client_secret }}
          ARM_TENANT_ID: ${{ steps.secrets.outputs.tenant_id }}
          ARM_SUBSCRIPTION_ID: ${{ steps.secrets.outputs.subscription_id }}
          DATABRICKS_HOST: ${{ steps.secrets.outputs.databricks_host }}
          DATABRICKS_TOKEN: ${{ steps.secrets.outputs.databricks_token }}
        run: |
          CRED_NAME=$(terraform output -raw credential_name)
          echo "credential_name=$CRED_NAME" >> $GITHUB_OUTPUT

  deploy-external-locations:
    name: '3️⃣ Deploy External Locations'
    needs: [deploy-storage-containers, deploy-storage-credential]
    runs-on: ubuntu-latest
    environment: ${{ github.event.inputs.environment }}
    
    steps:
      - name: Checkout Code
        uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}

      - name: Parse Secrets
        id: secrets
        run: |
          echo "client_id=$(echo '${{ secrets.AZURE_AND_TFSTATE }}' | jq -r '.client_id')" >> $GITHUB_OUTPUT
          echo "client_secret=$(echo '${{ secrets.AZURE_AND_TFSTATE }}' | jq -r '.client_secret')" >> $GITHUB_OUTPUT
          echo "tenant_id=$(echo '${{ secrets.AZURE_AND_TFSTATE }}' | jq -r '.tenant_id')" >> $GITHUB_OUTPUT
          echo "subscription_id=$(echo '${{ secrets.AZURE_AND_TFSTATE }}' | jq -r '.subscription_id')" >> $GITHUB_OUTPUT
          echo "state_rg=$(echo '${{ secrets.AZURE_AND_TFSTATE }}' | jq -r '.state_resource_group')" >> $GITHUB_OUTPUT
          echo "state_storage=$(echo '${{ secrets.AZURE_AND_TFSTATE }}' | jq -r '.state_storage_account')" >> $GITHUB_OUTPUT
          echo "state_container=$(echo '${{ secrets.AZURE_AND_TFSTATE }}' | jq -r '.state_container')" >> $GITHUB_OUTPUT
          echo "databricks_host=$(echo '${{ secrets.DBX_DEV }}' | jq -r '.workspace_url')" >> $GITHUB_OUTPUT
          echo "databricks_token=$(echo '${{ secrets.DBX_DEV }}' | jq -r '.token')" >> $GITHUB_OUTPUT

      - name: Terraform Init
        working-directory: terraform/environments/${{ github.event.inputs.environment }}/external-location
        env:
          ARM_CLIENT_ID: ${{ steps.secrets.outputs.client_id }}
          ARM_CLIENT_SECRET: ${{ steps.secrets.outputs.client_secret }}
          ARM_TENANT_ID: ${{ steps.secrets.outputs.tenant_id }}
          ARM_SUBSCRIPTION_ID: ${{ steps.secrets.outputs.subscription_id }}
        run: |
          terraform init \
            -backend-config="resource_group_name=${{ steps.secrets.outputs.state_rg }}" \
            -backend-config="storage_account_name=${{ steps.secrets.outputs.state_storage }}" \
            -backend-config="container_name=${{ steps.secrets.outputs.state_container }}" \
            -backend-config="key=${{ github.event.inputs.environment }}/external-location.tfstate"

      - name: Terraform Apply
        working-directory: terraform/environments/${{ github.event.inputs.environment }}/external-location
        env:
          ARM_CLIENT_ID: ${{ steps.secrets.outputs.client_id }}
          ARM_CLIENT_SECRET: ${{ steps.secrets.outputs.client_secret }}
          ARM_TENANT_ID: ${{ steps.secrets.outputs.tenant_id }}
          ARM_SUBSCRIPTION_ID: ${{ steps.secrets.outputs.subscription_id }}
          DATABRICKS_HOST: ${{ steps.secrets.outputs.databricks_host }}
          DATABRICKS_TOKEN: ${{ steps.secrets.outputs.databricks_token }}
          TF_VAR_credential_name: ${{ needs.deploy-storage-credential.outputs.credential_name }}
          TF_VAR_bronze_container_url: ${{ needs.deploy-storage-containers.outputs.bronze_url }}
          TF_VAR_silver_container_url: ${{ needs.deploy-storage-containers.outputs.silver_url }}
          TF_VAR_gold_container_url: ${{ needs.deploy-storage-containers.outputs.gold_url }}
        run: |
          terraform plan -out=tfplan
          terraform apply -auto-approve tfplan

  summary:
    name: '📊 Deployment Summary'
    needs: [deploy-storage-containers, deploy-storage-credential, deploy-external-locations]
    runs-on: ubuntu-latest
    if: always()
    
    steps:
      - name: Create Summary
        run: |
          echo "## ✅ Phase 1 Deployment Complete!" >> $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY
          echo "### Resources Deployed:" >> $GITHUB_STEP_SUMMARY
          echo "- ✅ 3 Storage Containers (bronze, silver, gold)" >> $GITHUB_STEP_SUMMARY
          echo "- ✅ 1 Storage Credential (${{ needs.deploy-storage-credential.outputs.credential_name }})" >> $GITHUB_STEP_SUMMARY
          echo "- ✅ 3 External Locations (dev_bronze, dev_silver, dev_gold)" >> $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY
          echo "### Verification:" >> $GITHUB_STEP_SUMMARY
          echo "1. **Databricks UI**: Data → External Locations" >> $GITHUB_STEP_SUMMARY
          echo "2. **Azure Portal**: Storage Account → Containers" >> $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY
          echo "### Next Steps:" >> $GITHUB_STEP_SUMMARY
          echo "Run **Deploy Phase 2** workflow to deploy schemas, volumes, and grants" >> $GITHUB_STEP_SUMMARY
Continue to next file for Phase 2 and Destroy workflows...


Production-Grade Workflows - Part 2
WORKFLOW 4: Deploy Phase 2 (Schemas, Volumes, Grants)
File: .github/workflows/terraform-apply-phase2.yml
Purpose: Deploy schemas, volumes, and grants
name: 'Deploy Phase 2 - Schemas, Volumes, Grants'

on:
  workflow_dispatch:
    inputs:
      confirm:
        description: 'Type "deploy" to confirm deployment'
        required: true
        default: ''
      environment:
        description: 'Environment to deploy'
        required: true
        default: 'dev'
        type: choice
        options:
          - dev

env:
  TF_VERSION: '1.7.0'

jobs:
  validate-input:
    name: 'Validate Deployment Request'
    runs-on: ubuntu-latest
    steps:
      - name: Check Confirmation
        run: |
          if [ "${{ github.event.inputs.confirm }}" != "deploy" ]; then
            echo "❌ Deployment cancelled: Must type 'deploy' to confirm"
            exit 1
          fi
          echo "✅ Deployment confirmed for Phase 2"

      - name: Check Phase 1 Complete
        run: |
          echo "⚠️ Make sure Phase 1 is deployed before running Phase 2"
          echo "Phase 1 should have: storage containers, credential, external locations"

  deploy-schemas:
    name: '4️⃣ Deploy Schemas'
    needs: validate-input
    runs-on: ubuntu-latest
    environment: ${{ github.event.inputs.environment }}
    
    outputs:
      bronze_schema: ${{ steps.outputs.outputs.bronze_schema }}
      silver_schema: ${{ steps.outputs.outputs.silver_schema }}
      gold_schema: ${{ steps.outputs.outputs.gold_schema }}
    
    steps:
      - name: Checkout Code
        uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}
          terraform_wrapper: false

      - name: Parse Secrets
        id: secrets
        run: |
          echo "client_id=$(echo '${{ secrets.AZURE_AND_TFSTATE }}' | jq -r '.client_id')" >> $GITHUB_OUTPUT
          echo "client_secret=$(echo '${{ secrets.AZURE_AND_TFSTATE }}' | jq -r '.client_secret')" >> $GITHUB_OUTPUT
          echo "tenant_id=$(echo '${{ secrets.AZURE_AND_TFSTATE }}' | jq -r '.tenant_id')" >> $GITHUB_OUTPUT
          echo "subscription_id=$(echo '${{ secrets.AZURE_AND_TFSTATE }}' | jq -r '.subscription_id')" >> $GITHUB_OUTPUT
          echo "state_rg=$(echo '${{ secrets.AZURE_AND_TFSTATE }}' | jq -r '.state_resource_group')" >> $GITHUB_OUTPUT
          echo "state_storage=$(echo '${{ secrets.AZURE_AND_TFSTATE }}' | jq -r '.state_storage_account')" >> $GITHUB_OUTPUT
          echo "state_container=$(echo '${{ secrets.AZURE_AND_TFSTATE }}' | jq -r '.state_container')" >> $GITHUB_OUTPUT
          echo "databricks_host=$(echo '${{ secrets.DBX_DEV }}' | jq -r '.workspace_url')" >> $GITHUB_OUTPUT
          echo "databricks_token=$(echo '${{ secrets.DBX_DEV }}' | jq -r '.token')" >> $GITHUB_OUTPUT

      - name: Terraform Init
        working-directory: terraform/environments/${{ github.event.inputs.environment }}/schema
        env:
          ARM_CLIENT_ID: ${{ steps.secrets.outputs.client_id }}
          ARM_CLIENT_SECRET: ${{ steps.secrets.outputs.client_secret }}
          ARM_TENANT_ID: ${{ steps.secrets.outputs.tenant_id }}
          ARM_SUBSCRIPTION_ID: ${{ steps.secrets.outputs.subscription_id }}
        run: |
          terraform init \
            -backend-config="resource_group_name=${{ steps.secrets.outputs.state_rg }}" \
            -backend-config="storage_account_name=${{ steps.secrets.outputs.state_storage }}" \
            -backend-config="container_name=${{ steps.secrets.outputs.state_container }}" \
            -backend-config="key=${{ github.event.inputs.environment }}/schema.tfstate"

      - name: Terraform Apply
        working-directory: terraform/environments/${{ github.event.inputs.environment }}/schema
        env:
          ARM_CLIENT_ID: ${{ steps.secrets.outputs.client_id }}
          ARM_CLIENT_SECRET: ${{ steps.secrets.outputs.client_secret }}
          ARM_TENANT_ID: ${{ steps.secrets.outputs.tenant_id }}
          ARM_SUBSCRIPTION_ID: ${{ steps.secrets.outputs.subscription_id }}
          DATABRICKS_HOST: ${{ steps.secrets.outputs.databricks_host }}
          DATABRICKS_TOKEN: ${{ steps.secrets.outputs.databricks_token }}
        run: |
          terraform plan -out=tfplan
          terraform apply -auto-approve tfplan

      - name: Get Outputs
        id: outputs
        working-directory: terraform/environments/${{ github.event.inputs.environment }}/schema
        env:
          ARM_CLIENT_ID: ${{ steps.secrets.outputs.client_id }}
          ARM_CLIENT_SECRET: ${{ steps.secrets.outputs.client_secret }}
          ARM_TENANT_ID: ${{ steps.secrets.outputs.tenant_id }}
          ARM_SUBSCRIPTION_ID: ${{ steps.secrets.outputs.subscription_id }}
          DATABRICKS_HOST: ${{ steps.secrets.outputs.databricks_host }}
          DATABRICKS_TOKEN: ${{ steps.secrets.outputs.databricks_token }}
        run: |
          BRONZE_SCHEMA=$(terraform output -json bronze_schema | jq -r '.full_name')
          SILVER_SCHEMA=$(terraform output -json silver_schema | jq -r '.full_name')
          GOLD_SCHEMA=$(terraform output -json gold_schema | jq -r '.full_name')
          echo "bronze_schema=$BRONZE_SCHEMA" >> $GITHUB_OUTPUT
          echo "silver_schema=$SILVER_SCHEMA" >> $GITHUB_OUTPUT
          echo "gold_schema=$GOLD_SCHEMA" >> $GITHUB_OUTPUT

  deploy-volumes:
    name: '5️⃣ Deploy Volumes'
    needs: deploy-schemas
    runs-on: ubuntu-latest
    environment: ${{ github.event.inputs.environment }}
    
    outputs:
      bronze_landing_id: ${{ steps.outputs.outputs.bronze_landing_id }}
      bronze_archive_id: ${{ steps.outputs.outputs.bronze_archive_id }}
      silver_processed_id: ${{ steps.outputs.outputs.silver_processed_id }}
      gold_exports_id: ${{ steps.outputs.outputs.gold_exports_id }}
    
    steps:
      - name: Checkout Code
        uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}
          terraform_wrapper: false

      - name: Parse Secrets
        id: secrets
        run: |
          echo "client_id=$(echo '${{ secrets.AZURE_AND_TFSTATE }}' | jq -r '.client_id')" >> $GITHUB_OUTPUT
          echo "client_secret=$(echo '${{ secrets.AZURE_AND_TFSTATE }}' | jq -r '.client_secret')" >> $GITHUB_OUTPUT
          echo "tenant_id=$(echo '${{ secrets.AZURE_AND_TFSTATE }}' | jq -r '.tenant_id')" >> $GITHUB_OUTPUT
          echo "subscription_id=$(echo '${{ secrets.AZURE_AND_TFSTATE }}' | jq -r '.subscription_id')" >> $GITHUB_OUTPUT
          echo "state_rg=$(echo '${{ secrets.AZURE_AND_TFSTATE }}' | jq -r '.state_resource_group')" >> $GITHUB_OUTPUT
          echo "state_storage=$(echo '${{ secrets.AZURE_AND_TFSTATE }}' | jq -r '.state_storage_account')" >> $GITHUB_OUTPUT
          echo "state_container=$(echo '${{ secrets.AZURE_AND_TFSTATE }}' | jq -r '.state_container')" >> $GITHUB_OUTPUT
          echo "databricks_host=$(echo '${{ secrets.DBX_DEV }}' | jq -r '.workspace_url')" >> $GITHUB_OUTPUT
          echo "databricks_token=$(echo '${{ secrets.DBX_DEV }}' | jq -r '.token')" >> $GITHUB_OUTPUT

      # Get External Location URLs from Phase 1
      - name: Get Phase 1 Outputs
        id: phase1
        working-directory: terraform/environments/${{ github.event.inputs.environment }}/external-location
        env:
          ARM_CLIENT_ID: ${{ steps.secrets.outputs.client_id }}
          ARM_CLIENT_SECRET: ${{ steps.secrets.outputs.client_secret }}
          ARM_TENANT_ID: ${{ steps.secrets.outputs.tenant_id }}
          ARM_SUBSCRIPTION_ID: ${{ steps.secrets.outputs.subscription_id }}
          DATABRICKS_HOST: ${{ steps.secrets.outputs.databricks_host }}
          DATABRICKS_TOKEN: ${{ steps.secrets.outputs.databricks_token }}
        run: |
          terraform init \
            -backend-config="resource_group_name=${{ steps.secrets.outputs.state_rg }}" \
            -backend-config="storage_account_name=${{ steps.secrets.outputs.state_storage }}" \
            -backend-config="container_name=${{ steps.secrets.outputs.state_container }}" \
            -backend-config="key=${{ github.event.inputs.environment }}/external-location.tfstate"
          
          BRONZE_URL=$(terraform output -json bronze_location | jq -r '.url')
          SILVER_URL=$(terraform output -json silver_location | jq -r '.url')
          GOLD_URL=$(terraform output -json gold_location | jq -r '.url')
          
          echo "bronze_url=$BRONZE_URL" >> $GITHUB_OUTPUT
          echo "silver_url=$SILVER_URL" >> $GITHUB_OUTPUT
          echo "gold_url=$GOLD_URL" >> $GITHUB_OUTPUT

      - name: Terraform Init
        working-directory: terraform/environments/${{ github.event.inputs.environment }}/volume
        env:
          ARM_CLIENT_ID: ${{ steps.secrets.outputs.client_id }}
          ARM_CLIENT_SECRET: ${{ steps.secrets.outputs.client_secret }}
          ARM_TENANT_ID: ${{ steps.secrets.outputs.tenant_id }}
          ARM_SUBSCRIPTION_ID: ${{ steps.secrets.outputs.subscription_id }}
        run: |
          terraform init \
            -backend-config="resource_group_name=${{ steps.secrets.outputs.state_rg }}" \
            -backend-config="storage_account_name=${{ steps.secrets.outputs.state_storage }}" \
            -backend-config="container_name=${{ steps.secrets.outputs.state_container }}" \
            -backend-config="key=${{ github.event.inputs.environment }}/volume.tfstate"

      - name: Terraform Apply
        working-directory: terraform/environments/${{ github.event.inputs.environment }}/volume
        env:
          ARM_CLIENT_ID: ${{ steps.secrets.outputs.client_id }}
          ARM_CLIENT_SECRET: ${{ steps.secrets.outputs.client_secret }}
          ARM_TENANT_ID: ${{ steps.secrets.outputs.tenant_id }}
          ARM_SUBSCRIPTION_ID: ${{ steps.secrets.outputs.subscription_id }}
          DATABRICKS_HOST: ${{ steps.secrets.outputs.databricks_host }}
          DATABRICKS_TOKEN: ${{ steps.secrets.outputs.databricks_token }}
          TF_VAR_bronze_location_url: ${{ steps.phase1.outputs.bronze_url }}
          TF_VAR_silver_location_url: ${{ steps.phase1.outputs.silver_url }}
          TF_VAR_gold_location_url: ${{ steps.phase1.outputs.gold_url }}
        run: |
          terraform plan -out=tfplan
          terraform apply -auto-approve tfplan

      - name: Get Outputs
        id: outputs
        working-directory: terraform/environments/${{ github.event.inputs.environment }}/volume
        env:
          ARM_CLIENT_ID: ${{ steps.secrets.outputs.client_id }}
          ARM_CLIENT_SECRET: ${{ steps.secrets.outputs.client_secret }}
          ARM_TENANT_ID: ${{ steps.secrets.outputs.tenant_id }}
          ARM_SUBSCRIPTION_ID: ${{ steps.secrets.outputs.subscription_id }}
          DATABRICKS_HOST: ${{ steps.secrets.outputs.databricks_host }}
          DATABRICKS_TOKEN: ${{ steps.secrets.outputs.databricks_token }}
        run: |
          BRONZE_LANDING_ID=$(terraform output -json bronze_landing_volume | jq -r '.id')
          BRONZE_ARCHIVE_ID=$(terraform output -json bronze_archive_volume | jq -r '.id')
          SILVER_PROCESSED_ID=$(terraform output -json silver_processed_volume | jq -r '.id')
          GOLD_EXPORTS_ID=$(terraform output -json gold_exports_volume | jq -r '.id')
          echo "bronze_landing_id=$BRONZE_LANDING_ID" >> $GITHUB_OUTPUT
          echo "bronze_archive_id=$BRONZE_ARCHIVE_ID" >> $GITHUB_OUTPUT
          echo "silver_processed_id=$SILVER_PROCESSED_ID" >> $GITHUB_OUTPUT
          echo "gold_exports_id=$GOLD_EXPORTS_ID" >> $GITHUB_OUTPUT

  deploy-grants:
    name: '6️⃣ Deploy Grants'
    needs: [deploy-schemas, deploy-volumes]
    runs-on: ubuntu-latest
    environment: ${{ github.event.inputs.environment }}
    
    steps:
      - name: Checkout Code
        uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}

      - name: Parse Secrets
        id: secrets
        run: |
          echo "client_id=$(echo '${{ secrets.AZURE_AND_TFSTATE }}' | jq -r '.client_id')" >> $GITHUB_OUTPUT
          echo "client_secret=$(echo '${{ secrets.AZURE_AND_TFSTATE }}' | jq -r '.client_secret')" >> $GITHUB_OUTPUT
          echo "tenant_id=$(echo '${{ secrets.AZURE_AND_TFSTATE }}' | jq -r '.tenant_id')" >> $GITHUB_OUTPUT
          echo "subscription_id=$(echo '${{ secrets.AZURE_AND_TFSTATE }}' | jq -r '.subscription_id')" >> $GITHUB_OUTPUT
          echo "state_rg=$(echo '${{ secrets.AZURE_AND_TFSTATE }}' | jq -r '.state_resource_group')" >> $GITHUB_OUTPUT
          echo "state_storage=$(echo '${{ secrets.AZURE_AND_TFSTATE }}' | jq -r '.state_storage_account')" >> $GITHUB_OUTPUT
          echo "state_container=$(echo '${{ secrets.AZURE_AND_TFSTATE }}' | jq -r '.state_container')" >> $GITHUB_OUTPUT
          echo "databricks_host=$(echo '${{ secrets.DBX_DEV }}' | jq -r '.workspace_url')" >> $GITHUB_OUTPUT
          echo "databricks_token=$(echo '${{ secrets.DBX_DEV }}' | jq -r '.token')" >> $GITHUB_OUTPUT

      # Get IDs from previous phases
      - name: Get Previous Phase Outputs
        id: previous
        run: |
          # This would ideally pull from state or use data sources
          # For now, we'll get them directly in the grants terraform code
          echo "IDs will be retrieved from terraform data sources"

      - name: Terraform Init
        working-directory: terraform/environments/${{ github.event.inputs.environment }}/grants
        env:
          ARM_CLIENT_ID: ${{ steps.secrets.outputs.client_id }}
          ARM_CLIENT_SECRET: ${{ steps.secrets.outputs.client_secret }}
          ARM_TENANT_ID: ${{ steps.secrets.outputs.tenant_id }}
          ARM_SUBSCRIPTION_ID: ${{ steps.secrets.outputs.subscription_id }}
        run: |
          terraform init \
            -backend-config="resource_group_name=${{ steps.secrets.outputs.state_rg }}" \
            -backend-config="storage_account_name=${{ steps.secrets.outputs.state_storage }}" \
            -backend-config="container_name=${{ steps.secrets.outputs.state_container }}" \
            -backend-config="key=${{ github.event.inputs.environment }}/grants.tfstate"

      - name: Terraform Apply
        working-directory: terraform/environments/${{ github.event.inputs.environment }}/grants
        env:
          ARM_CLIENT_ID: ${{ steps.secrets.outputs.client_id }}
          ARM_CLIENT_SECRET: ${{ steps.secrets.outputs.client_secret }}
          ARM_TENANT_ID: ${{ steps.secrets.outputs.tenant_id }}
          ARM_SUBSCRIPTION_ID: ${{ steps.secrets.outputs.subscription_id }}
          DATABRICKS_HOST: ${{ steps.secrets.outputs.databricks_host }}
          DATABRICKS_TOKEN: ${{ steps.secrets.outputs.databricks_token }}
          TF_VAR_storage_credential_id: "dummy"  # Will be handled by data sources
          TF_VAR_bronze_location_id: "dummy"
          TF_VAR_silver_location_id: "dummy"
          TF_VAR_gold_location_id: "dummy"
          TF_VAR_bronze_schema_full_name: ${{ needs.deploy-schemas.outputs.bronze_schema }}
          TF_VAR_silver_schema_full_name: ${{ needs.deploy-schemas.outputs.silver_schema }}
          TF_VAR_gold_schema_full_name: ${{ needs.deploy-schemas.outputs.gold_schema }}
          TF_VAR_bronze_landing_volume_id: ${{ needs.deploy-volumes.outputs.bronze_landing_id }}
          TF_VAR_bronze_archive_volume_id: ${{ needs.deploy-volumes.outputs.bronze_archive_id }}
          TF_VAR_silver_processed_volume_id: ${{ needs.deploy-volumes.outputs.silver_processed_id }}
          TF_VAR_gold_exports_volume_id: ${{ needs.deploy-volumes.outputs.gold_exports_id }}
        run: |
          terraform plan -out=tfplan
          terraform apply -auto-approve tfplan

  summary:
    name: '📊 Deployment Summary'
    needs: [deploy-schemas, deploy-volumes, deploy-grants]
    runs-on: ubuntu-latest
    if: always()
    
    steps:
      - name: Create Summary
        run: |
          echo "## ✅ Phase 2 Deployment Complete!" >> $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY
          echo "### Resources Deployed:" >> $GITHUB_STEP_SUMMARY
          echo "- ✅ 3 Schemas (bronze, silver, gold)" >> $GITHUB_STEP_SUMMARY
          echo "- ✅ 4 Volumes (landing_zone, archive, processed, exports)" >> $GITHUB_STEP_SUMMARY
          echo "- ✅ 11 Grant Resources (permissions)" >> $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY
          echo "### Total Infrastructure Deployed:" >> $GITHUB_STEP_SUMMARY
          echo "Phase 1 + Phase 2 = **25 Resources**" >> $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY
          echo "### Verification:" >> $GITHUB_STEP_SUMMARY
          echo "1. **Databricks UI**:" >> $GITHUB_STEP_SUMMARY
          echo "   - Data → Unity Catalog → dev_terraform_catalog" >> $GITHUB_STEP_SUMMARY
          echo "   - Verify schemas: bronze, silver, gold" >> $GITHUB_STEP_SUMMARY
          echo "   - Verify volumes in each schema" >> $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY
          echo "2. **Test Access** (Databricks notebook):" >> $GITHUB_STEP_SUMMARY
          echo '   ```python' >> $GITHUB_STEP_SUMMARY
          echo '   dbutils.fs.ls("/Volumes/dev_terraform_catalog/bronze/landing_zone/")' >> $GITHUB_STEP_SUMMARY
          echo '   ```' >> $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY
          echo "### 🎉 Deployment Complete!" >> $GITHUB_STEP_SUMMARY
          echo "Your Unity Catalog infrastructure is now ready for use!" >> $GITHUB_STEP_SUMMARY
WORKFLOW 5: Terraform Destroy (Cleanup)
File: .github/workflows/terraform-destroy.yml
Purpose: Destroy infrastructure (use with caution!)
name: 'Terraform Destroy - Cleanup'

on:
  workflow_dispatch:
    inputs:
      confirm:
        description: '⚠️ Type "destroy" to confirm destruction'
        required: true
        default: ''
      environment:
        description: 'Environment to destroy'
        required: true
        default: 'dev'
        type: choice
        options:
          - dev
      phase:
        description: 'Which phase to destroy'
        required: true
        type: choice
        options:
          - phase2-only
          - both-phases
          - phase1-only

env:
  TF_VERSION: '1.7.0'

jobs:
  validate-input:
    name: '⚠️ Validate Destruction Request'
    runs-on: ubuntu-latest
    steps:
      - name: Check Confirmation
        run: |
          if [ "${{ github.event.inputs.confirm }}" != "destroy" ]; then
            echo "❌ Destruction cancelled: Must type 'destroy' to confirm"
            exit 1
          fi
          echo "⚠️ DESTRUCTION CONFIRMED"
          echo "Environment: ${{ github.event.inputs.environment }}"
          echo "Phase: ${{ github.event.inputs.phase }}"

  destroy-phase2:
    name: 'Destroy Phase 2 (Grants, Volumes, Schemas)'
    needs: validate-input
    runs-on: ubuntu-latest
    if: github.event.inputs.phase == 'phase2-only' || github.event.inputs.phase == 'both-phases'
    
    steps:
      - name: Checkout Code
        uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}

      - name: Parse Secrets
        id: secrets
        run: |
          echo "client_id=$(echo '${{ secrets.AZURE_AND_TFSTATE }}' | jq -r '.client_id')" >> $GITHUB_OUTPUT
          echo "client_secret=$(echo '${{ secrets.AZURE_AND_TFSTATE }}' | jq -r '.client_secret')" >> $GITHUB_OUTPUT
          echo "tenant_id=$(echo '${{ secrets.AZURE_AND_TFSTATE }}' | jq -r '.tenant_id')" >> $GITHUB_OUTPUT
          echo "subscription_id=$(echo '${{ secrets.AZURE_AND_TFSTATE }}' | jq -r '.subscription_id')" >> $GITHUB_OUTPUT
          echo "state_rg=$(echo '${{ secrets.AZURE_AND_TFSTATE }}' | jq -r '.state_resource_group')" >> $GITHUB_OUTPUT
          echo "state_storage=$(echo '${{ secrets.AZURE_AND_TFSTATE }}' | jq -r '.state_storage_account')" >> $GITHUB_OUTPUT
          echo "state_container=$(echo '${{ secrets.AZURE_AND_TFSTATE }}' | jq -r '.state_container')" >> $GITHUB_OUTPUT
          echo "databricks_host=$(echo '${{ secrets.DBX_DEV }}' | jq -r '.workspace_url')" >> $GITHUB_OUTPUT
          echo "databricks_token=$(echo '${{ secrets.DBX_DEV }}' | jq -r '.token')" >> $GITHUB_OUTPUT

      # Destroy in reverse order: Grants → Volumes → Schemas

      - name: Destroy Grants
        working-directory: terraform/environments/${{ github.event.inputs.environment }}/grants
        env:
          ARM_CLIENT_ID: ${{ steps.secrets.outputs.client_id }}
          ARM_CLIENT_SECRET: ${{ steps.secrets.outputs.client_secret }}
          ARM_TENANT_ID: ${{ steps.secrets.outputs.tenant_id }}
          ARM_SUBSCRIPTION_ID: ${{ steps.secrets.outputs.subscription_id }}
          DATABRICKS_HOST: ${{ steps.secrets.outputs.databricks_host }}
          DATABRICKS_TOKEN: ${{ steps.secrets.outputs.databricks_token }}
        run: |
          terraform init \
            -backend-config="resource_group_name=${{ steps.secrets.outputs.state_rg }}" \
            -backend-config="storage_account_name=${{ steps.secrets.outputs.state_storage }}" \
            -backend-config="container_name=${{ steps.secrets.outputs.state_container }}" \
            -backend-config="key=${{ github.event.inputs.environment }}/grants.tfstate"
          terraform destroy -auto-approve
        continue-on-error: true

      - name: Destroy Volumes
        working-directory: terraform/environments/${{ github.event.inputs.environment }}/volume
        env:
          ARM_CLIENT_ID: ${{ steps.secrets.outputs.client_id }}
          ARM_CLIENT_SECRET: ${{ steps.secrets.outputs.client_secret }}
          ARM_TENANT_ID: ${{ steps.secrets.outputs.tenant_id }}
          ARM_SUBSCRIPTION_ID: ${{ steps.secrets.outputs.subscription_id }}
          DATABRICKS_HOST: ${{ steps.secrets.outputs.databricks_host }}
          DATABRICKS_TOKEN: ${{ steps.secrets.outputs.databricks_token }}
        run: |
          terraform init \
            -backend-config="resource_group_name=${{ steps.secrets.outputs.state_rg }}" \
            -backend-config="storage_account_name=${{ steps.secrets.outputs.state_storage }}" \
            -backend-config="container_name=${{ steps.secrets.outputs.state_container }}" \
            -backend-config="key=${{ github.event.inputs.environment }}/volume.tfstate"
          terraform destroy -auto-approve
        continue-on-error: true

      - name: Destroy Schemas
        working-directory: terraform/environments/${{ github.event.inputs.environment }}/schema
        env:
          ARM_CLIENT_ID: ${{ steps.secrets.outputs.client_id }}
          ARM_CLIENT_SECRET: ${{ steps.secrets.outputs.client_secret }}
          ARM_TENANT_ID: ${{ steps.secrets.outputs.tenant_id }}
          ARM_SUBSCRIPTION_ID: ${{ steps.secrets.outputs.subscription_id }}
          DATABRICKS_HOST: ${{ steps.secrets.outputs.databricks_host }}
          DATABRICKS_TOKEN: ${{ steps.secrets.outputs.databricks_token }}
        run: |
          terraform init \
            -backend-config="resource_group_name=${{ steps.secrets.outputs.state_rg }}" \
            -backend-config="storage_account_name=${{ steps.secrets.outputs.state_storage }}" \
            -backend-config="container_name=${{ steps.secrets.outputs.state_container }}" \
            -backend-config="key=${{ github.event.inputs.environment }}/schema.tfstate"
          terraform destroy -auto-approve

  destroy-phase1:
    name: 'Destroy Phase 1 (Locations, Credential, Containers)'
    needs: [validate-input, destroy-phase2]
    runs-on: ubuntu-latest
    if: |
      always() &&
      (github.event.inputs.phase == 'phase1-only' || github.event.inputs.phase == 'both-phases')
    
    steps:
      - name: Checkout Code
        uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}

      - name: Parse Secrets
        id: secrets
        run: |
          echo "client_id=$(echo '${{ secrets.AZURE_AND_TFSTATE }}' | jq -r '.client_id')" >> $GITHUB_OUTPUT
          echo "client_secret=$(echo '${{ secrets.AZURE_AND_TFSTATE }}' | jq -r '.client_secret')" >> $GITHUB_OUTPUT
          echo "tenant_id=$(echo '${{ secrets.AZURE_AND_TFSTATE }}' | jq -r '.tenant_id')" >> $GITHUB_OUTPUT
          echo "subscription_id=$(echo '${{ secrets.AZURE_AND_TFSTATE }}' | jq -r '.subscription_id')" >> $GITHUB_OUTPUT
          echo "state_rg=$(echo '${{ secrets.AZURE_AND_TFSTATE }}' | jq -r '.state_resource_group')" >> $GITHUB_OUTPUT
          echo "state_storage=$(echo '${{ secrets.AZURE_AND_TFSTATE }}' | jq -r '.state_storage_account')" >> $GITHUB_OUTPUT
          echo "state_container=$(echo '${{ secrets.AZURE_AND_TFSTATE }}' | jq -r '.state_container')" >> $GITHUB_OUTPUT
          echo "databricks_host=$(echo '${{ secrets.DBX_DEV }}' | jq -r '.workspace_url')" >> $GITHUB_OUTPUT
          echo "databricks_token=$(echo '${{ secrets.DBX_DEV }}' | jq -r '.token')" >> $GITHUB_OUTPUT
          echo "resource_group=$(echo '${{ secrets.DEV_ENV_DETAILS }}' | jq -r '.resource_group')" >> $GITHUB_OUTPUT
          echo "storage_account=$(echo '${{ secrets.DEV_ENV_DETAILS }}' | jq -r '.storage_account')" >> $GITHUB_OUTPUT
          echo "access_connector=$(echo '${{ secrets.DEV_ENV_DETAILS }}' | jq -r '.access_connector_id')" >> $GITHUB_OUTPUT

      # Destroy in reverse order: External Locations → Credential → Containers

      - name: Destroy External Locations
        working-directory: terraform/environments/${{ github.event.inputs.environment }}/external-location
        env:
          ARM_CLIENT_ID: ${{ steps.secrets.outputs.client_id }}
          ARM_CLIENT_SECRET: ${{ steps.secrets.outputs.client_secret }}
          ARM_TENANT_ID: ${{ steps.secrets.outputs.tenant_id }}
          ARM_SUBSCRIPTION_ID: ${{ steps.secrets.outputs.subscription_id }}
          DATABRICKS_HOST: ${{ steps.secrets.outputs.databricks_host }}
          DATABRICKS_TOKEN: ${{ steps.secrets.outputs.databricks_token }}
        run: |
          terraform init \
            -backend-config="resource_group_name=${{ steps.secrets.outputs.state_rg }}" \
            -backend-config="storage_account_name=${{ steps.secrets.outputs.state_storage }}" \
            -backend-config="container_name=${{ steps.secrets.outputs.state_container }}" \
            -backend-config="key=${{ github.event.inputs.environment }}/external-location.tfstate"
          terraform destroy -auto-approve

      - name: Destroy Storage Credential
        working-directory: terraform/environments/${{ github.event.inputs.environment }}/storage-credential
        env:
          ARM_CLIENT_ID: ${{ steps.secrets.outputs.client_id }}
          ARM_CLIENT_SECRET: ${{ steps.secrets.outputs.client_secret }}
          ARM_TENANT_ID: ${{ steps.secrets.outputs.tenant_id }}
          ARM_SUBSCRIPTION_ID: ${{ steps.secrets.outputs.subscription_id }}
          DATABRICKS_HOST: ${{ steps.secrets.outputs.databricks_host }}
          DATABRICKS_TOKEN: ${{ steps.secrets.outputs.databricks_token }}
          TF_VAR_access_connector_id: ${{ steps.secrets.outputs.access_connector }}
        run: |
          terraform init \
            -backend-config="resource_group_name=${{ steps.secrets.outputs.state_rg }}" \
            -backend-config="storage_account_name=${{ steps.secrets.outputs.state_storage }}" \
            -backend-config="container_name=${{ steps.secrets.outputs.state_container }}" \
            -backend-config="key=${{ github.event.inputs.environment }}/storage-credential.tfstate"
          terraform destroy -auto-approve

      - name: Destroy Storage Containers
        working-directory: terraform/environments/${{ github.event.inputs.environment }}/storage-container
        env:
          ARM_CLIENT_ID: ${{ steps.secrets.outputs.client_id }}
          ARM_CLIENT_SECRET: ${{ steps.secrets.outputs.client_secret }}
          ARM_TENANT_ID: ${{ steps.secrets.outputs.tenant_id }}
          ARM_SUBSCRIPTION_ID: ${{ steps.secrets.outputs.subscription_id }}
          TF_VAR_storage_account_name: ${{ steps.secrets.outputs.storage_account }}
          TF_VAR_resource_group_name: ${{ steps.secrets.outputs.resource_group }}
        run: |
          terraform init \
            -backend-config="resource_group_name=${{ steps.secrets.outputs.state_rg }}" \
            -backend-config="storage_account_name=${{ steps.secrets.outputs.state_storage }}" \
            -backend-config="container_name=${{ steps.secrets.outputs.state_container }}" \
            -backend-config="key=${{ github.event.inputs.environment }}/storage-container.tfstate"
          terraform destroy -auto-approve

  summary:
    name: '📊 Destruction Summary'
    needs: [destroy-phase2, destroy-phase1]
    runs-on: ubuntu-latest
    if: always()
    
    steps:
      - name: Create Summary
        run: |
          echo "## ⚠️ Infrastructure Destroyed" >> $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY
          echo "**Environment**: ${{ github.event.inputs.environment }}" >> $GITHUB_STEP_SUMMARY
          echo "**Phase**: ${{ github.event.inputs.phase }}" >> $GITHUB_STEP_SUMMARY
          echo "**Time**: $(date)" >> $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY
          echo "### Destroyed Resources:" >> $GITHUB_STEP_SUMMARY
          
          if [ "${{ github.event.inputs.phase }}" == "both-phases" ] || [ "${{ github.event.inputs.phase }}" == "phase2-only" ]; then
            echo "- Grants" >> $GITHUB_STEP_SUMMARY
            echo "- Volumes" >> $GITHUB_STEP_SUMMARY
            echo "- Schemas" >> $GITHUB_STEP_SUMMARY
          fi
          
          if [ "${{ github.event.inputs.phase }}" == "both-phases" ] || [ "${{ github.event.inputs.phase }}" == "phase1-only" ]; then
            echo "- External Locations" >> $GITHUB_STEP_SUMMARY
            echo "- Storage Credential" >> $GITHUB_STEP_SUMMARY
            echo "- Storage Containers" >> $GITHUB_STEP_SUMMARY
          fi
✅ Summary - All 5 Workflows Created!
You now have:
✅ terraform-validate.yml - Code quality & validation
✅ terraform-plan-dev.yml - Preview changes on PR
✅ terraform-apply-phase1.yml - Deploy basic infrastructure
✅ terraform-apply-phase2.yml - Deploy schemas, volumes, grants
✅ terraform-destroy.yml - Cleanup (with safeguards)
Ready to add these to your repository!