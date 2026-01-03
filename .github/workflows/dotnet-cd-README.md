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

3. **Application-Specific Secrets in Consumer Repository**:
   - `DOCKER_REGISTRY` - ACR registry URL (e.g., `myregistry.azurecr.io`)
   - `ACR_USERNAME` - ACR username
   - `ACR_PASSWORD` - ACR password
   - Database credentials (e.g., `GS_POC_DATABASE_HOST`, `GS_POC_DATABASE_PORT`, etc.)
   - API keys (e.g., `GS_POC_API_KEY`)
   - Connection strings
   - Any other application-specific secrets

4. **Important**: Use `secrets: inherit` in your workflow to allow access to secrets from both repositories.

### Consumer Repository Setup

#### 1. Create Helm Values File

Create a `values-production.yaml` file in your repository with variable placeholders using `${VAR_NAME}` syntax:

```yaml
# helm/values-production.yaml
replicaCount: ${POC_REPLICA_COUNT}

image:
  repository: ${POC_IMAGE_REPOSITORY}
  pullPolicy: Always
  tag: ${POC_IMAGE_TAG}

imagePullSecrets:
  - name: acr-secret

nameOverride: ""
fullnameOverride: ""

serviceAccount:
  create: true
  automount: true
  annotations: {}
  name: ""

podAnnotations: {}
podLabels: {}

podSecurityContext:
  runAsNonRoot: true
  runAsUser: 1000
  fsGroup: 1000

securityContext:
  allowPrivilegeEscalation: false
  capabilities:
    drop:
    - ALL
  readOnlyRootFilesystem: true

service:
  type: LoadBalancer
  port: 80
  targetPort: 8080
  annotations: {}

ingress:
  enabled: false
  className: ""
  annotations: {}
  hosts:
    - host: chart-example.local
      paths:
        - path: /
          pathType: ImplementationSpecific
  tls: []

internalIngress:
   enabled: true
   className: "nginx-internal-static"
   path: /my-api(/$)(.*)
   pathType: ImplementationSpecific
   annotations:
      nginx.ingress.kubernetes.io/rewrite-target: /$2
      nginx.ingress.kubernetes.io/use-regex: "true"
   tls: []

resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 250m
    memory: 256Mi

livenessProbe:
  httpGet:
    path: /health
    port: http
  initialDelaySeconds: 30
  periodSeconds: 10
  timeoutSeconds: 5
  failureThreshold: 3

readinessProbe:
  httpGet:
    path: /health
    port: http
  initialDelaySeconds: 10
  periodSeconds: 5
  timeoutSeconds: 3
  failureThreshold: 3

autoscaling:
  enabled: false
  minReplicas: 1
  maxReplicas: 10
  targetCPUUtilizationPercentage: 80

volumes:
  - name: tmp
    emptyDir: {}

volumeMounts:
  - name: tmp
    mountPath: /tmp

nodeSelector: {}

tolerations: []

affinity: {}

# Secret configuration
secret:
  enabled: true
  name: "poc-api-dotnet-cicd-template-usage-fase4"
  annotations: {}
  data:
    poc-db-host: "${POC_DATABASE_HOST}"
    poc-db-port: "${POC_DATABASE_PORT}"
    poc-db-name: "${POC_DATABASE_NAME}"
    poc-db-user: "${POC_DATABASE_USER}"
    poc-db-password: "${POC_DATABASE_PASSWORD}"
    poc-jwt-secret: "${POC_JWT_SECRET}"
    poc-api-key: "${POC_API_KEY}"
  envVars:
    - name: POC_DATABASE_HOST
      secretKey: poc-db-host
    - name: POC_DATABASE_PORT
      secretKey: poc-db-port
    - name: POC_DATABASE_NAME
      secretKey: poc-db-name
    - name: POC_DATABASE_USER
      secretKey: poc-db-user
    - name: POC_DATABASE_PASSWORD
      secretKey: poc-db-password
    - name: POC_JWT_SECRET
      secretKey: poc-jwt-secret
    - name: POC_API_KEY
      secretKey: poc-api-key

# Additional environment variables (non-secret)
env:
  - name: ASPNETCORE_ENVIRONMENT
    value: "${POC_ENVIRONMENT}"
```

