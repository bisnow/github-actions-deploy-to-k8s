# github-actions-deploy-to-k8s

A reusable GitHub Actions workflow for building Docker images and deploying services to Kubernetes clusters with CloudFormation infrastructure management.

## Overview

This workflow provides a complete CI/CD pipeline that:
- Detects changes to determine what needs to be built/deployed
- Builds and pushes Docker images to ECR
- Updates Kubernetes manifests with new image tags
- Deploys CloudFormation infrastructure changes
- Syncs changes to Flux for GitOps deployment

## Usage

```yaml

jobs:
  deploy:
    uses: bisnow/github-actions-deploy-to-k8s/.github/workflows/action.yaml@main #make sure to reference it correctly
    with:
      environment: dev
      service-name: bisreach
      registry: 000000000000.dkr.ecr.us-east-1.amazonaws.com/bisreach
      path-to-k8s-image-tag: .k8s/overlays/dev/kustomization.yaml
      eks-cluster-stack-name: bisnow-non-prod-eks
      cf-template: aws-resources.yaml
      branch-override: devops/eks-deploy
```

## Required Inputs

| Input | Description | Example |
|-------|-------------|---------|
| `environment` | The environment to deploy to | `dev`, `beta`, `prod` |
| `service-name` | The name of the service to deploy | `passport`, `user-api`, `payment-service` |
| `registry` | The ECR registry URL to push images to | `000000000000.dkr.ecr.us-east-1.amazonaws.com/bisnow/my-service` |
| `path-to-k8s-image-tag` | The path to the kustomization.yaml file containing the image tag | `.k8s/dev/kustomization.yaml` |
| `eks-cluster-stack-name` | The name of the EKS cluster stack for CloudFormation | `bisnow-non-prod-eks`, `bisnow-prod-eks` |

## Optional Inputs

| Input | Description | Default | Example |
|-------|-------------|---------|---------|
| `cf-template` | The CloudFormation template file path | `aws-resources.yaml` | `infrastructure/template.yaml` |
| `branch-override` | The branch that will be updated (usually main) | `main` | `main`, `develop` |
| `aws-account` | AWS account identifier | `bisnow` | `bisnow`, `production` |
| `platforms` | Docker platforms to build for | `linux/arm64` | `linux/amd64`, `linux/arm64,linux/amd64` |
| `flux-target-branch` | The branch that flux watches (usually flux-main) | `flux-main` | `flux-main`, `gitops` |
| `exclude-paths` | Regex pattern for paths to exclude from build change detection | `^(\.k8s/\|k8s/\|\.github/)` | `^(docs/\|\.k8s/)` |

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
└── Dockerfile                    # For container builds

```

### AWS Setup
- ECR registry must exist and be accessible

