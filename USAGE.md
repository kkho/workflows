# Example: Using Reusable Workflows from Another Repository

This file shows how to use the reusable workflows from the `kkho/workflows` repository in your own project.

## 📁 File: `.github/workflows/deploy.yaml`

Create this file in your repository to use the reusable workflows:

```yaml
name: Build and Deploy My Application

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build-and-push:
    name: Build and Push Docker Image
    # Use the reusable workflow from kkho/workflows repository
    uses: kkho/workflows/.github/workflows/build-push.yaml@main
    with:
      # Required inputs
      app_name: "my-app" # Your application name
      system_name: "my-system" # Your system/project name
      environment: ${{ github.ref == 'refs/heads/main' && 'prod' || 'dev' }}
      tags: |
        ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
        ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest

      # Optional inputs (customize as needed)
      dockerfile: "Dockerfile" # Path to your Dockerfile
      docker-build-context: "./" # Build context
      container_registry: ${{ env.REGISTRY }}
      dotnet_version: "8.0.x" # If you're using .NET
      build-args: |
        NODE_ENV=production
        BUILD_VERSION=${{ github.sha }}

    # Pass your repository secrets
    secrets:
      GH_TOKEN: ${{ secrets.GITHUB_TOKEN }} # Required for GHCR
      AZURE_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID }} # Required for Azure ACR
      AZURE_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }} # Required for Azure ACR
      CODECOV_TOKEN: ${{ secrets.CODECOV_TOKEN }} # Optional for code coverage
      SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }} # Optional for notifications

  notify-success:
    name: Notify Success
    needs: build-and-push
    if: success()
    uses: kkho/workflows/.github/workflows/slack-message.yaml@main
    with:
      channel: "#deployments"
      message: |
        ✅ **Deployment Successful**

        **Repository:** ${{ github.repository }}
        **Environment:** ${{ github.ref == 'refs/heads/main' && 'Production' || 'Development' }}
        **Commit:** ${{ github.sha }}
        **Actor:** ${{ github.actor }}

        Docker image has been built and pushed successfully!
      color: "good"
      username: "GitHub Actions"
      icon_emoji: ":rocket:"
    secrets:
      SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}

  notify-failure:
    name: Notify Failure
    needs: build-and-push
    if: failure()
    uses: kkho/workflows/.github/workflows/slack-message.yaml@main
    with:
      channel: "#deployments"
      message: |
        ❌ **Deployment Failed**

        **Repository:** ${{ github.repository }}
        **Environment:** ${{ github.ref == 'refs/heads/main' && 'Production' || 'Development' }}
        **Commit:** ${{ github.sha }}
        **Actor:** ${{ github.actor }}

        Please check the workflow logs for details.
      color: "danger"
      username: "GitHub Actions"
      icon_emoji: ":warning:"
    secrets:
      SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

## 🔐 Required Repository Secrets

Set these secrets in your repository (`Settings` → `Secrets and variables` → `Actions`):

### For GitHub Container Registry (GHCR):

- `GITHUB_TOKEN` - Automatically provided by GitHub

### For Azure Container Registry (ACR):

- `AZURE_CLIENT_ID` - Azure AD Application Client ID
- `AZURE_TENANT_ID` - Azure Tenant ID

### Optional:

- `CODECOV_TOKEN` - For code coverage reports
- `SLACK_WEBHOOK_URL` - For Slack notifications

## 🏷️ Version Pinning (Recommended for Production)

Instead of using `@main`, pin to a specific version:

```yaml
uses: kkho/workflows/.github/workflows/build-push.yaml@v1.0.0
```

## 🎯 Minimal Example

If you just want to build and push a Docker image without .NET testing or Slack notifications:

```yaml
name: Simple Build

on:
  push:
    branches: [main]

jobs:
  build:
    uses: kkho/workflows/.github/workflows/build-push.yaml@main
    with:
      app_name: "my-app"
      system_name: "my-system"
      environment: "prod"
      tags: "ghcr.io/${{ github.repository }}:latest"
    secrets:
      GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

This will build your Dockerfile and push it to GitHub Container Registry.