#### 2. Create Workflow File

Create a workflow in your consumer repository using **two separate mappings**:

```yaml
# .github/workflows/cd.yml
name: CD Pipeline

on:
  workflow_dispatch:
    inputs:
      image_tag:
        description: 'Image tag to deploy'
        required: true
      helm_chart_version:
        description: 'Helm chart version to deploy'
        required: true
      helm_chart_name:
        description: 'Field to insert the Helm Chart Name'
        required: true
        default: "my-chart"
      helm_release_name:
        description: 'Field to define the release name'
        required: true
        default: 'my-release-name'

jobs:
  deploy:
    uses: Grupo-118-Tech-Challenge-Fiap-11SOAT/terraform-template-pipeline-grupo118-fase-3/.github/workflows/dotnet-cd-template.yml@develop
    secrets: inherit
    with:
      helm_chart_version: ${{ github.event.inputs.helm_chart_version }}
      helm_chart_name: ${{ github.event.inputs.helm_chart_name }}
      helm_values_file: "helm/values-production.yaml"
      helm_release_name: ${{ github.event.inputs.helm_release_name }}
      
      # Secret mappings (variable name → secret name to lookup)
      # Include infrastructure secrets (ACR) and application secrets
      secrets_mapping: |
        {
          "DOCKER_REGISTRY": "DOCKER_REGISTRY",
          "ACR_USERNAME": "ACR_USERNAME",
          "ACR_PASSWORD": "ACR_PASSWORD",
          "POC_DATABASE_HOST": "GS_POC_DATABASE_HOST",
          "POC_DATABASE_PORT": "GS_POC_DATABASE_PORT",
          "POC_DATABASE_NAME": "GS_POC_DATABASE_NAME",
          "POC_API_KEY": "GS_POC_API_KEY"
        }
      
      # Non-secret variables (visible in logs for debugging)
      variables_mapping: |
        {
          "POC_IMAGE_TAG": "${{ github.event.inputs.image_tag }}"
        }
```

## Workflow Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `helm_chart_version` | Yes | - | Version of the Helm chart to deploy |
| `helm_chart_name` | Yes | - | Name of the Helm chart |
| `helm_release_name` | No | Same as `helm_chart_name` | Helm release name for the deployment |
| `helm_values_file` | No | `values-production.yaml` | Path to the Helm values file in your repository |
| `variables_mapping` | No | `{}` | JSON object mapping variable names to non-secret values |
| `secrets_mapping` | Yes | - | JSON object mapping variable names to secret names (for lookup) |

## Variable and Secret Mapping

### Important: Using `secrets: inherit`

The workflow **must** include `secrets: inherit` to allow the reusable workflow to access secrets from both the consumer repository and the template repository. This is essential for the workflow to function properly.

```yaml
jobs:
  deploy:
    uses: org/repo/.github/workflows/dotnet-cd-template.yml@develop
    secrets: inherit  # ← Required for secret access
    with:
      # ... inputs
```

### Variables Mapping (Non-Secrets)

The `variables_mapping` input contains **non-sensitive values** that will be visible in workflow logs. Use this for:
- Image tags
- Image repository paths
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
  "POC_IMAGE_TAG": "${{ github.event.inputs.image_tag }}",
  "POC_IMAGE_REPOSITORY": "pocdevopsgrupo118.azurecr.io/poc-api-dotnet",
  "POC_ENVIRONMENT": "Production",
  "POC_REPLICA_COUNT": "3"
}
```

### Secrets Mapping (Sensitive Data)

The `secrets_mapping` input maps variable names to **secret names** (not values). The workflow will lookup these secrets from the consumer repository's secrets store.

**Important**: 
- This mapping uses secret **names**, not values
- Include both infrastructure secrets (ACR credentials) and application secrets
- ACR credentials (`DOCKER_REGISTRY`, `ACR_USERNAME`, `ACR_PASSWORD`) should typically map to themselves

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
  "DOCKER_REGISTRY": "DOCKER_REGISTRY",
  "ACR_USERNAME": "ACR_USERNAME",
  "ACR_PASSWORD": "ACR_PASSWORD",
  "POC_DATABASE_HOST": "GS_POC_DATABASE_HOST",
  "POC_DATABASE_PORT": "GS_POC_DATABASE_PORT",
  "POC_DATABASE_NAME": "GS_POC_DATABASE_NAME",
  "POC_API_KEY": "GS_POC_API_KEY"
}
```

