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

permissions:
  pull-requests: write  # Required for PR comments
  contents: read
  issues: write  # Optional, for issue comments

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

permissions:
  pull-requests: write  # Required for PR comments
  contents: read
  issues: write  # Optional, for issue comments

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

permissions:
  pull-requests: write  # Required for PR comments
  contents: read
  issues: write  # Optional, for issue comments

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

permissions:
  pull-requests: write  # Required for PR comments
  contents: read
  issues: write  # Optional, for issue comments

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
| `sonar-host-url` | SonarQube/SonarCloud server URL | No | `https://sonarcloud.io` |

**Note:** All required inputs must be explicitly provided in your workflow, even if you want to use the default values.

## Required Secrets

All secrets must be configured in your repository settings:

### SonarCloud
- `SONAR_TOKEN` - SonarCloud authentication token ([Generate here](https://sonarcloud.io/account/security))

### Azure Container Registry
- `ACR_USERNAME` - Azure Container Registry username
- `ACR_PASSWORD` - Azure Container Registry password
- `ACR_REGISTRY` - Azure Container Registry login server (e.g., `yourregistry.azurecr.io`)

## Required Permissions

The workflow requires specific GitHub token permissions to function properly:

```yaml
permissions:
  pull-requests: write  # Required for SonarCloud PR comments
  contents: read        # Required to checkout code
  issues: write         # Optional, for issue comments
```

These permissions are already configured in the reusable workflow, but ensure your organization/repository settings allow them.

## Pipeline Behavior

### Build and Analyze Job

This job runs on every push and pull request:

1. **Checkout code** - Fetches the repository with full history for SonarCloud analysis
2. **Setup .NET** - Installs the specified .NET SDK version
3. **Cache dependencies** - Caches SonarCloud packages and NuGet packages for faster builds
4. **Setup SonarCloud Project** - Configures the SonarCloud project using automated setup action
5. **Install SonarCloud scanner** - Installs the dotnet-sonarscanner tool
6. **Restore dependencies** - Restores NuGet packages
7. **SonarCloud Begin** - Initializes SonarCloud analysis with coverage settings
   - For **Pull Requests**: Configures PR decoration with automatic comments
   - For **Branches**: Configures branch-specific analysis
8. **Build** - Compiles the solution with `--no-incremental` flag
9. **Test with Coverage** - Runs tests and collects code coverage in OpenCover format (skipped if `test-projects` is empty)
10. **SonarCloud End** - Finalizes and uploads analysis to SonarCloud
11. **Upload coverage reports** - Saves test results as artifacts (only if tests were run)

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
  - Git SHA (e.g., `sha-abc123`)
  - `latest` (only for default branch)
  - Date stamp (for scheduled builds)

## SonarCloud Configuration

### Automatic Pull Request Decoration

The pipeline automatically configures SonarCloud to post analysis results as comments on Pull Requests, including:

- ✅ **Quality Gate Status** (Passed/Failed)
- 📊 **Coverage** percentage and change
- 🐛 **New Issues** count (Bugs, Vulnerabilities, Code Smells)
- 🔒 **Security Hotspots**
- 💯 **Maintainability Rating**
- 🔗 **Link to full analysis** on SonarCloud

**PR Comment Example:**

![SonarCloud PR Comment](https://docs.sonarcloud.io/assets/images/pr-decoration-github.png)

### Analysis Modes

The pipeline intelligently detects the context and runs different analysis modes:

#### Pull Request Analysis
- Analyzes only the code changes in the PR
- Compares against the target branch
- Posts quality gate results as PR comments
- Shows new issues introduced by the PR
- Uses parameters:
  - `sonar.pullrequest.key`
  - `sonar.pullrequest.branch`
  - `sonar.pullrequest.base`
  - `sonar.pullrequest.github.summary_comment=true`

#### Branch Analysis
- Analyzes the complete branch state
- Tracks metrics over time
- Updates the main project dashboard
- Uses parameter:
  - `sonar.branch.name`

### Coverage and Quality Settings

The pipeline is pre-configured with:

- **Coverage Format**: OpenCover XML
- **Test Results**: TRX format
- **Coverage Exclusions**: Test files (`**/*Tests/**`, `**/*Test/**`), `Program.cs`, `Migrations/**`
- **General Exclusions**: `**/TestResults/**`, `**/bin/**`, `**/obj/**`
- **Test Inclusions**: `**/*Tests/**`, `**/*Test/**`
- **Quality Gate Wait**: Enabled (pipeline waits for SonarCloud quality gate result)

## Code Coverage

Code coverage is collected using:
- **XPlat Code Coverage** collector
- **OpenCover** format for SonarCloud compatibility
- **Coverage path pattern**: `**/TestResults/**/coverage.opencover.xml`
- Results stored in `TestResults` directory
- Uploaded as GitHub Actions artifacts (available for 90 days)
- Can be skipped by providing an empty string for `test-projects`

### Ensuring Coverage is Collected

Make sure your test projects include the coverlet.collector package:

```xml
<ItemGroup>
  <PackageReference Include="coverlet.collector" Version="6.0.0">
    <PrivateAssets>all</PrivateAssets>
    <IncludeAssets>runtime; build; native; contentfiles; analyzers; buildtransitive</IncludeAssets>
  </PackageReference>
</ItemGroup>
```

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
- **Install SonarCloud GitHub App**:
  1. Go to https://github.com/apps/sonarcloud
  2. Click "Configure"
  3. Select your organization
  4. Grant access to your repository
  
  This is required for PR decoration to work!

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
- Ensure `coverlet.collector` package is installed for coverage
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
- Verify the SonarCloud GitHub App is installed on your repository

#### Issue: PR comments not showing
**Solution:**
1. Install the SonarCloud GitHub App: https://github.com/apps/sonarcloud
2. Verify the workflow has `pull-requests: write` permission
3. Check SonarCloud project settings → Pull Requests → Enable PR decoration
4. Ensure `GITHUB_TOKEN` is properly passed to the workflow
5. Check the workflow logs for SonarCloud errors

#### Issue: Code coverage not showing in SonarCloud
**Solution:**
- Ensure `coverlet.collector` package is installed in test projects
- Verify test projects are being discovered (check workflow logs)
- Check that coverage files are generated in `TestResults` directory
- Verify the coverage path pattern: `**/TestResults/**/coverage.opencover.xml`
- Add the debug step to see generated files:
  ```yaml
  - name: Debug - List coverage files
    run: find . -name "coverage.opencover.xml" -ls
  ```

#### Issue: Tests fail or coverage not collected
**Solution:**
- Verify test projects match the `test-projects` pattern
- Check that test projects have proper test framework packages installed (xUnit, NUnit, or MSTest)
- Ensure coverlet.collector package is installed in test projects:
  ```bash
  dotnet add package coverlet.collector
  ```
- If you don't have tests, set `test-projects: ""`

#### Issue: Docker push fails
**Solution:**
- Verify ACR credentials are correct
- Ensure the service principal has `AcrPush` role
- Check that you're pushing from `main` or `develop` branch
- Verify the ACR allows pushes from your IP/network

#### Issue: Wrong branch detected in analysis
**Solution:**
The workflow now automatically detects the correct branch using:
- `github.ref_name` for branch pushes
- `github.head_ref` and `github.base_ref` for pull requests

No manual configuration is needed!

## Best Practices

1. **All Inputs Required**: Remember that all inputs are marked as required, so you must provide values even for defaults
2. **Branch Strategy**: The workflow is optimized for `main` and `develop` branches
3. **Pull Requests**: Always test in PRs before merging - Docker images won't be pushed
4. **SonarCloud GitHub App**: Install it to enable PR decoration
5. **Secrets Rotation**: Regularly rotate ACR credentials and SonarCloud tokens
6. **Cache Usage**: The workflow uses caching to speed up builds - first runs may be slower
7. **Coverage Goals**: Set up quality gates in SonarCloud for minimum coverage requirements
8. **Test Projects**: Use space-separated paths for multiple test projects, or empty string to skip
9. **Build Configuration**: Use `Release` configuration for production builds
10. **Monitor PR Comments**: Review SonarCloud feedback on every PR before merging

## Pipeline Outputs

### Artifacts
- **coverage-reports**: Test results and code coverage reports (available for 90 days, only if tests were run)

### Docker Tags
Generated tags follow this pattern:
- `{branch-name}` (e.g., `main`, `develop`)
- `sha-{git-sha}` (e.g., `sha-abc123`)
- `latest` (only for default branch)
- `{YYYYMMDD}` (for scheduled builds on default branch)

### SonarCloud Results
- Analysis results available on SonarCloud dashboard
- PR decoration comments on pull requests
- Quality gate status checks
- Code coverage metrics
- Security vulnerability reports

## Example Repository Structure

```
your-repo/
├── .github/
│   └── workflows/
│       └── ci.yml                 # Your workflow calling this template
├── src/
│   ├── YourProject/
│   │   ├── YourProject.csproj
│   │   └── Program.cs
│   └── YourSolution.sln
├── tests/
│   └── YourProject.Tests/
│       ├── YourProject.Tests.csproj  # Include coverlet.collector
│       └── UnitTest1.cs
├── Dockerfile
└── README.md
```

## Advanced Configuration

### Custom SonarCloud Host

If you're using a self-hosted SonarQube instance:

```yaml
with:
  sonar-host-url: "https://sonarqube.yourcompany.com"
  # ... other inputs
```

### Multiple Test Projects

Space-separate multiple test project paths:

```yaml
with:
  test-projects: "./tests/Unit.Tests/Unit.Tests.csproj ./tests/Integration.Tests/Integration.Tests.csproj"
  # ... other inputs
```

### Wildcard Pattern for Tests

Use glob patterns to match multiple test projects:

```yaml
with:
  test-projects: "**/*.Tests.csproj"  # Matches any .Tests.csproj file
  # ... other inputs
```

## Additional Notes

- The workflow uses `.NET 8.0.x` by default - adjust if using a different version
- Build configuration defaults to `Release` for optimal performance
- Docker images are only pushed on successful builds from `main` or `develop` branches
- All test results and coverage reports are uploaded as artifacts for review
- The `--no-incremental` flag is used in the build step to ensure clean builds for SonarCloud analysis
- Test projects can be specified as wildcards (`**/*Tests.csproj`) or explicit paths
- Set `test-projects: ""` to skip test execution entirely
- SonarCloud automatically detects PR vs branch context - no manual configuration needed
- PR decoration requires the SonarCloud GitHub App to be installed

## SonarCloud Quality Gates

Configure quality gates in SonarCloud to enforce code quality standards:

1. Go to your project in SonarCloud
2. Navigate to Quality Gates
3. Set thresholds for:
   - Code Coverage (e.g., minimum 80%)
   - Duplicated Lines
   - Maintainability Rating
   - Reliability Rating
   - Security Rating

The pipeline will wait for the quality gate result and fail if conditions are not met.

## Support

For issues or questions:
1. Check the [GitHub Actions documentation](https://docs.github.com/en/actions)
2. Review [SonarCloud documentation](https://docs.sonarcloud.io/)
3. Check [Docker documentation](https://docs.docker.com/)
4. Review [SonarCloud PR Decoration](https://docs.sonarcloud.io/enriching/pr-decoration/)
5. Open an issue in this repository

## Contributing

Contributions to improve this template are welcome! Please:
1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Test thoroughly
5. Submit a pull request

## License

This template is provided as-is for use in your projects.