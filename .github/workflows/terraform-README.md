# Terraform Template Pipeline

This repository provides reusable GitHub Actions workflows for managing Terraform infrastructure on Azure. It includes workflows for creating and destroying resources with proper validation and planning steps.

## Workflows

### 1. Terraform Creation Workflow
**File**: [.github/workflows/terraform-template-creation-resource.yml](.github/workflows/terraform-template-creation-resource.yml)

This workflow performs the complete Terraform lifecycle:
- **Validate**: Checks Terraform configuration syntax and formatting
- **Plan**: Creates an execution plan showing what changes will be made
- **Apply**: Applies the changes to create/update infrastructure (only on main branch pushes)
- **Dynamic Variable Mapping**: Maps GitHub secrets to Terraform variables based on configuration

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
    with:
      terraform_var_mapping: |
        {
          "environment": "ENVIRONMENT",
          "region": "AZURE_REGION",
          "instance_count": "INSTANCE_COUNT",
          "database_password": "DB_PASSWORD",
          "api_key": "API_SECRET_KEY"
        }
    secrets: inherit
```

### Example: Using the Destroy Workflow

```yaml
name: Destroy Infrastructure
on:
  workflow_dispatch: # Manual trigger

jobs:
  destroy:
    uses: Grupo-118-Tech-Challenge-Fiap-11SOAT/terraform-template-pipeline-grupo118-fase-3/.github/workflows/terraform-template-destroy-resource.yml@main
    with:
      terraform_var_mapping: |
        {
          "environment": "ENVIRONMENT",
          "region": "AZURE_REGION",
          "instance_count": "INSTANCE_COUNT",
          "database_password": "DB_PASSWORD",
          "api_key": "API_SECRET_KEY"
        }
    secrets: inherit
```

## Required Inputs

### Creation Workflow
- `terraform_var_mapping` (required): JSON mapping of Terraform variable names to GitHub secret names

**Example mapping:**
```json
{
  "terraform_variable_name": "GITHUB_SECRET_NAME",
  "environment": "ENVIRONMENT",
  "database_url": "DATABASE_CONNECTION_STRING"
}
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

### Custom Secrets
Any additional secrets referenced in your `terraform_var_mapping` should be configured in your repository settings.

## Prerequisites

1. **Azure Service Principal**: Create a service principal with appropriate permissions for your Azure resources
2. **Terraform Configuration**: Your repository should contain valid Terraform configuration files
3. **GitHub Secrets**: Configure the required secrets in your repository settings
4. **Variable Mapping**: Define the mapping between Terraform variables and GitHub secrets

## Workflow Behavior

### Creation Workflow
- Runs on any branch for validation and planning
- Only applies changes when pushing to the `main` branch
- Uses `terraform fmt -check` to ensure code formatting
- Dynamically generates `terraform.tfvars` from secret mappings
- Continues on error during planning to show results
- Secrets are inherited from the calling workflow using `secrets: inherit`

### Destroy Workflow
- Only runs on the `main` branch
- Requires manual trigger or specific conditions
- Automatically approves destruction without user input
- Inherits all secrets from the calling workflow

## Dynamic Variable Mapping

The creation workflow uses a JSON mapping to dynamically assign GitHub secrets to Terraform variables:

1. **Define the mapping** in the `terraform_var_mapping` input
2. **Configure secrets** in your repository that match the mapping
3. **The workflow automatically**:
   - Reads the mapping configuration
   - Retrieves corresponding secret values
   - Generates a `terraform.tfvars` file
   - Passes it to Terraform commands

**Benefits:**
- No hardcoded secret names in the template
- Flexible variable assignment per project
- Secure handling of sensitive values
- Automatic validation of secret availability

## Security Notes

- Both workflows run in the `production` environment
- Azure credentials are passed as environment variables with ARM_ prefix
- All Terraform commands run with `-input=false` to prevent interactive prompts
- Auto-approval is enabled for apply and destroy operations
- Secret values are redacted in logs for security
- Uses `secrets: inherit` to securely pass all secrets to reusable workflows

## Troubleshooting

### Common Issues

1. **Missing Secret Warning**: If you see warnings about missing secrets, ensure all secrets referenced in `terraform_var_mapping` exist in your repository settings.

2. **Empty tfvars**: If no variables are generated, verify your JSON mapping syntax and secret names.

3. **Permission Issues**: Ensure your Azure service principal has sufficient permissions for the resources you're trying to create.

## Contributing

1. Ensure your Terraform code is properly formatted (`terraform fmt`)
2. Test workflows in a separate branch before merging to main
3. Review the execution plan before applying changes to production resources
4. Validate your `terraform_var_mapping` JSON syntax before committing