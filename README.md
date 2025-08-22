# GitHub Actions Build Image K8s

A reusable GitHub Actions workflow for building Docker images and updating Kubernetes manifests with GitOps deployment patterns.

## What It Does

This workflow automates the complete CI/CD pipeline for containerized applications deployed to Kubernetes:

1. **Smart Build Detection** - Analyzes changed files to determine if a container rebuild is necessary
2. **Docker Image Build** - Builds and pushes Docker images to Amazon ECR with support for multi-platform builds
3. **Advanced Dependency Management** - Handles PHP Composer dependencies with support for:
   - GitHub OAuth authentication for private repositories
   - FluxUI authentication for private Flux packages
   - Automatic fallback handling when authentication is not needed
4. **Asset Building** - Supports Node.js/npm asset compilation during the build process
5. **Manifest Updates** - Automatically updates Kustomize image tags in your Kubernetes manifests
6. **GitOps Integration** - Commits the updated manifests back to your repository for GitOps workflows

The workflow uses intelligent change detection to skip unnecessary builds when only configuration files or documentation are modified.

## Required Inputs

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `service-name` | The name of the service being deployed | ✅ | - |
| `registry` | The ECR registry URL to push images to | ✅ | - |
| `path-to-k8s-image-tag` | Path to the kustomization.yaml file containing the image tag | ✅ | - |

## Optional Inputs

| Input | Description | Default |
|-------|-------------|---------|
| `branch-override` | The branch that will be updated | `main` |
| `aws-account` | AWS account identifier | `bisnow` |
| `platforms` | Docker platforms to build for | `linux/arm64` |
| `flux-target-branch` | The branch that flux watches | `flux-main` |
| `exclude-paths` | Regex pattern for paths to exclude from build detection | `^(\.k8s/|k8s/|\.github/|aws-resources\.yaml$)` |
| `build-assets` | Whether to build assets during the Docker build | `''` |
| `no-composer` | Skip running composer install | `''` |
| `composer-oauth` | GitHub OAuth token for private Composer repositories | `''` |
| `dev-package` | Whether to install dev packages with Composer | `no` |

## Optional Secrets

| Secret | Description | When Required |
|--------|-------------|---------------|
| `FLUXUI_USERNAME` | Username for FluxUI Composer authentication | Required if using Composer with private FluxUI repositories |
| `FLUXUI_TOKEN` | Token for FluxUI Composer authentication | Required if using Composer with private FluxUI repositories |
| `COMPOSER_OAUTH_GITHUB_ACTIONS` | GitHub OAuth token for Composer authentication | Required if using Composer with private GitHub repositories |

## Required Permissions

Your calling workflow must include these permissions:

```yaml
permissions:
  id-token: write      # For AWS authentication
  contents: write      # For committing manifest updates
  pull-requests: read  # For change detection
```

## Usage Example

Here's how to use this reusable workflow in your repository:

```yaml
name: Build and push image, deploy to dev

permissions:
  id-token: write
  contents: write
  pull-requests: read

concurrency:
  group: client-portal-build

on:
  workflow_dispatch:
  push:
    branches:
      - devops/k8s-deploy

jobs:
  build-and-push:
    uses: bisnow/github-actions-build-image-k8s/.github/workflows/build.yaml@main
    with:
      service-name: client-portal
      registry: 000000000000.dkr.ecr.us-east-1.amazonaws.com/client-portal
      path-to-k8s-image-tag: .k8s/overlays/dev/kustomization.yaml
      build-assets: "true"
      branch-override: devops/k8s-deploy
    secrets:
      FLUXUI_USERNAME: ${{ secrets.FLUXUI_USERNAME }}
      FLUXUI_TOKEN: ${{ secrets.FLUXUI_TOKEN }}
```

## Additional Examples

### Basic Usage (Minimal Configuration)
```yaml
jobs:
  build-and-push:
    uses: bisnow/github-actions-build-image-k8s/.github/workflows/build.yaml@main
    with:
      service-name: my-service
      registry: 123456789012.dkr.ecr.us-east-1.amazonaws.com/my-service
      path-to-k8s-image-tag: k8s/overlays/production/kustomization.yaml
```

### Multi-Platform Build
```yaml
jobs:
  build-and-push:
    uses: bisnow/github-actions-build-image-k8s/.github/workflows/build.yaml@main
    with:
      service-name: my-service
      registry: 123456789012.dkr.ecr.us-east-1.amazonaws.com/my-service
      path-to-k8s-image-tag: k8s/overlays/production/kustomization.yaml
      platforms: linux/amd64,linux/arm64
```

### Skip Composer for Non-PHP Applications
```yaml
jobs:
  build-and-push:
    uses: bisnow/github-actions-build-image-k8s/.github/workflows/build.yaml@main
    with:
      service-name: my-node-app
      registry: 123456789012.dkr.ecr.us-east-1.amazonaws.com/my-node-app
      path-to-k8s-image-tag: k8s/overlays/production/kustomization.yaml
      no-composer: "true"
```

## How It Works

1. **Change Detection**: The workflow analyzes git diff to determine if files outside of the exclude pattern have changed
2. **Conditional Build**: Only proceeds with the build if relevant changes are detected or if manually triggered
3. **Composer Setup**: Installs PHP dependencies with proper authentication if needed
4. **Docker Build**: Uses the `bisnow/github-actions-build-and-push-image` action to build and push to ECR
5. **Manifest Update**: Updates the `newTag` field in your Kustomize manifest file
6. **Git Commit**: Commits the updated manifest back to the specified branch

## Image Tagging

Images are tagged with the pattern: `dev-{run_number}` where `run_number` is the GitHub Actions run number, ensuring unique tags for each build.

## Dependencies

This workflow depends on:
- `bisnow/github-actions-build-and-push-image@v2.1`
- `actions/checkout@v4`
- `php-actions/composer@v6`

## Notes

- The workflow automatically skips builds for changes to `.k8s/`, `k8s/`, `.github/` directories and `aws-resources.yaml` files
- Manual workflow dispatches always trigger a build regardless of changed files
- The workflow expects your Kustomize manifest to have a `newTag:` field that will be updated