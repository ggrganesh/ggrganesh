---
name: CD Terraform Standalone

on:
  workflow_call:
    inputs:
      org:
        description: "Invoking org"
        required: false
        type: string
        default: "None"
      repo:
        description: "Invoking repo"
        required: false
        type: string
        default: "None"
      check_run_id:
        description: "Check run id to update post job completion"
        required: false
        type: string
        default: "None"
      issue-data:
        type: string
        required: false
        description: JSON Data

jobs:
  workflow-bootstrapping:
    if: ${{ contains(github.event.issue.labels.*.name, 'cd-terraform-standalone') || inputs.issue-data}}
    uses: ./.github/workflows/runner-resolver.yml
    secrets: inherit
    with:
      workflow-names: "cd-input-handler, change-orchestrator, cd-terraform-standalone"

  handle-input:
    needs: workflow-bootstrapping
    uses: ./.github/workflows/cd-input-handler.yml
    secrets: inherit
    with:
      issue-data: ${{ inputs.issue-data }}
      is-tf-pipeline: true
      is-tf-standalone: true
      runner-label: ${{ fromJson(needs.workflow-bootstrapping.outputs.labels).cd-input-handler }}

  change-orchestrator:
    if: ${{ needs.handle-input.outputs.is-new-change-request  == 'true' && needs.handle-input.outputs.environment-type == 'change-control' }}
    name: Create CHG Req (Common for all environments)
    uses: ./.github/workflows/change-orchestrator.yml
    needs: [workflow-bootstrapping, handle-input]
    secrets: inherit
    with:
      target-environment: ${{ needs.handle-input.outputs.environment-matrix }}
      is-existing-ticket-number: ${{ needs.handle-input.outputs.is-existing-ticket-number }}
      existing-ticket-number: ${{ needs.handle-input.outputs.existing-ticket-number }}
      is-new-change-request: ${{ needs.handle-input.outputs.is-new-change-request }}
      start-date: ${{ needs.handle-input.outputs.start-date }}
      end-date: ${{ needs.handle-input.outputs.end-date }}
      cmdb-name: ${{ needs.handle-input.outputs.cmdb-name }}
      assignment-group: ${{ needs.handle-input.outputs.assignment-group }}
      description: ${{ needs.handle-input.outputs.description }}
      short-description: ${{ needs.handle-input.outputs.short-description }}
      requestor: ${{ needs.handle-input.outputs.requestor-email }}
      workflow-run-id: ${{ github.run_id }}
      repository-name: ${{ github.repository }}
      git-url: ${{ github.event.repository.html_url }}
      issue-body: ${{ needs.handle-input.outputs.issue-details-json }}
      deployment-type: "cd-terraform-standalone"
      is-exempted: ${{ needs.handle-input.outputs.is-exempted }}
      runner-label: ${{ fromJson(needs.workflow-bootstrapping.outputs.labels).change-orchestrator }}

  deploy-terraform-standalone:
    name: ${{ matrix.environment }} - Deployment
    needs: [handle-input, workflow-bootstrapping]
    strategy:
      fail-fast: ${{ fromJson(needs.handle-input.outputs.is-fail-fast) }}
      max-parallel: ${{ fromJson(needs.handle-input.outputs.is-parallel) }}
      matrix:
        environment: ${{ fromJson(needs.handle-input.outputs.environment-matrix) }}
    uses: ./.github/workflows/cd-terraform-standalone-reusable.yml
    secrets: inherit
    with:
      issue-details-json: "${{ needs.handle-input.outputs.issue-details-json != '{}' && needs.handle-input.outputs.issue-details-json || inputs.issue-data }}"
      environment: ${{ matrix.environment }}
      is-existing-ticket-number: ${{ needs.handle-input.outputs.is-existing-ticket-number }}
      existing-ticket-number: ${{ needs.handle-input.outputs.existing-ticket-number }}
      environment-type: ${{ needs.handle-input.outputs.environment-type }}
      org: ${{ inputs.org }}
      repo: ${{ inputs.repo }}
      check_run_id: ${{ inputs.check_run_id }}
      runner-label: ${{ fromJson(needs.workflow-bootstrapping.outputs.labels).cd-terraform-standalone }}
      chg-orchestrator-runner-label: ${{ fromJson(needs.workflow-bootstrapping.outputs.labels).change-orchestrator }}
