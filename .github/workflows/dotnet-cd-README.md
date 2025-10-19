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

2. **Required Secrets in Template Repository** (this repository):
   - `AZURE_CREDENTIALS` - Azure service principal credentials (JSON format)
   - `AZURE_RESOURCE_GROUP` - Azure resource group name
   - `AKS_CLUSTER_NAME` - AKS cluster name
   - `DOCKER_REGISTRY` - ACR registry URL (e.g., `myregistry.azurecr.io`)
   - `ACR_USERNAME` - ACR username
   - `ACR_PASSWORD` - ACR password

3. **Application-Specific Secrets in Consumer Repository**:
   - Database credentials
   - API keys
   - Connection strings
   - Any other application-specific secrets

### Consumer Repository Setup

#### 1. Create Helm Values File

Create a `values-production.yaml` file in your repository with variable placeholders using `${VAR_NAME}` syntax:

```yaml
# helm/values-production.yaml
replicaCount: ${POC_REPLICA_COUNT}

image:
  repository: ${DOCKER_REGISTRY}/${POC_IMAGE_NAME}
  tag: ${POC_IMAGE_TAG}
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
  - name: ENVIRONMENT
    value: "${POC_ENVIRONMENT}"
  - name: DATABASE_HOST
    value: "${POC_DATABASE_HOST}"
  - name: DATABASE_PORT
    value: "${POC_DATABASE_PORT}"
  - name: DATABASE_NAME
    value: "${POC_DATABASE_NAME}"
  - name: DATABASE_USER
    value: "${POC_DATABASE_USER}"
  - name: DATABASE_PASSWORD
    value: "${POC_DATABASE_PASSWORD}"
  - name: API_KEY
    value: "${POC_API_KEY}"
  - name: ASPNETCORE_ENVIRONMENT
    value: "Production"
```

#### 2. Create Workflow File

Create a workflow in your consumer repository using **two separate mappings**:

```yaml
# .github/workflows/deploy-production.yml
name: Deploy to Production

on:
  push:
    branches: [main]
  workflow_dispatch:
    inputs:
      image_tag:
        description: 'Image tag to deploy'
        required: true
        type: string
        default: 'latest'

jobs:
  deploy:
    uses: Grupo-118-Tech-Challenge-Fiap-11SOAT/terraform-template-pipeline-grupo118-fase-3/.github/workflows/dotnet-cd-template.yml@main
    with:
      helm_chart_version: "1.0.0"
      helm_chart_name: "my-app"
      helm_values_file: "helm/values-production.yaml"
      
      # Non-secret variables (visible in logs for debugging)
      variables_mapping: |
        {
          "POC_IMAGE_TAG": "${{ github.event.inputs.image_tag || github.sha }}",
          "POC_IMAGE_NAME": "my-app",
          "POC_ENVIRONMENT": "production",
          "POC_REPLICA_COUNT": "3"
        }
      
      # Secret mappings (variable name → secret name to lookup)
      secrets_mapping: |
        {
          "DOCKER_REGISTRY": "DOCKER_REGISTRY",
          "ACR_USERNAME": "ACR_USERNAME",
          "ACR_PASSWORD": "ACR_PASSWORD",
          "POC_DATABASE_HOST": "GS_POC_DATABASE_HOST",
          "POC_DATABASE_PORT": "GS_POC_DATABASE_PORT",
          "POC_DATABASE_NAME": "GS_POC_DATABASE_NAME",
          "POC_DATABASE_USER": "GS_POC_DATABASE_USER",
          "POC_DATABASE_PASSWORD": "GS_POC_DATABASE_PASSWORD",
          "POC_API_KEY": "GS_POC_API_KEY"
        }
```

## Workflow Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `helm_chart_version` | Yes | - | Version of the Helm chart to deploy |
| `helm_chart_name` | Yes | - | Name of the Helm chart |
| `helm_values_file` | No | `values-production.yaml` | Path to the Helm values file in your repository |
| `variables_mapping` | No | `{}` | JSON object mapping variable names to non-secret values |
| `secrets_mapping` | Yes | - | JSON object mapping variable names to secret names (for lookup) |

