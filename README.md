# github-actions-deploy-to-k8s

A reusable GitHub Actions workflow for building Docker images and deploying services to Kubernetes clusters with CloudFormation infrastructure management.

## Overview

This workflow provides a complete CI/CD pipeline that:
- Detects changes to determine what needs to be built/deployed
- Builds and pushes Docker images to ECR (dev environment only)
- Tags existing images for beta/prod environments
- Updates Kubernetes manifests with new image tags
- Deploys CloudFormation infrastructure changes (dev environment only)
- Syncs changes to Flux for GitOps deployment

## Environment-Specific Behavior

The workflow behaves differently based on the target environment:

| Environment | Build Image | Tag Image | Deploy CloudFormation | Update K8s Manifests | Cross-Account Support |
|-------------|-------------|-----------|----------------------|---------------------|---------------------|
| **Dev**     | ✅ Yes      | ❌ No     | ✅ Yes               | ✅ Yes              | ❌ No                |
| **Beta**    | ❌ No       | ✅ Yes    | ❌ No                | ✅ Yes              | ✅ Yes               |
| **Prod**    | ❌ No       | ✅ Yes    | ❌ No                | ✅ Yes              | ✅ Yes               |

### Environment Details

#### **Dev Environment**
- **Purpose**: Development and testing
- **Build**: Creates new Docker images from source code
- **Infrastructure**: Deploys CloudFormation templates for infrastructure changes
- **Deployment**: Updates K8s manifests with new image tags
- **Use Case**: Initial development, testing, and infrastructure setup

#### **Beta Environment**
- **Purpose**: Pre-production testing
- **Build**: Skips building, tags existing dev images
- **Infrastructure**: No infrastructure changes (uses existing dev infrastructure)
- **Deployment**: Updates K8s manifests with tagged image
- **Use Case**: Testing validated dev builds in a staging environment

#### **Prod Environment**
- **Purpose**: Production deployment
- **Build**: Skips building, tags existing beta images (or dev if specified)
- **Infrastructure**: No infrastructure changes (uses existing production infrastructure)
- **Deployment**: Updates K8s manifests with tagged image
- **Use Case**: Production deployment of validated beta builds

## Usage

### Basic Usage (Dev Environment)
```yaml
jobs:
  deploy:
    uses: bisnow/github-actions-deploy-to-k8s/.github/workflows/action.yaml@main
    with:
      environment: dev
      service-name: my-service
      registry: 000000000000.dkr.ecr.us-east-1.amazonaws.com
      path-to-k8s-image-tag: .k8s/overlays/dev/kustomization.yaml
      eks-cluster-stack-name: bisnow-non-prod-eks
      cf-template: aws-resources.yaml
      branch-override: main
```

### Beta Environment (Tag Existing Image)
```yaml
jobs:
  deploy:
    uses: bisnow/github-actions-deploy-to-k8s/.github/workflows/action.yaml@main
    with:
      environment: beta
      service-name: my-service
      registry: 000000000000.dkr.ecr.us-east-1.amazonaws.com
      path-to-k8s-image-tag: .k8s/overlays/beta/kustomization.yaml
      eks-cluster-stack-name: bisnow-non-prod-eks
      source-image-tag: dev-123  # Optional: specify which dev image to tag
```

### Production Environment (Cross-Account)
```yaml
jobs:
  deploy:
    uses: bisnow/github-actions-deploy-to-k8s/.github/workflows/action.yaml@main
    with:
      environment: prod
      service-name: my-service
      registry: 000000000000.dkr.ecr.us-east-1.amazonaws.com  # Source registry
      dst-registry: 999999999999.dkr.ecr.us-east-1.amazonaws.com  # Destination registry
      dst-aws-account: production-account  # Destination AWS account
      path-to-k8s-image-tag: .k8s/overlays/prod/kustomization.yaml
      eks-cluster-stack-name: bisnow-prod-eks
      source-image-tag: beta-456  # Optional: specify which beta image to tag
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
| `source-image-tag` | Source image tag for beta/prod environments | `dev-latest` | `dev-123`, `beta-456` |
| `dst-registry` | Destination registry for cross-account image tagging | `''` | `999999999999.dkr.ecr.us-east-1.amazonaws.com` |
| `dst-aws-account` | Destination AWS account for cross-account image tagging | `''` | `production-account` |

## How It Works

### 1. Environment-Specific Change Detection
The workflow automatically detects what needs to be updated based on the environment:

#### **Dev Environment**
- **Build Changes**: Any file changes except K8s manifests, GitHub workflows, and CloudFormation templates
- **K8s Changes**: Changes to files in `.k8s/` or `k8s/` directories
- **Infrastructure Changes**: Changes to the CloudFormation template file

#### **Beta/Prod Environments**
- **Build Changes**: Always skipped (no building in these environments)
- **K8s Changes**: Changes to files in `.k8s/` or `k8s/` directories
- **Infrastructure Changes**: Always skipped (no infrastructure changes in these environments)

### 2. Conditional Execution
Jobs only run when relevant changes are detected:
- Container build only runs in dev environment if application code changes
- Image tagging only runs in beta/prod environments
- CloudFormation deployment only runs in dev environment if infrastructure template changes
- Flux sync only runs if deployments were successful

### 3. Image Tagging Strategy
- **Dev**: Images tagged as `dev-{run_number}` (e.g., `dev-123`)
- **Beta**: Images tagged as `beta-{run_number}` (e.g., `beta-456`)
- **Prod**: Images tagged as `prod-{run_number}` (e.g., `prod-789`)

### 4. Cross-Account Support
For beta/prod environments, you can specify:
- `dst-registry`: Different ECR registry for the tagged image
- `dst-aws-account`: Different AWS account for deployment
- This enables promoting images across different AWS accounts

### 5. Resource Tagging
CloudFormation resources are automatically tagged with:
- `cdci: github` - Identifies deployment method
- `service: {service-name}` - Identifies the service
- `environment: {environment}` - Identifies the environment

## Prerequisites

### Repository Structure
Your repository should have:
```
├── .k8s/
│   ├── overlays/
│   │   ├── dev/
│   │   │   └── kustomization.yaml    # Contains newTag field for dev
│   │   ├── beta/
│   │   │   └── kustomization.yaml    # Contains newTag field for beta
│   │   └── prod/
│   │       └── kustomization.yaml    # Contains newTag field for prod
├── aws-resources.yaml                 # CloudFormation template (dev only)
└── Dockerfile                         # For container builds (dev only)
```

### AWS Setup
- ECR registry must exist and be accessible
- For cross-account deployments, appropriate IAM roles must be configured
- CloudFormation stack must exist in the target environment

### Composite Action
The workflow uses a composite action for image tagging:
- Location: `.github/actions/tag-docker-image`
- Handles cross-account image promotion
- Supports multi-platform image tagging
- Manages AWS role assumption and ECR login

