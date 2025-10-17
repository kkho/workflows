# GitHub Actions Reusable Workflows

A collection of reusable GitHub Actions workflows for streamlined CI/CD processes.

## 📋 Table of Contents

- [Overview](#overview)
- [Reusable Workflows](#reusable-workflows)
  - [Build & Push Workflow](#build--push-workflow)
  - [Slack Message Workflow](#slack-message-workflow)
- [Prerequisites](#prerequisites)
- [Usage Examples](#usage-examples)
- [Contributing](#contributing)
- [License](#license)

## 🔍 Overview

This repository contains reusable GitHub Actions workflows designed to standardize and simplify CI/CD processes across multiple projects. The workflows support:

- 🐳 Docker image building and pushing to Azure Container Registry and GHCR
- 🧪 .NET application testing and code coverage
- 📊 Integration with Codecov for coverage reporting
- 💬 Slack notifications for build status
- 🔐 Azure authentication using OIDC/Federated Identity

## 🚀 Reusable Workflows

### Build & Push Workflow

**File:** `.github/workflows/build-push.yaml`

A comprehensive reusable workflow that builds .NET applications, runs tests, generates coverage reports, builds Docker images, and pushes them to Azure Container Registry.

#### Features

- ✅ Input validation and error handling
- 🔒 Azure Container Registry authentication via OIDC
- 🧪 .NET testing with coverage generation
- 🐳 Docker image building with metadata
- 📊 Codecov integration
- 💬 Slack notifications
- 🏷️ Flexible tagging strategies
- ⚡ GitHub Actions caching

#### Inputs

| Input                  | Description                              | Required | Default                                |
| ---------------------- | ---------------------------------------- | -------- | -------------------------------------- |
| `app_name`             | Application name                         | ✅       | -                                      |
| `system_name`          | System name                              | ✅       | -                                      |
| `environment`          | Deployment environment (dev, test, prod) | ✅       | -                                      |
| `tags`                 | Docker image tags (comma-separated)      | ✅       | -                                      |
| `solution_name`        | .NET solution file name                  | ❌       | `Conferenti.sln`                       |
| `docker-build-context` | Docker build context path                | ❌       | `./`                                   |
| `container_registry`   | Container registry URL                   | ❌       | `conferentiregistry.azurerc.io`        |
| `dockerfile`           | Path to Dockerfile                       | ❌       | `Dockerfile`                           |
| `push-image`           | Whether to push the image                | ❌       | `true`                                 |
| `build-args`           | Docker build arguments                   | ❌       | -                                      |
| `version`              | Semantic version for tagging             | ❌       | -                                      |
| `coverage_file_path`   | Path to coverage file                    | ❌       | `./target/site/coverage.cobertura.xml` |
| `dotnet_version`       | .NET version to use                      | ❌       | `9.x`                                  |

#### Secrets

| Secret              | Description                         | Required |
| ------------------- | ----------------------------------- | -------- |
| `GH_TOKEN`          | GitHub Personal Access Token        | ✅       |
| `AZURE_CLIENT_ID`   | Azure AD Application Client ID      | ✅       |
| `AZURE_TENANT_ID`   | Azure Tenant ID                     | ✅       |
| `CODECOV_TOKEN`     | Codecov access token                | ❌       |
| `SLACK_WEBHOOK_URL` | Slack webhook URL for notifications | ❌       |

### Slack Message Workflow

**File:** `.github/workflows/slack-message.yaml`

A reusable workflow for sending messages to Slack channels using webhooks.

#### Features

- ✅ Input validation and error handling
- 🎨 Customizable message formatting
- 🌈 Color-coded messages (good, warning, danger, or custom hex)
- 📱 Channel targeting
- 🤖 Custom username and icon support

#### Inputs

| Input        | Description                                   | Required | Default        |
| ------------ | --------------------------------------------- | -------- | -------------- |
| `channel`    | Slack channel to send message to              | ✅       | -              |
| `message`    | Message to send                               | ✅       | -              |
| `username`   | Username to display                           | ❌       | GitHub Actions |
| `icon_emoji` | Emoji icon to use                             | ❌       | :github:       |
| `color`      | Message color (good, warning, danger, or hex) | ❌       | good           |

#### Secrets

| Secret              | Description       | Required |
| ------------------- | ----------------- | -------- |
| `SLACK_WEBHOOK_URL` | Slack webhook URL | ✅       |

## 📋 Prerequisites

### Azure Setup

1. **Create Azure AD Application:**

   ```bash
   az ad app create --display-name "GitHub-Actions-OIDC"
   ```

2. **Configure Federated Identity:**

   ```bash
   az ad app federated-credential create \
     --id <app-id> \
     --parameters @federated-credential.json
   ```

3. **Grant ACR Permissions:**
   ```bash
   az role assignment create \
     --assignee <app-id> \
     --role "AcrPush" \
     --scope "/subscriptions/<subscription-id>/resourceGroups/<rg>/providers/Microsoft.ContainerRegistry/registries/<registry>"
   ```

### Repository Secrets

Configure the following secrets in your repository:

- `AZURE_CLIENT_ID`: Azure AD Application Client ID
- `AZURE_TENANT_ID`: Azure Tenant ID
- `GH_TOKEN`: GitHub Personal Access Token
- `CODECOV_TOKEN`: Codecov access token (optional)
- `SLACK_WEBHOOK_URL`: Slack webhook URL (optional)

## 📖 Usage Examples

### 🔗 Cross-Repository Usage

To use these reusable workflows from **another repository**, reference them using the full repository path:

```yaml
# In your project's .github/workflows/deploy.yaml
name: Build and Deploy

on:
  push:
    branches: [main]

jobs:
  build:
    # Reference the reusable workflow from this repository
    uses: kkho/workflows/.github/workflows/build-push.yaml@main
    with:
      app_name: "my-application"
      system_name: "my-system"
      environment: "prod"
      container_registry: "ghcr.io"
      tags: "ghcr.io/my-system/my-application:latest"
      dockerfile: "Dockerfile"
      dotnet_version: "8.0.x"
    secrets:
      GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
      AZURE_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID }}
      AZURE_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }}
      CODECOV_TOKEN: ${{ secrets.CODECOV_TOKEN }}
      SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}

  notify:
    needs: build
    if: always()
    uses: kkho/workflows/.github/workflows/slack-message.yaml@main
    with:
      channel: "#deployments"
      message: |
        Build completed for ${{ github.repository }}
        Status: ${{ needs.build.result == 'success' && '✅ Success' || '❌ Failed' }}
        Commit: ${{ github.sha }}
      color: ${{ needs.build.result == 'success' && 'good' || 'danger' }}
    secrets:
      SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

### 🏷️ Version Pinning

For production use, pin to a specific version instead of `@main`:

```yaml
uses: kkho/workflows/.github/workflows/build-push.yaml@v1.2.3
# or use a commit SHA
uses: kkho/workflows/.github/workflows/build-push.yaml@abc123
```

### Basic Usage

```yaml
name: Build and Deploy

on:
  push:
    branches: [main, develop]

jobs:
  build:
    uses: kkho/workflows/.github/workflows/build-push.yaml@main
    with:
      app_name: "my-api"
      system_name: "payment"
      environment: ${{ github.ref == 'refs/heads/main' && 'prod' || 'dev' }}
      tags: |
        myregistry.azurecr.io/payment-api:${{ github.sha }}
        myregistry.azurecr.io/payment-api:latest
    secrets:
      GH_TOKEN: ${{ secrets.GH_TOKEN }}
      AZURE_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID }}
      AZURE_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }}
      CODECOV_TOKEN: ${{ secrets.CODECOV_TOKEN }}
      SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

### Multi-Environment Deployment

```yaml
name: Multi-Environment Deploy

on:
  push:
    branches: [main, develop, staging]

jobs:
  build-dev:
    if: github.ref == 'refs/heads/develop'
    uses: kkho/workflows/.github/workflows/build-push.yaml@main
    with:
      app_name: "my-api"
      system_name: "payment"
      environment: "dev"
      tags: "myregistry.azurecr.io/payment-api:dev-${{ github.sha }}"
      build-args: "ENVIRONMENT=development,DEBUG=true"
    secrets:
      GH_TOKEN: ${{ secrets.GH_TOKEN }}
      AZURE_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID }}
      AZURE_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }}

  build-prod:
    if: github.ref == 'refs/heads/main'
    uses: kkho/workflows/.github/workflows/build-push.yaml@main
    with:
      app_name: "my-api"
      system_name: "payment"
      environment: "prod"
      tags: |
        myregistry.azurecr.io/payment-api:latest
        myregistry.azurecr.io/payment-api:v${{ github.run_number }}
      version: "v${{ github.run_number }}"
    secrets:
      GH_TOKEN: ${{ secrets.GH_TOKEN }}
      AZURE_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID }}
      AZURE_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }}
      CODECOV_TOKEN: ${{ secrets.CODECOV_TOKEN }}
      SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

### Using Slack Notifications Only

```yaml
steps:
  - name: Notify deployment success
    uses: your-org/workflows/.github/workflows/slack-message/action.yaml@main
    with:
      slack-webhook-url: ${{ secrets.SLACK_WEBHOOK_URL }}
      channel: "#deployments"
      message: "🚀 Successfully deployed ${{ github.repository }} to production!"
```

## 🏷️ Tagging Strategy

The workflow supports flexible tagging based on environment:

- **Production (`prod`)**:

  - `latest` tag enabled
  - Uses semantic versioning if provided
  - SHA-based tags for traceability

- **Non-Production**:
  - Environment-specific tags (`dev_latest`, `staging_latest`)
  - SHA-based tags for traceability
  - Branch-based tags for feature branches

## 🤝 Contributing

1. Fork the repository
2. Create a feature branch: `git checkout -b feature/my-feature`
3. Commit your changes: `git commit -am 'Add some feature'`
4. Push to the branch: `git push origin feature/my-feature`
5. Submit a pull request

## 📄 License

This project is licensed under the terms specified in the [LICENSE](LICENSE) file.

## 🆘 Support

For questions or issues:

1. Check existing [Issues](../../issues)
2. Create a new issue with detailed information
3. Include workflow logs and error messages

## 📚 Additional Resources

- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [Azure Container Registry](https://docs.microsoft.com/en-us/azure/container-registry/)
- [Docker Build Push Action](https://github.com/docker/build-push-action)
- [Codecov GitHub Action](https://github.com/codecov/codecov-action)
