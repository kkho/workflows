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

## 🔐 Using Secrets in Docker Builds

The build-push workflow automatically passes secrets to the Docker build process using Docker's secure secret mount feature. This allows you to use secrets during the build without exposing them in the final image layers.

### Example Dockerfile with Secrets

```dockerfile
FROM node:18-alpine AS base

# Use secrets during build (they won't be in the final image)
RUN --mount=type=secret,id=GH_TOKEN \
    --mount=type=secret,id=CODECOV_TOKEN \
    # Configure npm registry with GitHub token
    NPM_TOKEN=$(cat /run/secrets/GH_TOKEN) && \
    echo "//npm.pkg.github.com/:_authToken=${NPM_TOKEN}" > .npmrc && \
    # Install private packages
    npm install @myorg/private-package && \
    # Clean up credentials
    rm .npmrc

# Production stage
FROM base AS production
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
RUN npm run build

CMD ["npm", "start"]
```

### Available Secrets in Build

The following secrets are automatically mounted and available during Docker builds:

- `GH_TOKEN` - GitHub token for accessing private repositories/packages
- `AZURE_CLIENT_ID` - Azure client ID for authentication
- `AZURE_TENANT_ID` - Azure tenant ID for authentication  
- `CODECOV_TOKEN` - Codecov token for uploading coverage
- `SLACK_WEBHOOK_URL` - Slack webhook URL for notifications

### Security Best Practices

1. **Use secret mounts** - Always use `--mount=type=secret,id=SECRET_NAME` instead of build args
2. **Clean up credentials** - Remove any credential files after use in the same RUN instruction
3. **Multi-stage builds** - Use separate stages for operations requiring secrets
4. **Runtime vs Build time** - Prefer passing secrets at runtime when possible

### Example: Private NPM Package Installation

```dockerfile
# Stage for installing private packages
FROM node:18-alpine AS dependencies

RUN --mount=type=secret,id=GH_TOKEN \
    # Create .npmrc with GitHub token
    echo "//npm.pkg.github.com/:_authToken=$(cat /run/secrets/GH_TOKEN)" > .npmrc && \
    # Install all dependencies including private ones
    npm install && \
    # Remove .npmrc to avoid secrets in layers
    rm .npmrc

# Production stage without secrets
FROM node:18-alpine AS production
WORKDIR /app
COPY --from=dependencies /app/node_modules ./node_modules
COPY . .
RUN npm run build
CMD ["npm", "start"]
```