## Variable and Secret Mapping

### Variables Mapping (Non-Secrets)

The `variables_mapping` input contains **non-sensitive values** that will be visible in workflow logs. Use this for:
- Image tags
- Build numbers
- Environment names
- Replica counts
- GitHub context values

**Format:**
```json
{
  "VAR_NAME": "value",
  "ANOTHER_VAR": "${{ github.sha }}",
  "BUILD_NUMBER": "${{ github.run_number }}"
}
```

**Example:**
```json
{
  "POC_IMAGE_TAG": "${{ github.sha }}",
  "POC_ENVIRONMENT": "production",
  "POC_REPLICA_COUNT": "3",
  "POC_BUILD_NUMBER": "${{ github.run_number }}"
}
```

### Secrets Mapping (Sensitive Data)

The `secrets_mapping` input maps variable names to **secret names** (not values). The workflow will lookup these secrets from the consumer repository's secrets store.

**Important**: This mapping uses secret **names**, not values. The workflow will automatically retrieve the actual secret values.

**Format:**
```json
{
  "VAR_NAME_IN_HELM": "SECRET_NAME_IN_REPO",
  "ANOTHER_VAR": "ANOTHER_SECRET_NAME"
}
```

**Example:**
```json
{
  "POC_DATABASE_HOST": "GS_POC_DATABASE_HOST",
  "POC_DATABASE_PASSWORD": "GS_POC_DATABASE_PASSWORD",
  "POC_API_KEY": "GS_POC_API_KEY"
}
```

In this example:
- The Helm values file uses `${POC_DATABASE_HOST}`
- The workflow looks up the secret named `GS_POC_DATABASE_HOST` from your repository
- The secret value is used to replace `${POC_DATABASE_HOST}` in the values file

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
    uses: Grupo-118-Tech-Challenge-Fiap-11SOAT/terraform-template-pipeline-grupo118-fase-3/.github/workflows/dotnet-cd-template.yml@main
    with:
      helm_chart_version: "1.0.0"
      helm_chart_name: "my-dotnet-api"
      variables_mapping: |
        {
          "POC_IMAGE_TAG": "${{ github.sha }}"
        }
      secrets_mapping: |
        {
          "DOCKER_REGISTRY": "DOCKER_REGISTRY",
          "ACR_USERNAME": "ACR_USERNAME",
          "ACR_PASSWORD": "ACR_PASSWORD"
        }
```

### Deployment with Database Secrets

```yaml
jobs:
  deploy:
    uses: Grupo-118-Tech-Challenge-Fiap-11SOAT/terraform-template-pipeline-grupo118-fase-3/.github/workflows/dotnet-cd-template.yml@main
    with:
      helm_chart_version: "1.0.0"
      helm_chart_name: "my-app"
      helm_values_file: "helm/values-production.yaml"
      variables_mapping: |
        {
          "POC_IMAGE_TAG": "${{ github.sha }}",
          "POC_ENVIRONMENT": "production",
          "POC_REPLICA_COUNT": "3"
        }
      secrets_mapping: |
        {
          "DOCKER_REGISTRY": "DOCKER_REGISTRY",
          "POC_DATABASE_HOST": "PROD_DB_HOST",
          "POC_DATABASE_PORT": "PROD_DB_PORT",
          "POC_DATABASE_NAME": "PROD_DB_NAME",
          "POC_DATABASE_USER": "PROD_DB_USER",
          "POC_DATABASE_PASSWORD": "PROD_DB_PASSWORD"
        }
