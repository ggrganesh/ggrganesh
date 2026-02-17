name: 'Deploy Schema and Grants'

on:
  workflow_dispatch:
    inputs:
      confirm:
        description: 'Type "deploy" to confirm'
        required: true
        default: ''

env:
  TF_VERSION: '1.7.0'

jobs:

  deploy-schema:
    name: '1️⃣ Create Schema'
    runs-on: ubuntu-latest
    if: github.event.inputs.confirm == 'deploy'

    steps:
      - name: Checkout Code
        uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}
          terraform_wrapper: false

      - name: Parse Databricks Credentials
        id: secrets
        run: |
          echo "databricks_host=$(echo '${{ secrets.DBX_DEV }}' | jq -r '.workspace_url')" >> $GITHUB_OUTPUT
          echo "databricks_token=$(echo '${{ secrets.DBX_DEV }}' | jq -r '.token')" >> $GITHUB_OUTPUT

      - name: Terraform Init
        working-directory: terraform/environments/dev/schema
        run: terraform init -backend=false

      - name: Terraform Plan
        working-directory: terraform/environments/dev/schema
        env:
          DATABRICKS_HOST: ${{ steps.secrets.outputs.databricks_host }}
          DATABRICKS_TOKEN: ${{ steps.secrets.outputs.databricks_token }}
        run: terraform plan -out=tfplan

      - name: Terraform Apply
        working-directory: terraform/environments/dev/schema
        env:
          DATABRICKS_HOST: ${{ steps.secrets.outputs.databricks_host }}
          DATABRICKS_TOKEN: ${{ steps.secrets.outputs.databricks_token }}
        run: terraform apply -auto-approve tfplan

      - name: Show Created Schema
        working-directory: terraform/environments/dev/schema
        env:
          DATABRICKS_HOST: ${{ steps.secrets.outputs.databricks_host }}
          DATABRICKS_TOKEN: ${{ steps.secrets.outputs.databricks_token }}
        run: |
          echo "✅ Schema created successfully!"
          terraform output

  deploy-grants:
    name: '2️⃣ Apply Grants'
    needs: deploy-schema
    runs-on: ubuntu-latest

    steps:
      - name: Checkout Code
        uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}

      - name: Parse Databricks Credentials
        id: secrets
        run: |
          echo "databricks_host=$(echo '${{ secrets.DBX_DEV }}' | jq -r '.workspace_url')" >> $GITHUB_OUTPUT
          echo "databricks_token=$(echo '${{ secrets.DBX_DEV }}' | jq -r '.token')" >> $GITHUB_OUTPUT

      - name: Terraform Init
        working-directory: terraform/environments/dev/grants
        run: terraform init -backend=false

      - name: Terraform Plan
        working-directory: terraform/environments/dev/grants
        env:
          DATABRICKS_HOST: ${{ steps.secrets.outputs.databricks_host }}
          DATABRICKS_TOKEN: ${{ steps.secrets.outputs.databricks_token }}
        run: terraform plan -out=tfplan

      - name: Terraform Apply
        working-directory: terraform/environments/dev/grants
        env:
          DATABRICKS_HOST: ${{ steps.secrets.outputs.databricks_host }}
          DATABRICKS_TOKEN: ${{ steps.secrets.outputs.databricks_token }}
        run: terraform apply -auto-approve tfplan

      - name: Show Applied Grants
        working-directory: terraform/environments/dev/grants
        env:
          DATABRICKS_HOST: ${{ steps.secrets.outputs.databricks_host }}
          DATABRICKS_TOKEN: ${{ steps.secrets.outputs.databricks_token }}
        run: |
          echo "✅ Grants applied successfully!"
          terraform output

  summary:
    name: '📊 Deployment Summary'
    needs: [deploy-schema, deploy-grants]
    runs-on: ubuntu-latest
    if: always()
    steps:
      - name: Create Summary
        run: |
          echo "## ✅ Schema + Grants Deployed!" >> $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY
          echo "### What was created:" >> $GITHUB_STEP_SUMMARY
          echo "- ✅ Schema: bronze" >> $GITHUB_STEP_SUMMARY
          echo "- ✅ Grants: USE_SCHEMA, SELECT, MODIFY, CREATE_TABLE" >> $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY
          echo "### Verify in Databricks UI:" >> $GITHUB_STEP_SUMMARY
          echo "1. Open Databricks workspace" >> $GITHUB_STEP_SUMMARY
          echo "2. Click **Data** (left sidebar)" >> $GITHUB_STEP_SUMMARY
          echo "3. Click **your catalog name**" >> $GITHUB_STEP_SUMMARY
          echo "4. You should see **bronze** schema!" >> $GITHUB_STEP_SUMMARY
          echo "5. Click schema → **Permissions** tab to verify grants" >> $GITHUB_STEP_SUMMARY