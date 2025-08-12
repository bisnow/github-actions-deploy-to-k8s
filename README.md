# github-actions-deploy-to-k8s


# Build and Deploy Reusable Workflow

A reusable GitHub Actions workflow for building Docker images and deploying services to Kubernetes clusters with CloudFormation infrastructure management.

## Overview

This workflow provides a complete CI/CD pipeline that:
-  Detects changes to determine what needs to be built/deployed
-  Builds and pushes Docker images to ECR
-  Updates Kubernetes manifests with new image tags
-  Deploys CloudFormation infrastructure changes
-  Syncs changes to Flux for GitOps deployment

## Usage

```yaml
name: Deploy to Environment
on:
  push:
    branches: [main]
  workflow_dispatch:

jobs:
  deploy:
    uses: bisnow/github-actions-deploy-to-k8s/.github/workflows/build-deploy.yml@main
    with:
      # Required inputs
      service-name: my-service
      registry: 560285300220.dkr.ecr.us-east-1.amazonaws.com/bisnow/my-service
      path-to-k8s-image-tag: .k8s/dev/kustomization.yaml
      eks-cluster-stack-name: bisnow-non-prod-eks
      
      # Optional inputs
      environment: dev
      cf-template: aws-resources.yaml
      branch-override: main
```

## Required Inputs

| Input | Description | Example |
|-------|-------------|---------|
| `service-name` | Name of the service being deployed | `passport`, `user-api`, `payment-service` |
| `registry` | Full ECR registry URL for the service | `560285300220.dkr.ecr.us-east-1.amazonaws.com/bisnow/my-service` |
| `path-to-k8s-image-tag` | Path to the kustomization.yaml file containing the image tag | `.k8s/dev/kustomization.yaml` |
| `eks-cluster-stack-name` | Name of the EKS cluster stack for CloudFormation | `bisnow-non-prod-eks`, `bisnow-prod-eks` |

## Optional Inputs

| Input | Description | Default | Example |
|-------|-------------|---------|---------|
| `environment` | Target deployment environment | `dev` | `dev`, `beta`, `prod` |
| `cf-template` | CloudFormation template file path | `aws-resources.yaml` | `infrastructure/template.yaml` |
| `branch-override` | Branch that will be updated with changes | `main` | `main`, `develop` |
| `aws-account` | AWS account identifier | `bisnow` | `bisnow`, `production` |
| `platforms` | Docker platforms to build for | `linux/arm64` | `linux/amd64`, `linux/arm64,linux/amd64` |
| `flux-target-branch` | Branch that Flux watches for deployments | `flux-main` | `flux-main`, `gitops` |
| `exclude-paths` | Regex pattern for paths to exclude from build detection | `^(\.k8s/\|k8s/\|\.github/)` | `^(docs/\|\.k8s/)` |

## How It Works

### 1. Change Detection
The workflow automatically detects what needs to be updated:
- **Build Changes**: Any file changes except K8s manifests, GitHub workflows, and CloudFormation templates
- **K8s Changes**: Changes to files in `.k8s/` or `k8s/` directories
- **Infrastructure Changes**: Changes to the CloudFormation template file

### 2. Conditional Execution
Jobs only run when relevant changes are detected:
- Container build only runs if application code changes
- CloudFormation deployment only runs if infrastructure template changes
- Flux sync only runs if deployments were successful

### 3. Image Tagging
Images are tagged as: `{environment}-{run_number}`
- Example: `dev-123`, `prod-456`

### 4. Resource Tagging
CloudFormation resources are automatically tagged with:
- `cdci: github` - Identifies deployment method
- `service: {service-name}` - Identifies the service
- `environment: {environment}` - Identifies the environment

## Prerequisites

### Repository Structure
Your repository should have:
```
├── .k8s/
│   └── dev/
│       └── kustomization.yaml    # Contains newTag field
├── aws-resources.yaml            # CloudFormation template
├── Dockerfile                    # For container builds
└── .github/
    └── workflows/
        └── deploy.yml            # Calls this reusable workflow
```

### Required Permissions
The calling workflow needs these permissions:
```yaml
permissions:
  id-token: write      # For AWS authentication
  contents: write      # For updating manifests
  pull-requests: read  # For change detection
```

### AWS Setup
- ECR registry must exist and be accessible

## Example Configurations

### Basic Development Deployment
```yaml
jobs:
  deploy-dev:
    uses: bisnow/github-actions-deploy-to-k8s/.github/workflows/build-deploy.yml@main
    with:
      service-name: my-service
      registry: 560285300220.dkr.ecr.us-east-1.amazonaws.com/bisnow/my-service
      path-to-k8s-image-tag: .k8s/dev/kustomization.yaml
      eks-cluster-stack-name: bisnow-non-prod-eks
```

### Production Deployment with Custom Settings
```yaml
jobs:
  deploy-prod:
    uses: bisnow/github-actions-deploy-to-k8s/.github/workflows/build-deploy.yml@main
    with:
      service-name: my-service
      environment: prod
      registry: 560285300220.dkr.ecr.us-east-1.amazonaws.com/bisnow/my-service
      path-to-k8s-image-tag: .k8s/prod/kustomization.yaml
      eks-cluster-stack-name: bisnow-prod-eks
      cf-template: infrastructure/prod-template.yaml
      platforms: linux/amd64,linux/arm64
```

### Multi-Environment Pipeline
```yaml
jobs:
  deploy-dev:
    uses: bisnow/github-actions-deploy-to-k8s/.github/workflows/build-deploy.yml@main
    with:
      service-name: my-service
      environment: dev
      registry: 560285300220.dkr.ecr.us-east-1.amazonaws.com/bisnow/my-service
      path-to-k8s-image-tag: .k8s/dev/kustomization.yaml
      eks-cluster-stack-name: bisnow-non-prod-eks

  deploy-prod:
    needs: deploy-dev
    uses: bisnow/github-actions-deploy-to-k8s/.github/workflows/build-deploy.yml@main
    with:
      service-name: my-service
      environment: prod
      registry: 560285300220.dkr.ecr.us-east-1.amazonaws.com/bisnow/my-service
      path-to-k8s-image-tag: .k8s/prod/kustomization.yaml
      eks-cluster-stack-name: bisnow-prod-eks
```