```

### Deployment with Multiple Environments

```yaml
jobs:
  deploy-staging:
    uses: Grupo-118-Tech-Challenge-Fiap-11SOAT/terraform-template-pipeline-grupo118-fase-3/.github/workflows/dotnet-cd-template.yml@main
    with:
      helm_chart_version: "1.0.0"
      helm_chart_name: "my-app"
      helm_values_file: "helm/values-staging.yaml"
      variables_mapping: |
        {
          "POC_IMAGE_TAG": "${{ github.sha }}",
          "POC_ENVIRONMENT": "staging",
          "POC_REPLICA_COUNT": "2"
        }
      secrets_mapping: |
        {
          "DOCKER_REGISTRY": "DOCKER_REGISTRY",
          "POC_DATABASE_HOST": "STAGING_DB_HOST",
          "POC_API_KEY": "STAGING_API_KEY"
        }

  deploy-production:
    needs: deploy-staging
    uses: Grupo-118-Tech-Challenge-Fiap-11SOAT/terraform-template-pipeline-grupo118-fase-3/.github/workflows/dotnet-cd-template.yml@main
    with:
      helm_chart_version: "1.0.0"
      helm_chart_name: "my-app"
      helm_values_file: "helm/values-production.yaml"
      variables_mapping: |
        {
          "POC_IMAGE_TAG": "${{ github.sha }}",
          "POC_ENVIRONMENT": "production",
          "POC_REPLICA_COUNT": "5"
        }
      secrets_mapping: |
        {
          "DOCKER_REGISTRY": "DOCKER_REGISTRY",
          "POC_DATABASE_HOST": "PROD_DB_HOST",
          "POC_API_KEY": "PROD_API_KEY"
        }
```

### Using Workflow Dispatch with Custom Image Tag

```yaml
on:
  workflow_dispatch:
    inputs:
      image_tag:
        description: 'Docker image tag to deploy'
        required: true
        type: string
      environment:
        description: 'Target environment'
        required: true
        type: choice
        options:
          - staging
          - production

jobs:
  deploy:
    uses: Grupo-118-Tech-Challenge-Fiap-11SOAT/terraform-template-pipeline-grupo118-fase-3/.github/workflows/dotnet-cd-template.yml@main
    with:
      helm_chart_version: "1.0.0"
      helm_chart_name: "my-app"
      variables_mapping: |
        {
          "POC_IMAGE_TAG": "${{ github.event.inputs.image_tag }}",
          "POC_ENVIRONMENT": "${{ github.event.inputs.environment }}",
          "POC_BUILD_NUMBER": "${{ github.run_number }}",
          "POC_COMMIT_SHA": "${{ github.sha }}"
        }
      secrets_mapping: |
        {
          "DOCKER_REGISTRY": "DOCKER_REGISTRY",
          "POC_DATABASE_HOST": "GS_POC_DATABASE_HOST",
          "POC_API_KEY": "GS_POC_API_KEY"
        }
```

## Workflow Steps

1. **Checkout Code** - Checks out the consumer repository
2. **Azure Login** - Authenticates with Azure using service principal (from template repo secrets)
3. **Setup kubectl** - Installs kubectl CLI
4. **Setup Helm** - Installs Helm v3.12.0
5. **Get AKS Credentials** - Retrieves AKS cluster credentials (from template repo secrets)
6. **Process Helm Values with Secrets and Variables** - Processes both mappings:
   - Exports variables from `variables_mapping` (visible in logs)
   - Looks up and exports secrets from `secrets_mapping` (values masked in logs)
   - Replaces all variables in the Helm values file using `envsubst`
7. **Login to ACR** - Authenticates with Azure Container Registry (from template repo secrets)
8. **Deploy to Kubernetes** - Deploys the application using Helm
9. **Cleanup** - Removes temporary processed files

## Security Architecture

### Secrets Separation

The workflow uses a two-tier secret management approach:

**Template Repository Secrets** (Infrastructure):
- Azure credentials (`AZURE_CREDENTIALS`)
- AKS cluster information (`AZURE_RESOURCE_GROUP`, `AKS_CLUSTER_NAME`)
- ACR credentials (`ACR_USERNAME`, `ACR_PASSWORD`)
- Shared infrastructure secrets

**Consumer Repository Secrets** (Application):
- Database credentials
- API keys
- Application-specific secrets
- Environment-specific configuration

### Secret Flow

```
Consumer Repo Secrets
        ↓
  secrets_mapping (secret names only)
        ↓
