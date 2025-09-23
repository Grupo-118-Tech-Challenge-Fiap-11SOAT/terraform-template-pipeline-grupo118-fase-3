# Terraform Template Pipeline

This repository provides reusable GitHub Actions workflows for managing Terraform infrastructure on Azure. It includes workflows for creating and destroying resources with proper validation and planning steps.

## Workflows

### 1. Terraform Creation Workflow
**File**: [.github/workflows/terraform-template-creation-resource.yml](.github/workflows/terraform-template-creation-resource.yml)

This workflow performs the complete Terraform lifecycle:
- **Validate**: Checks Terraform configuration syntax and formatting
- **Plan**: Creates an execution plan showing what changes will be made
- **Apply**: Applies the changes to create/update infrastructure (only on main branch pushes)

### 2. Terraform Destroy Workflow
**File**: [.github/workflows/terraform-template-destroy-resource.yml](.github/workflows/terraform-template-destroy-resource.yml)

This workflow destroys all resources defined in the Terraform configuration (only runs on main branch).

## Usage

To use these reusable workflows in your repository, create workflow files that call them:

### Example: Using the Creation Workflow

```yaml
name: Deploy Infrastructure
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  deploy:
    uses: Grupo-118-Tech-Challenge-Fiap-11SOAT/terraform-template-pipeline-grupo118-fase-3/.github/workflows/terraform-template-creation-resource.yml@main
    secrets:
      AZURE_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID }}
      AZURE_CLIENT_SECRET: ${{ secrets.AZURE_CLIENT_SECRET }}
      AZURE_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
      AZURE_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }}
      TF_API_TOKEN: ${{ secrets.TF_API_TOKEN }}
```

### Example: Using the Destroy Workflow

```yaml
name: Destroy Infrastructure
on:
  workflow_dispatch: # Manual trigger

jobs:
  destroy:
    uses: Grupo-118-Tech-Challenge-Fiap-11SOAT/terraform-template-pipeline-grupo118-fase-3/.github/workflows/terraform-template-destroy-resource.yml@main
    secrets:
      AZURE_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID }}
      AZURE_CLIENT_SECRET: ${{ secrets.AZURE_CLIENT_SECRET }}
      AZURE_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
      AZURE_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }}
      TF_API_TOKEN: ${{ secrets.TF_API_TOKEN }}
```

## Required Secrets

Both workflows require the following secrets to be configured in your repository:

### Azure Authentication (Required)
- `AZURE_CLIENT_ID` - Azure service principal client ID
- `AZURE_CLIENT_SECRET` - Azure service principal client secret
- `AZURE_SUBSCRIPTION_ID` - Azure subscription ID
- `AZURE_TENANT_ID` - Azure tenant ID

### Terraform Cloud (Optional)
- `TF_API_TOKEN` - Terraform Cloud API token (only required if using Terraform Cloud)

## Prerequisites

1. **Azure Service Principal**: Create a service principal with appropriate permissions for your Azure resources
2. **Terraform Configuration**: Your repository should contain valid Terraform configuration files
3. **GitHub Secrets**: Configure the required secrets in your repository settings

## Workflow Behavior

### Creation Workflow
- Runs on any branch for validation and planning
- Only applies changes when pushing to the `main` branch
- Uses `terraform fmt -check` to ensure code formatting
- Continues on error during planning to show results

### Destroy Workflow
- Only runs on the `main` branch
- Requires manual trigger or specific conditions
- Automatically approves destruction without user input

## Security Notes

- Both workflows run in the `production` environment
- Azure credentials are passed as environment variables with ARM_ prefix
- All Terraform commands run with `-input=false` to prevent interactive prompts
- Auto-approval is enabled for apply and destroy operations

## Contributing

1. Ensure your Terraform code is properly formatted (`terraform fmt`)
2. Test workflows in a separate branch before merging to main
3. Review the execution plan before applying changes to production resources