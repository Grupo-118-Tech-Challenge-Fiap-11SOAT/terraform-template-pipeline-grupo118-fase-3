# .NET CI/CD Pipeline Template

This repository provides a reusable GitHub Actions workflow for .NET projects with SonarCloud analysis and Docker container builds. The workflow automates code quality checks, testing, coverage reporting, and Docker image creation/publishing.

## Overview

The pipeline consists of two main jobs:
1. **build-and-analyze**: Runs SonarCloud analysis, builds the project, and executes tests with code coverage
2. **build-docker**: Builds and optionally pushes Docker images to Azure Container Registry

## Usage

To use this reusable workflow in your .NET repository, create a workflow file that calls it:

### Basic Example (Minimum Required)

```yaml
name: CI/CD Pipeline
on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

jobs:
  ci:
    uses: Grupo-118-Tech-Challenge-Fiap-11SOAT/terraform-template-pipeline-grupo118-fase-3/.github/workflows/dotnet/ci-template.yml@main
    with:
      sonar-project-key: "your-org_your-repo"
      sonar-organization: "your-organization"
      solution-path: "."
      test-projects: "**/*Tests.csproj"
      dotnet-version: "8.0.x"
      build-configuration: "Release"
      dockerfile-path: "./Dockerfile"
      image-name: "my-app-name"
    secrets:
      SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
      ACR_USERNAME: ${{ secrets.ACR_USERNAME }}
      ACR_PASSWORD: ${{ secrets.ACR_PASSWORD }}
      ACR_REGISTRY: ${{ secrets.ACR_REGISTRY }}
```

### Example with Custom Paths

```yaml
name: CI/CD Pipeline
on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

jobs:
  ci:
    uses: Grupo-118-Tech-Challenge-Fiap-11SOAT/terraform-template-pipeline-grupo118-fase-3/.github/workflows/dotnet/ci-template.yml@main
    with:
      sonar-project-key: "your-org_your-repo"
      sonar-organization: "your-organization"
      solution-path: "./src/YourSolution.sln"
      test-projects: "./tests/Project1.Tests/Project1.Tests.csproj ./tests/Project2.Tests/Project2.Tests.csproj"
      dotnet-version: "8.0.x"
      build-configuration: "Release"
      dockerfile-path: "./src/Dockerfile"
      image-name: "my-custom-app"
    secrets:
      SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
      ACR_USERNAME: ${{ secrets.ACR_USERNAME }}
      ACR_PASSWORD: ${{ secrets.ACR_PASSWORD }}
      ACR_REGISTRY: ${{ secrets.ACR_REGISTRY }}
```

### Example with Different .NET Version

```yaml
name: CI/CD Pipeline
on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

jobs:
  ci:
    uses: Grupo-118-Tech-Challenge-Fiap-11SOAT/terraform-template-pipeline-grupo118-fase-3/.github/workflows/dotnet/ci-template.yml@main
    with:
      sonar-project-key: "your-org_your-repo"
      sonar-organization: "your-organization"
      solution-path: "."
      test-projects: "**/*Tests.csproj"
      dotnet-version: "9.0.x"
      build-configuration: "Release"
      dockerfile-path: "./Dockerfile"
      image-name: "dotnet9-app"
    secrets:
      SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
      ACR_USERNAME: ${{ secrets.ACR_USERNAME }}
      ACR_PASSWORD: ${{ secrets.ACR_PASSWORD }}
      ACR_REGISTRY: ${{ secrets.ACR_REGISTRY }}
```

### Example Without Tests (SonarCloud Analysis Only)

```yaml
name: CI/CD Pipeline
on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

jobs:
  ci:
    uses: Grupo-118-Tech-Challenge-Fiap-11SOAT/terraform-template-pipeline-grupo118-fase-3/.github/workflows/dotnet/ci-template.yml@main
    with:
      sonar-project-key: "your-org_your-repo"
      sonar-organization: "your-organization"
      solution-path: "."
      test-projects: ""  # Empty string to skip tests
      dotnet-version: "8.0.x"
      build-configuration: "Release"
      dockerfile-path: "./Dockerfile"
      image-name: "my-app"
    secrets:
      SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
      ACR_USERNAME: ${{ secrets.ACR_USERNAME }}
      ACR_PASSWORD: ${{ secrets.ACR_PASSWORD }}
      ACR_REGISTRY: ${{ secrets.ACR_REGISTRY }}
```