Workflow (looks up secret values)
        ↓
  envsubst (replaces in values file)
        ↓
  values-processed.yaml
        ↓
    Helm Deploy
```

### Key Security Features

1. **Secret names, not values** - `secrets_mapping` only contains secret names
2. **Automatic masking** - Secret values are automatically masked in logs
3. **Separate mappings** - Clear distinction between secrets and non-secrets
4. **No secrets in workflow definition** - Secrets never appear in YAML files
5. **Runtime lookup** - Secrets are retrieved only during execution

## Troubleshooting

### Common Issues

**1. Values file not found**
```
Error: Values file helm/values-production.yaml not found
```
- Ensure the `helm_values_file` path is correct relative to repository root
- Verify the file exists in your repository
- Check the file path doesn't have leading slashes

**2. Variable not replaced**
```
Final values still show ${VAR_NAME}
```
- For variables: Check that the variable is defined in `variables_mapping`
- For secrets: Check that the secret name in `secrets_mapping` matches a repository secret
- Ensure you're using the correct syntax: `${VAR_NAME}` (not `{{VAR_NAME}}`)
- Verify the JSON syntax is valid (no trailing commas)

**3. Secret not found warning**
```
Warning: Secret 'SECRET_NAME' not found for variable 'VAR_NAME'
```
- Verify the secret exists in your consumer repository settings
- Check secret name spelling in `secrets_mapping`
- Ensure the secret has a value set
- Remember: use secret **name**, not the value

**4. JSON parsing error**
```
jq: parse error: Expected another key-value pair at line X, column Y
```
- Validate your JSON syntax (use a JSON validator)
- Remove trailing commas (common issue)
- Ensure all strings are properly quoted
- Use multi-line string syntax with `|` in YAML

**5. Helm deployment failed**
```
Error: release failed
```
- Check the Helm chart version exists in your ACR
- Verify ACR credentials in template repository are correct
- Review the processed values file output in the workflow logs
- Check Kubernetes namespace permissions

**6. Export command error**
```
export_vars.sh: line X: Setting: command not found
```
- This is already fixed in the current implementation
- Ensure all echo statements use `>&2` for logging messages
- Only export commands should go to `export_vars.sh`

## Security Best Practices

1. **Never commit secrets** - Always use GitHub Secrets for sensitive data
2. **Use least privilege** - Service principals should have minimal required permissions
3. **Rotate credentials** - Regularly update ACR passwords and service principal credentials
4. **Review processed values** - Verify the workflow logs show correctly replaced values
5. **Use separate environments** - Different secrets for staging, production, etc.
6. **Explicit secret mapping** - Only map secrets that are actually needed
7. **Audit secret access** - Monitor who can modify repository secrets
8. **Use meaningful secret names** - Name secrets clearly (e.g., `PROD_DB_PASSWORD`, not `SECRET1`)

## Complete Working Example

### Consumer Repository Structure
```
my-dotnet-app/
├── .github/
│   └── workflows/
│       └── deploy.yml
└── helm/
    ├── values-staging.yaml
    └── values-production.yaml
```

### helm/values-production.yaml
```yaml
replicaCount: ${POC_REPLICA_COUNT}

image:
  repository: ${DOCKER_REGISTRY}/${POC_IMAGE_NAME}
  tag: ${POC_IMAGE_TAG}
  pullPolicy: Always

service:
  type: LoadBalancer
  port: 80
  targetPort: 8080

env:
  - name: ENVIRONMENT
    value: "${POC_ENVIRONMENT}"
  - name: ConnectionStrings__DefaultConnection
    value: "Server=${POC_DATABASE_HOST};Port=${POC_DATABASE_PORT};Database=${POC_DATABASE_NAME};User Id=${POC_DATABASE_USER};Password=${POC_DATABASE_PASSWORD};"
  - name: JwtSettings__Secret
    value: "${POC_JWT_SECRET}"
  - name: ApiKeys__ExternalService
    value: "${POC_API_KEY}"