In this example:
- Infrastructure secrets (ACR) are mapped directly
- The Helm values file uses `${POC_DATABASE_HOST}` in the `secret.data` section
- The workflow looks up the secret named `GS_POC_DATABASE_HOST` from your repository
- The secret value is used to replace `${POC_DATABASE_HOST}` in the values file
- The Helm chart creates a Kubernetes Secret and injects it as environment variables via `secret.envVars`

## Variable Replacement

The workflow uses `envsubst` to replace variables in your Helm values file. Variables should be defined using the `${VAR_NAME}` syntax.

**Supported formats:**
- `${VAR_NAME}` - Simple variable replacement
- `${VAR_NAME:-default}` - Variable with default value if not set
- `"${VAR_NAME}"` - Quoted variables (recommended for string values)

## Examples

### Basic Deployment (Based on POC Usage)

```yaml
name: CD Pipeline

on:
  workflow_dispatch:
    inputs:
      image_tag:
        description: 'Image tag to deploy'
        required: true
      helm_chart_version:
        description: 'Helm chart version to deploy'
        required: true
      helm_chart_name:
        description: 'Helm Chart Name'
        required: true
        default: "my-chart"
      helm_release_name:
        description: 'Release name'
        required: true
        default: 'my-app'

jobs:
  deploy:
    uses: Grupo-118-Tech-Challenge-Fiap-11SOAT/terraform-template-pipeline-grupo118-fase-3/.github/workflows/dotnet-cd-template.yml@develop
    secrets: inherit
    with:
      helm_chart_version: ${{ github.event.inputs.helm_chart_version }}
      helm_chart_name: ${{ github.event.inputs.helm_chart_name }}
      helm_values_file: "helm/values-production.yaml"
      helm_release_name: ${{ github.event.inputs.helm_release_name }}
      secrets_mapping: |
        {
          "DOCKER_REGISTRY": "DOCKER_REGISTRY",
          "ACR_USERNAME": "ACR_USERNAME",
          "ACR_PASSWORD": "ACR_PASSWORD",
          "POC_DATABASE_HOST": "GS_POC_DATABASE_HOST",
          "POC_DATABASE_PORT": "GS_POC_DATABASE_PORT",
          "POC_DATABASE_NAME": "GS_POC_DATABASE_NAME",
          "POC_API_KEY": "GS_POC_API_KEY"
        }
      variables_mapping: |
        {
          "POC_IMAGE_TAG": "${{ github.event.inputs.image_tag }}"
        }
```

### Deployment with Additional Secrets

```yaml
jobs:
  deploy:
    uses: Grupo-118-Tech-Challenge-Fiap-11SOAT/terraform-template-pipeline-grupo118-fase-3/.github/workflows/dotnet-cd-template.yml@develop
    secrets: inherit
    with:
      helm_chart_version: "1.0.0"
      helm_chart_name: "poc-api-dotnet"
      helm_values_file: "helm/values-production.yaml"
      helm_release_name: "my-app-production"
      variables_mapping: |
        {
          "POC_IMAGE_TAG": "${{ github.sha }}",
          "POC_IMAGE_REPOSITORY": "pocdevopsgrupo118.azurecr.io/poc-api-dotnet",
          "POC_ENVIRONMENT": "Production",
          "POC_REPLICA_COUNT": "3"
        }
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
          "POC_JWT_SECRET": "GS_POC_JWT_SECRET",
          "POC_API_KEY": "GS_POC_API_KEY"
        }
```

### Deployment with Multiple Environments