## Required Inputs

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `sonar-project-key` | SonarCloud project key (e.g., `org_repo-name`) | **Yes** | - |
| `sonar-organization` | SonarCloud organization key | **Yes** | - |
| `solution-path` | Path to the solution file | **Yes** | `.` |
| `test-projects` | Space-separated list of test project paths (empty string to skip tests) | **Yes** | `**/*Tests.csproj` |
| `dotnet-version` | .NET SDK version to use | **Yes** | `8.0.x` |
| `build-configuration` | Build configuration (Debug/Release) | **Yes** | `Release` |
| `dockerfile-path` | Path to the Dockerfile | **Yes** | `./Dockerfile` |
| `image-name` | Name of the Docker image to build | **Yes** | - |

**Note:** All inputs are marked as `required: true` in the workflow. If you want to use default values, you must explicitly provide them or the workflow will fail.

## Required Secrets

All secrets must be configured in your repository settings:

### SonarCloud
- `SONAR_TOKEN` - SonarCloud authentication token ([Generate here](https://sonarcloud.io/account/security))

### Azure Container Registry
- `ACR_USERNAME` - Azure Container Registry username
- `ACR_PASSWORD` - Azure Container Registry password
- `ACR_REGISTRY` - Azure Container Registry login server (e.g., `yourregistry.azurecr.io`)

## Pipeline Behavior

### Build and Analyze Job

This job runs on every push and pull request:

1. **Checkout code** - Fetches the repository with full history for SonarCloud analysis
2. **Setup .NET** - Installs the specified .NET SDK version
3. **Cache dependencies** - Caches SonarCloud packages and NuGet packages for faster builds
4. **Install SonarCloud scanner** - Installs the dotnet-sonarscanner tool
5. **Restore dependencies** - Restores NuGet packages
6. **SonarCloud Begin** - Initializes SonarCloud analysis with coverage settings
7. **Build** - Compiles the solution with `--no-incremental` flag
8. **Test with Coverage** - Runs tests and collects code coverage in OpenCover format (skipped if `test-projects` is empty)
9. **SonarCloud End** - Finalizes and uploads analysis to SonarCloud
10. **Upload coverage reports** - Saves test results as artifacts (only if tests were run)

### Build Docker Job

This job runs after successful build and analysis:

**For Pull Requests:**
- Builds the Docker image (validation only)
- Does **not** push to the registry
- Uses build cache for faster execution

**For Main/Develop Branch Pushes:**
- Builds the Docker image
- Logs in to Azure Container Registry
- Pushes the image with multiple tags:
  - Branch name (e.g., `main`, `develop`)
  - Git SHA
  - `latest` (only for default branch)
  - Date stamp (for scheduled builds)

## SonarCloud Configuration

The pipeline is pre-configured with the following SonarCloud settings:

- **Coverage Format**: OpenCover XML
- **Test Results**: TRX format
- **Coverage Exclusions**: Test files (`**/*Tests.cs`, `**/*Test.cs`)
- **General Exclusions**: Coverage reports and test results directories (`**/coverage.opencover.xml`, `**/TestResults/**`)
- **Test Inclusions**: Test files for proper categorization

## Code Coverage

Code coverage is collected using:
- **XPlat Code Coverage** collector
- **OpenCover** format for SonarCloud compatibility
- Results stored in `TestResults` directory
- Uploaded as GitHub Actions artifacts (available for 90 days)
- Can be skipped by providing an empty string for `test-projects`

## Docker Build Features

- **BuildKit** support for improved performance
- **Multi-platform builds** capability via buildx
- **Layer caching** using GitHub Actions cache
- **Automatic tagging** based on branch, SHA, and date
- **Conditional push** based on branch and event type
- **Customizable Dockerfile path**

## Prerequisites

Before using this workflow, ensure:

### 1. SonarCloud Project Setup
- Create a project on [SonarCloud.io](https://sonarcloud.io)
- Note your project key and organization
- Generate an authentication token
- Add the token to your repository secrets as `SONAR_TOKEN`

### 2. Azure Container Registry
- Create an ACR instance in Azure
- Create a service principal with push permissions:
  ```bash
  az ad sp create-for-rbac \
    --name "github-actions-acr" \
    --role "AcrPush" \
    --scopes /subscriptions/{subscription-id}/resourceGroups/{resource-group}/providers/Microsoft.ContainerRegistry/registries/{registry-name}
  ```
- Note the registry URL, username (appId), and password

### 3. Repository Secrets
Add all required secrets to your repository:
- Go to Settings → Secrets and variables → Actions → New repository secret
- Add: `SONAR_TOKEN`, `ACR_USERNAME`, `ACR_PASSWORD`, `ACR_REGISTRY`

### 4. Dockerfile
- Ensure your project has a valid Dockerfile
- Place it at the root or specify the path in `dockerfile-path` input

### 5. Test Projects (Optional)
- Test projects should follow the naming convention `*Tests.csproj`
- Or specify custom paths in `test-projects` input (space-separated)
- Set `test-projects: ""` to skip tests entirely

## Troubleshooting

### Common Issues

#### Issue: Workflow fails with "required input not provided"
**Solution:**
All inputs are marked as required in the workflow definition. You must provide values for all inputs, even if you want to use the defaults:
```yaml
with:
  sonar-project-key: "your-org_your-repo"
  sonar-organization: "your-organization"
  solution-path: "."  # Explicitly provide default
  test-projects: "**/*Tests.csproj"  # Explicitly provide default
  dotnet-version: "8.0.x"  # Explicitly provide default
  build-configuration: "Release"  # Explicitly provide default
  dockerfile-path: "./Dockerfile"  # Explicitly provide default
  image-name: "my-app"  # Must provide
```

#### Issue: SonarCloud analysis fails
**Solution:**
- Verify your `SONAR_TOKEN` is valid and not expired
- Check that the project key and organization match your SonarCloud setup
- Ensure fetch-depth is set to 0 for proper analysis

#### Issue: Tests fail or coverage not collected
**Solution:**
- Verify test projects match the `test-projects` pattern
- Check that test projects have proper test framework packages installed (xUnit, NUnit, or MSTest)
- Ensure coverlet.collector package is installed in test projects
- If you don't have tests, set `test-projects: ""`

#### Issue: Docker push fails
**Solution:**
- Verify ACR credentials are correct
- Ensure the service principal has `AcrPush` role
- Check that you're pushing from `main` or `develop` branch
- Verify the ACR allows pushes from your IP/network

#### Issue: Job dependency error - "build-dotnet" not found
**Solution:**
There's a bug in the workflow file. The `build-docker` job references `needs: build-dotnet` but the actual job name is `build-and-analyze`. This needs to be fixed in the template:
```yaml
build-docker:
  needs: build-and-analyze  # Should be this instead of build-dotnet
```

## Best Practices

1. **All Inputs Required**: Remember that all inputs are marked as required, so you must provide values even for defaults
2. **Branch Strategy**: The workflow is optimized for `main` and `develop` branches
3. **Pull Requests**: Always test in PRs before merging - Docker images won't be pushed
4. **Secrets Rotation**: Regularly rotate ACR credentials and SonarCloud tokens
5. **Cache Usage**: The workflow uses caching to speed up builds - first runs may be slower
6. **Coverage Goals**: Set up quality gates in SonarCloud for minimum coverage requirements
7. **Test Projects**: Use space-separated paths for multiple test projects, or empty string to skip
8. **Build Configuration**: Use `Release` configuration for production builds

## Pipeline Outputs

### Artifacts
- **coverage-reports**: Test results and code coverage reports (available for 90 days, only if tests were run)

### Docker Tags
Generated tags follow this pattern:
- `branch-name` (e.g., `main`, `develop`)
- `sha-{git-sha}` (e.g., `sha-abc123`)
- `latest` (only for default branch)
- `YYYYMMDD` (for scheduled builds on default branch)

## Example Repository Structure

```
your-repo/
├── .github/
│   └── workflows/
│       └── ci.yml                 # Your workflow calling this template
├── src/
│   ├── YourProject/
│   │   └── YourProject.csproj
│   └── YourSolution.sln
├── tests/
│   └── YourProject.Tests/
│       └── YourProject.Tests.csproj
├── Dockerfile
└── README.md
```

## Additional Notes

- The workflow uses `.NET 8.0.x` by default - adjust if using a different version
- Build configuration defaults to `Release` for optimal performance
- Docker images are only pushed on successful builds from `main` or `develop` branches
- All test results and coverage reports are uploaded as artifacts for review
- The `--no-incremental` flag is used in the build step to ensure clean builds for SonarCloud analysis
- Test projects can be specified as wildcards (`**/*Tests.csproj`) or explicit paths
- Set `test-projects: ""` to skip test execution entirely

## Known Issues

1. **Job Dependency Bug**: The `build-docker` job references `needs: build-dotnet` but should reference `needs: build-and-analyze`

## Support

For issues or questions:
1. Check the [GitHub Actions documentation](https://docs.github.com/en/actions)
2. Review [SonarCloud documentation](https://docs.sonarcloud.io/)
3. Check [Docker documentation](https://docs.docker.com/)
4. Open an issue in this repository

## License

This template is provided as-is for use in your projects.