```

### .github/workflows/deploy.yml
```yaml
name: Deploy to Production

on:
  push:
    branches: [main]
  workflow_dispatch:
    inputs:
      image_tag:
        description: 'Image tag to deploy'
        required: true
        type: string

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to AKS
        uses: Grupo-118-Tech-Challenge-Fiap-11SOAT/terraform-template-pipeline-grupo118-fase-3/.github/workflows/dotnet-cd-template.yml@main
        with:
          helm_chart_version: "1.0.0"
          helm_chart_name: "my-dotnet-app"
          helm_values_file: "helm/values-production.yaml"
          
          variables_mapping: |
            {
              "POC_IMAGE_NAME": "my-dotnet-app",
              "POC_IMAGE_TAG": "${{ github.event.inputs.image_tag || github.sha }}",
              "POC_ENVIRONMENT": "production",
              "POC_REPLICA_COUNT": "3"
            }
          
          secrets_mapping: |
            {
              "DOCKER_REGISTRY": "DOCKER_REGISTRY",
              "ACR_USERNAME": "ACR_USERNAME",
              "ACR_PASSWORD": "ACR_PASSWORD",
              "POC_DATABASE_HOST": "PROD_DB_HOST",
              "POC_DATABASE_PORT": "PROD_DB_PORT",
              "POC_DATABASE_NAME": "PROD_DB_NAME",
              "POC_DATABASE_USER": "PROD_DB_USER",
              "POC_DATABASE_PASSWORD": "PROD_DB_PASSWORD",
              "POC_JWT_SECRET": "JWT_SECRET",
              "POC_API_KEY": "EXTERNAL_API_KEY"
            }
```

### Required Secrets in Consumer Repository
- `DOCKER_REGISTRY`
- `ACR_USERNAME`
- `ACR_PASSWORD`
- `PROD_DB_HOST`
- `PROD_DB_PORT`
- `PROD_DB_NAME`
- `PROD_DB_USER`
- `PROD_DB_PASSWORD`
- `JWT_SECRET`
- `EXTERNAL_API_KEY`

## Workflow Output Example

When the workflow runs, you'll see output like:

```
=== Processing Variables ===
Setting variable: POC_IMAGE_TAG
Setting variable: POC_ENVIRONMENT
Setting variable: POC_REPLICA_COUNT

=== Processing Secrets ===
Setting secret variable: DOCKER_REGISTRY (from secret: DOCKER_REGISTRY)
Setting secret variable: POC_DATABASE_HOST (from secret: PROD_DB_HOST)
Setting secret variable: POC_DATABASE_PASSWORD (from secret: PROD_DB_PASSWORD)

=== Generated Exports (secrets masked) ===
export POC_IMAGE_TAG=***MASKED***
export POC_ENVIRONMENT=***MASKED***
export DOCKER_REGISTRY=***MASKED***
...

=== Processed Helm Values ===
[values file content with secrets replaced]
```

## Support

For issues or questions:
1. Check the workflow run logs for detailed error messages
2. Review the "Processed Helm values" output in the workflow logs
3. Verify all required secrets are configured in both repositories
4. Ensure Helm chart is available in ACR
5. Validate your JSON syntax in both mappings (use a JSON validator)
6. Check that secret names in `secrets_mapping` match actual repository secrets

## Related Documentation

- [Helm Chart Repository](https://github.com/Grupo-118-Tech-Challenge-Fiap-11SOAT/helm-chart-grupo118-fase-4)
- [Azure Kubernetes Service](https://docs.microsoft.com/azure/aks/)
- [Helm Documentation](https://helm.sh/docs/)
- [GitHub Actions Reusable Workflows](https://docs.github.com/en/actions/using-workflows/reusing-workflows)
- [GitHub Secrets](https://docs.github.com/en/actions/security-guides/encrypted-secrets)
- [envsubst Documentation](https://www.gnu.org/software/gettext/manual/html_node/envsubst-Invocation.html)