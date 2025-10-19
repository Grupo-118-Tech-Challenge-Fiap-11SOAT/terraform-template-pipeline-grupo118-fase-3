# DotNet CD Pipeline - Reusable Workflow

This reusable workflow provides a standardized way to deploy .NET applications to Azure Kubernetes Service (AKS) using Helm charts.

## Overview

The workflow automates the deployment process by:
- Authenticating with Azure
- Connecting to AKS cluster
- Processing Helm values files with dynamic secret replacement
- Deploying applications using Helm charts from Azure Container Registry

## Usage

### Prerequisites

1. **Azure Resources**:
   - Azure Container Registry (ACR)
   - Azure Kubernetes Service (AKS) cluster
   - Service Principal with appropriate permissions

2. **Required Secrets in Consumer Repository**:
   - `AZURE_CREDENTIALS` - Azure service principal credentials (JSON format)
   - `AZURE_RESOURCE_GROUP` - Azure resource group name
   - `AKS_CLUSTER_NAME` - AKS cluster name
   - `DOCKER_REGISTRY` - ACR registry URL (e.g., `myregistry.azurecr.io`)
   - `ACR_USERNAME` - ACR username
   - `ACR_PASSWORD` - ACR password
   - Additional secrets as needed by your application

### Consumer Repository Setup

#### 1. Create Helm Values File

Create a `values-production.yaml` file in your repository with variable placeholders using `${VAR_NAME}` syntax:

```yaml
# helm/values-production.yaml
replicaCount: 3

image:
  repository: ${ACR_REGISTRY}/${IMAGE_NAME}
  tag: ${IMAGE_TAG}
  pullPolicy: Always

service:
  type: LoadBalancer
  port: 80
  targetPort: 8080

resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 250m
    memory: 256Mi

env:
  - name: DATABASE_HOST
    value: "${DATABASE_HOST}"
  - name: DATABASE_PORT
    value: "${DATABASE_PORT}"
  - name: DATABASE_NAME
    value: "${DATABASE_NAME}"
  - name: API_KEY
    value: "${API_KEY}"
  - name: ASPNETCORE_ENVIRONMENT
    value: "Production"
```

#### 2. Create Workflow File

Create a workflow in your consumer repository:

```yaml
# .github/workflows/deploy-production.yml
name: Deploy to Production

on:
  push:
    branches: [main]
  workflow_dispatch:

jobs:
  deploy:
    uses: your-org/terraform-template-pipeline-grupo118-fase-3/.github/workflows/dotnet-cd-template.yml@main
    secrets: inherit
    with:
      helm_chart_version: "1.0.0"
      helm_chart_name: "my-app"
      helm_values_file: "helm/values-production.yaml"
      secrets_mapping: |
        {
          "ACR_REGISTRY": "${{ secrets.ACR_REGISTRY }}",
          "IMAGE_NAME": "${{ secrets.IMAGE_NAME }}",
          "IMAGE_TAG": "${{ github.sha }}",
          "DATABASE_HOST": "${{ secrets.DATABASE_HOST }}",
          "DATABASE_PORT": "${{ secrets.DATABASE_PORT }}",
          "DATABASE_NAME": "${{ secrets.DATABASE_NAME }}",
          "API_KEY": "${{ secrets.API_KEY }}"
        }
```

## Workflow Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `helm_chart_version` | Yes | - | Version of the Helm chart to deploy |
| `helm_chart_name` | Yes | - | Name of the Helm chart |
| `helm_values_file` | No | `values-production.yaml` | Path to the Helm values file in your repository |
| `secrets_mapping` | Yes | - | JSON object mapping variable names to secret values |

## Secrets Mapping

The `secrets_mapping` input is a JSON object that maps environment variable names to their values. These variables will be replaced in your Helm values file.

**Format:**
```json
{
  "VAR_NAME": "${{ secrets.SECRET_NAME }}",
  "ANOTHER_VAR": "${{ github.sha }}",
  "STATIC_VAR": "some-static-value"
}
```

**Example:**
```json
{
  "ACR_REGISTRY": "${{ secrets.ACR_REGISTRY }}",
  "IMAGE_NAME": "${{ secrets.IMAGE_NAME }}",
  "IMAGE_TAG": "${{ github.sha }}",
  "DATABASE_HOST": "${{ secrets.DATABASE_HOST }}",
  "DATABASE_PORT": "${{ secrets.DATABASE_PORT }}",
  "API_KEY": "${{ secrets.API_KEY }}"
}
```

## Variable Replacement

The workflow uses `envsubst` to replace variables in your Helm values file. Variables should be defined using the `${VAR_NAME}` syntax.

**Supported formats:**
- `${VAR_NAME}` - Simple variable replacement
- `${VAR_NAME:-default}` - Variable with default value if not set
- `"${VAR_NAME}"` - Quoted variables (recommended for string values)

