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
    uses: Toyota-Motor-North-America/chofer-actions/.github/workflows/cd-terraform-standalone.yml@v1
    secrets: inherit
    with:
      issue-data: ${{ inputs.deploymentdata }}