```yaml
jobs:
  deploy-staging:
    uses: Grupo-118-Tech-Challenge-Fiap-11SOAT/terraform-template-pipeline-grupo118-fase-3/.github/workflows/dotnet-cd-template.yml@develop
    secrets: inherit
    with:
      helm_chart_version: "1.0.0"
      helm_chart_name: "poc-api-dotnet"
      helm_values_file: "helm/values-staging.yaml"
      helm_release_name: "my-app-staging"
      variables_mapping: |
        {
          "POC_IMAGE_TAG": "${{ github.sha }}",
          "POC_IMAGE_REPOSITORY": "pocdevopsgrupo118.azurecr.io/poc-api-dotnet",
          "POC_ENVIRONMENT": "Staging",
          "POC_REPLICA_COUNT": "2"
        }
      secrets_mapping: |
        {
          "DOCKER_REGISTRY": "DOCKER_REGISTRY",
          "ACR_USERNAME": "ACR_USERNAME",
          "ACR_PASSWORD": "ACR_PASSWORD",
          "POC_DATABASE_HOST": "STAGING_DATABASE_HOST",
          "POC_DATABASE_PORT": "STAGING_DATABASE_PORT",
          "POC_DATABASE_NAME": "STAGING_DATABASE_NAME",
          "POC_DATABASE_USER": "STAGING_DATABASE_USER",
          "POC_DATABASE_PASSWORD": "STAGING_DATABASE_PASSWORD",
          "POC_API_KEY": "STAGING_API_KEY"
        }

  deploy-production:
    needs: deploy-staging
    uses: Grupo-118-Tech-Challenge-Fiap-11SOAT/terraform-template-pipeline-grupo118-fase-3/.github/workflows/dotnet-cd-template.yml@develop
    secrets: inherit
    with:
      helm_chart_version: "1.0.0"
      helm_chart_name: "poc-api-dotnet"
      helm_values_file: "helm/values-production.yaml"
      helm_release_name: "my-app-production"
      variables_mapping: |
        {
          "POC_IMAGE_TAG": "${{ github.sha }}",
          "POC_IMAGE_REPOSITORY": "pocdevopsgrupo118.azurecr.io/poc-api-dotnet",
          "POC_ENVIRONMENT": "Production",
          "POC_REPLICA_COUNT": "5"
        }
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

### Deployment with Autoscaling

```yaml
jobs:
  deploy:
    uses: Grupo-118-Tech-Challenge-Fiap-11SOAT/terraform-template-pipeline-grupo118-fase-3/.github/workflows/dotnet-cd-template.yml@develop
    secrets: inherit
    with:
      helm_chart_version: "1.0.0"
      helm_chart_name: "poc-api-dotnet"
      helm_values_file: "helm/values-production.yaml"
      helm_release_name: "my-app"
      variables_mapping: |
        {
          "POC_IMAGE_TAG": "${{ github.sha }}",
          "POC_IMAGE_REPOSITORY": "pocdevopsgrupo118.azurecr.io/poc-api-dotnet",
          "POC_ENVIRONMENT": "Production",
          "POC_AUTOSCALING_ENABLED": "true",
          "POC_AUTOSCALING_MIN_REPLICAS": "2",
          "POC_AUTOSCALING_MAX_REPLICAS": "10",
          "POC_AUTOSCALING_TARGET_CPU": "80"
        }
      secrets_mapping: |
        {
          "DOCKER_REGISTRY": "DOCKER_REGISTRY",
          "ACR_USERNAME": "ACR_USERNAME",
          "ACR_PASSWORD": "ACR_PASSWORD",
          "POC_DATABASE_HOST": "GS_POC_DATABASE_HOST",
          "POC_DATABASE_PASSWORD": "GS_POC_DATABASE_PASSWORD"
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
      helm_chart_version:
        description: 'Helm chart version to deploy'
        required: true
      helm_chart_name:
        description: 'Helm Chart Name'
        required: true
        default: "poc-api-dotnet"
      helm_release_name:
        description: 'Release name'
        required: true
        default: 'my-app'
      environment:
        description: 'Target environment'
        required: true

jobs:
  deploy:
    uses: Grupo-118-Tech-Challenge-Fiap-11SOAT/terraform-template-pipeline-grupo118-fase-3/.github/workflows/dotnet-cd-template.yml@develop
    secrets: inherit
    with:
      helm_chart_version: ${{ github.event.inputs.helm_chart_version }}
      helm_chart_name: ${{ github.event.inputs.helm_chart_name }}
      helm_release_name: ${{ github.event.inputs.helm_release_name }}
      helm_values_file: "helm/values-production.yaml"
      variables_mapping: |
        {
          "POC_IMAGE_TAG": "${{ github.event.inputs.image_tag }}",
          "POC_IMAGE_REPOSITORY": "pocdevopsgrupo118.azurecr.io/poc-api-dotnet",
          "POC_ENVIRONMENT": "${{ github.event.inputs.environment }}",
          "POC_REPLICA_COUNT": "${{ github.event.inputs.environment == 'Production' && '5' || '2' }}",
          "POC_BUILD_NUMBER": "${{ github.run_number }}",
          "POC_COMMIT_SHA": "${{ github.sha }}"
        }
      secrets_mapping: |
        {
          "DOCKER_REGISTRY": "DOCKER_REGISTRY",
          "ACR_USERNAME": "ACR_USERNAME",
          "ACR_PASSWORD": "ACR_PASSWORD",
          "POC_DATABASE_HOST": "${{ github.event.inputs.environment == 'Production' && 'GS_POC_DATABASE_HOST' || 'STAGING_DATABASE_HOST' }}",
          "POC_DATABASE_PASSWORD": "${{ github.event.inputs.environment == 'Production' && 'GS_POC_DATABASE_PASSWORD' || 'STAGING_DATABASE_PASSWORD' }}",
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
- Shared infrastructure secrets

**Consumer Repository Secrets** (Application + ACR):
- ACR credentials (`DOCKER_REGISTRY`, `ACR_USERNAME`, `ACR_PASSWORD`)
- Database credentials
- API keys
- Application-specific secrets
- Environment-specific configuration

### Secret Flow

```
Consumer Repo Secrets (via secrets: inherit)
        ↓
  secrets_mapping (secret names only)
        ↓
Workflow (looks up secret values)
        ↓
  envsubst (replaces in values file)
        ↓
  values-processed.yaml (secret.data filled)
        ↓
    Helm Deploy
        ↓
Kubernetes Secret created
        ↓
Environment variables injected via secret.envVars
```

### Key Security Features

1. **secrets: inherit** - Enables access to secrets from both template and consumer repositories
2. **Secret names, not values** - `secrets_mapping` only contains secret names
3. **Automatic masking** - Secret values are automatically masked in logs
4. **Separate mappings** - Clear distinction between secrets and non-secrets
5. **No secrets in workflow definition** - Secrets never appear in YAML files
6. **Runtime lookup** - Secrets are retrieved only during execution
7. **Kubernetes Secrets** - Secrets are stored as Kubernetes Secrets and injected as environment variables

## Troubleshooting

### Common Issues

**1. Secret not accessible / Secret not found**
```
Error: Secret 'SECRET_NAME' not found
```
- Ensure `secrets: inherit` is included in your workflow job configuration
- Verify the secret exists in your consumer repository settings
- Check secret name spelling in `secrets_mapping`
- Ensure the secret has a value set
- Remember: use secret **name**, not the value

**2. Values file not found**
```
Error: Values file helm/values-production.yaml not found
```
- Ensure the `helm_values_file` path is correct relative to repository root
- Verify the file exists in your repository
- Check the file path doesn't have leading slashes

**3. Variable not replaced**
```
Final values still show ${VAR_NAME}
```
- For variables: Check that the variable is defined in `variables_mapping`
- For secrets: Check that the secret name in `secrets_mapping` matches a repository secret
- Ensure you're using the correct syntax: `${VAR_NAME}` (not `{{VAR_NAME}}`)
- Verify the JSON syntax is valid (no trailing commas)

**4. Secret not found warning**
```
Warning: Secret 'SECRET_NAME' not found for variable 'VAR_NAME'
```
- Verify `secrets: inherit` is present in the job
- Verify the secret exists in your consumer repository settings
- Check secret name spelling in `secrets_mapping`
- Ensure the secret has a value set
- Remember: use secret **name**, not the value

**5. JSON parsing error**
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

**6. Environment variables not available in pod**
```
Pod shows empty environment variables
```
- Verify `secret.enabled` is set to `true`
- Check that `secret.data` keys match the `secret.envVars` `secretKey` values
- Ensure all secret variables are properly replaced in the values file
- Review the Kubernetes Secret in the cluster: `kubectl get secret <secret-name> -o yaml`

## Security Best Practices

1. **Never commit secrets** - Always use GitHub Secrets for sensitive data
2. **Use least privilege** - Service principals should have minimal required permissions
3. **Rotate credentials** - Regularly update ACR passwords and service principal credentials
4. **Review processed values** - Verify the workflow logs show correctly replaced values
5. **Use separate environments** - Different secrets for staging, production, etc.
6. **Explicit secret mapping** - Only map secrets that are actually needed
7. **Audit secret access** - Monitor who can modify repository secrets
8. **Use meaningful secret names** - Name secrets clearly (e.g., `PROD_DB_PASSWORD`, not `SECRET1`)
9. **Secret key naming** - Use kebab-case for secret data keys (e.g., `poc-db-host`)
10. **Environment variable mapping** - Ensure `secret.envVars` correctly maps secret keys to environment variable names

## Complete Working Example

### Consumer Repository Structure
```
poc-api-dotnet-cicd-template-usage-fase4/
├── .github/
│   └── workflows/
│       └── cd.yml
└── helm/
    └── values-production.yaml
```

### helm/values-production.yaml
```yaml
replicaCount: 1

image:
  repository: pocdevopsgrupo118.azurecr.io/poc-api-dotnet
  pullPolicy: Always
  tag: ${POC_IMAGE_TAG}

imagePullSecrets:
  - name: acr-secret

serviceAccount:
  create: true
  automount: true

podSecurityContext:
  runAsNonRoot: true
  runAsUser: 1000
  fsGroup: 1000

securityContext:
  allowPrivilegeEscalation: false
  capabilities:
    drop:
    - ALL
  readOnlyRootFilesystem: true

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

livenessProbe:
  httpGet:
    path: /health
    port: http
  initialDelaySeconds: 30
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /health
    port: http
  initialDelaySeconds: 10
  periodSeconds: 5

volumes:
  - name: tmp
    emptyDir: {}

volumeMounts:
  - name: tmp
    mountPath: /tmp

secret:
  enabled: true
  name: "poc-api-dotnet-cicd-template-usage-fase4"
  data:
    poc-db-host: "${POC_DATABASE_HOST}"
    poc-db-port: "${POC_DATABASE_PORT}"
    poc-db-name: "${POC_DATABASE_NAME}"
    poc-api-key: "${POC_API_KEY}"
  envVars:
    - name: POC_DATABASE_HOST
      secretKey: poc-db-host
    - name: POC_DATABASE_PORT
      secretKey: poc-db-port
    - name: POC_DATABASE_NAME
      secretKey: poc-db-name
    - name: POC_API_KEY
      secretKey: poc-api-key
```

### .github/workflows/cd.yml
```yaml
name: CD Pipeline

on:
  workflow_dispatch:
    inputs:
      image_tag:
        description: 'Image tag to deploy'
        required: true
      helm_chart_version:
        description: 'Helm chart version to deploy'
        required: true
      helm_chart_name:
        description: 'Field to insert the Helm Chart Name'
        required: true
        default: "grupo118fase4"
      helm_release_name:
        description: 'Field to define the release name'
        required: true
        default: 'poc-api-dotnet-cicd-template-usage-fase4'

jobs:
  deploy:
    uses: Grupo-118-Tech-Challenge-Fiap-11SOAT/terraform-template-pipeline-grupo118-fase-3/.github/workflows/dotnet-cd-template.yml@develop
    secrets: inherit
    with:
      helm_chart_version: ${{ github.event.inputs.helm_chart_version }}
      helm_chart_name: ${{ github.event.inputs.helm_chart_name }}
      helm_values_file: "helm/values-production.yaml"
      helm_release_name: ${{ github.event.inputs.helm_release_name }}
      secrets_mapping: |
        {
          "DOCKER_REGISTRY": "DOCKER_REGISTRY",
          "ACR_USERNAME": "ACR_USERNAME",
          "ACR_PASSWORD": "ACR_PASSWORD",
          "POC_DATABASE_HOST": "GS_POC_DATABASE_HOST",
          "POC_DATABASE_PORT": "GS_POC_DATABASE_PORT",
          "POC_DATABASE_NAME": "GS_POC_DATABASE_NAME",
          "POC_API_KEY": "GS_POC_API_KEY"
        }
      variables_mapping: |
        {
          "POC_IMAGE_TAG": "${{ github.event.inputs.image_tag }}"
        }
```

### Required Secrets in Consumer Repository
- `DOCKER_REGISTRY` - ACR registry URL (e.g., `pocdevopsgrupo118.azurecr.io`)
- `ACR_USERNAME` - ACR username
- `ACR_PASSWORD` - ACR password
- `GS_POC_DATABASE_HOST` - Database host
- `GS_POC_DATABASE_PORT` - Database port
- `GS_POC_DATABASE_NAME` - Database name
- `GS_POC_API_KEY` - API key

## Workflow Output Example

When the workflow runs, you'll see output like:

```
=== Processing Variables ===
Setting variable: POC_IMAGE_TAG
Setting variable: POC_IMAGE_REPOSITORY
Setting variable: POC_ENVIRONMENT
Setting variable: POC_REPLICA_COUNT

=== Processing Secrets ===
Setting secret variable: POC_DATABASE_HOST (from secret: GS_POC_DATABASE_HOST)
Setting secret variable: POC_DATABASE_PORT (from secret: GS_POC_DATABASE_PORT)
Setting secret variable: POC_DATABASE_PASSWORD (from secret: GS_POC_DATABASE_PASSWORD)

=== Generated Exports (secrets masked) ===
export POC_IMAGE_TAG=***MASKED***
export POC_IMAGE_REPOSITORY=***MASKED***
export POC_DATABASE_HOST=***MASKED***
...

=== Processed Helm Values ===
secret:
  enabled: true
  name: "poc-api-dotnet-cicd-template-usage-fase4"
  data:
    poc-db-host: "***MASKED***"
    poc-db-port: "***MASKED***"
    poc-db-name: "***MASKED***"
...
```

## Support

For issues or questions:
1. Check the workflow run logs for detailed error messages
2. Review the "Processed Helm values" output in the workflow logs
3. Verify all required secrets are configured in both repositories
4. Ensure Helm chart is available in ACR
5. Validate your JSON syntax in both mappings (use a JSON validator)
6. Check that secret names in `secrets_mapping` match actual repository secrets
7. Verify Kubernetes Secret was created: `kubectl get secret poc-api-dotnet-cicd-template-usage-fase4 -o yaml`

## Related Documentation

- [Helm Chart Repository](https://github.com/Grupo-118-Tech-Challenge-Fiap-11SOAT/helm-chart-grupo118-fase-4)
- [POC API DotNet Template Usage](https://github.com/Grupo-118-Tech-Challenge-Fiap-11SOAT/poc-api-dotnet-cicd-template-usage-fase4)
- [Azure Kubernetes Service](https://docs.microsoft.com/azure/aks/)
- [Helm Documentation](https://helm.sh/docs/)
- [GitHub Actions Reusable Workflows](https://docs.github.com/en/actions/using-workflows/reusing-workflows)
- [GitHub Secrets](https://docs.github.com/en/actions/security-guides/encrypted-secrets)
- [envsubst Documentation](https://www.gnu.org/software/gettext/manual/html_node/envsubst-Invocation.html)
- [Kubernetes Secrets](https://kubernetes.io/docs/concepts/configuration/secret/)