## Examples

### Basic Deployment

```yaml
jobs:
  deploy:
    uses: your-org/terraform-template-pipeline-grupo118-fase-3/.github/workflows/dotnet-cd-template.yml@main
    secrets: inherit
    with:
      helm_chart_version: "1.0.0"
      helm_chart_name: "my-dotnet-api"
      secrets_mapping: |
        {
          "ACR_REGISTRY": "${{ secrets.ACR_REGISTRY }}",
          "IMAGE_NAME": "my-api",
          "IMAGE_TAG": "${{ github.sha }}"
        }
```

### Deployment with Multiple Environments

```yaml
jobs:
  deploy-staging:
    uses: your-org/terraform-template-pipeline-grupo118-fase-3/.github/workflows/dotnet-cd-template.yml@main
    secrets: inherit
    with:
      helm_chart_version: "1.0.0"
      helm_chart_name: "my-app"
      helm_values_file: "helm/values-staging.yaml"
      secrets_mapping: |
        {
          "ACR_REGISTRY": "${{ secrets.ACR_REGISTRY }}",
          "IMAGE_TAG": "${{ github.sha }}",
          "DATABASE_HOST": "${{ secrets.STAGING_DB_HOST }}",
          "API_KEY": "${{ secrets.STAGING_API_KEY }}"
        }

  deploy-production:
    needs: deploy-staging
    uses: your-org/terraform-template-pipeline-grupo118-fase-3/.github/workflows/dotnet-cd-template.yml@main
    secrets: inherit
    with:
      helm_chart_version: "1.0.0"
      helm_chart_name: "my-app"
      helm_values_file: "helm/values-production.yaml"
      secrets_mapping: |
        {
          "ACR_REGISTRY": "${{ secrets.ACR_REGISTRY }}",
          "IMAGE_TAG": "${{ github.sha }}",
          "DATABASE_HOST": "${{ secrets.PROD_DB_HOST }}",
          "API_KEY": "${{ secrets.PROD_API_KEY }}"
        }
```

### Using Computed Values

```yaml
jobs:
  deploy:
    uses: your-org/terraform-template-pipeline-grupo118-fase-3/.github/workflows/dotnet-cd-template.yml@main
    secrets: inherit
    with:
      helm_chart_version: "1.0.0"
      helm_chart_name: "my-app"
      secrets_mapping: |
        {
          "IMAGE_TAG": "${{ github.sha }}",
          "BUILD_NUMBER": "${{ github.run_number }}",
          "BRANCH_NAME": "${{ github.ref_name }}",
          "COMMIT_MESSAGE": "${{ github.event.head_commit.message }}"
        }
```

## Workflow Steps

1. **Checkout Code** - Checks out the consumer repository
2. **Azure Login** - Authenticates with Azure using service principal
3. **Setup kubectl** - Installs kubectl CLI
4. **Setup Helm** - Installs Helm v3.12.0
5. **Get AKS Credentials** - Retrieves AKS cluster credentials
6. **Process Helm Values** - Replaces variables in values file with mapped secrets
7. **Login to ACR** - Authenticates with Azure Container Registry
8. **Deploy to Kubernetes** - Deploys the application using Helm
9. **Cleanup** - Removes temporary processed files

## Troubleshooting

### Common Issues

**1. Values file not found**
```
Error: Values file helm/values-production.yaml not found
```
- Ensure the `helm_values_file` path is correct
- Verify the file exists in your repository

**2. Variable not replaced**
- Check that the variable name in `secrets_mapping` matches the placeholder in your values file
- Ensure you're using the correct syntax: `${VAR_NAME}`

**3. Secret not accessible**
```
Error: Secret SOME_SECRET is not available
```
- Verify the secret exists in your repository settings
- Ensure `secrets: inherit` is set in the calling workflow

**4. Helm deployment failed**
- Check the Helm chart version exists in your ACR
- Verify ACR credentials are correct
- Review the processed values file output in the workflow logs

## Security Best Practices

1. **Never commit secrets** - Always use GitHub Secrets
2. **Use least privilege** - Service principals should have minimal required permissions
3. **Rotate credentials** - Regularly update ACR passwords and service principal credentials
4. **Review processed values** - Check workflow logs to ensure secrets are properly replaced
5. **Use separate environments** - Different secrets for staging and production

## Support

For issues or questions:
1. Check the workflow run logs for detailed error messages
2. Review the processed values file output
3. Verify all required secrets are configured
4. Ensure Helm chart is available in ACR

## Related Documentation

- [Azure Kubernetes Service](https://docs.microsoft.com/azure/aks/)
- [Helm Documentation](https://helm.sh/docs/)
- [GitHub Actions Reusable Workflows](https://docs.github.com/en/actions/using-workflows/reusing-workflows)
- [GitHub Secrets](https://docs.github.com/en/actions/security-guides/encrypted-secrets)