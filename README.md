# GitHub Actions Deploy to Kubernetes

A reusable GitHub Actions workflow for deploying services to Kubernetes clusters with support for AWS ECR, CloudFormation, and Flux GitOps.

## What it does

This workflow automates the deployment process for Kubernetes applications by:

- **Tagging existing Docker images** in AWS ECR with environment-specific tags
- **Deploying CloudFormation infrastructure** (only in dev environment)
- **Updating Kubernetes manifests** with new image tags
- **Synchronizing with Flux GitOps** by updating the target branch
- **Supporting cross-account deployments** for beta/prod environments

The workflow intelligently detects changes to determine what needs to be deployed:
- K8s manifest changes trigger Flux branch updates
- CloudFormation template changes trigger infrastructure deployments
- Beta/prod environments tag existing dev images with new environment tags

## Required Inputs

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `environment` | The environment to deploy to (dev, beta, prod) | ✅ | `dev` |
| `service-name` | The name of the service to deploy | ✅ | - |
| `registry` | The ECR registry URL to push images to | ✅ | - |
| `path-to-k8s-image-tag` | Path to the kustomization.yaml file containing the image tag | ✅ | - |

## Optional Inputs

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `cf-template` | The CloudFormation template file path | ❌ | `aws-resources.yaml` |
| `branch-override` | The branch that will be updated | ❌ | `main` |
| `eks-cluster-stack-name` | The name of the EKS cluster stack for CloudFormation | ❌ | `bisnow-non-prod-eks` |
| `aws-account` | AWS account identifier | ❌ | `bisnow` |
| `flux-target-branch` | The branch that flux watches | ❌ | `flux-main` |
| `source-image-tag` | Source image tag to use for beta/prod environments | ❌ | `''` |
| `dst-registry` | Destination registry for cross-account image tagging | ❌ | `''` |
| `dst-aws-account` | Destination AWS account for cross-account image tagging | ❌ | `''` |
| `disable-cloudformation` | Disable CloudFormation deployment | ❌ | `false` |

## Usage Examples

### Basic Dev Deployment

```yaml
name: Deploy changes to dev

permissions:
  id-token: write
  contents: write
  pull-requests: read

concurrency:
  group: dev-client-portal-deploy

on:
  workflow_dispatch:
  workflow_call:

jobs:
  deploy:
    uses: bisnow/github-actions-deploy-to-k8s/.github/workflows/deploy.yaml@main
    with:
      environment: dev
      service-name: client-portal
      registry: 000000000000.dkr.ecr.us-east-1.amazonaws.com/client-portal
      path-to-k8s-image-tag: .k8s/overlays/dev/kustomization.yaml
      cf-template: aws-resources.yaml
      eks-cluster-stack-name: bisnow-non-prod-eks
      branch-override: devops/k8s-deploy
```

### Production Deployment with Cross-Account Image Tagging

```yaml
name: Deploy to production

permissions:
  id-token: write
  contents: write
  pull-requests: read

concurrency:
  group: prod-client-portal-deploy

on:
  workflow_dispatch:
  workflow_call:

jobs:
  deploy:
    uses: bisnow/github-actions-deploy-to-k8s/.github/workflows/deploy.yaml@main
    with:
      environment: prod
      service-name: client-portal
      registry: 000000000000.dkr.ecr.us-east-1.amazonaws.com/client-portal
      path-to-k8s-image-tag: .k8s/overlays/prod/kustomization.yaml
      source-image-tag: dev-123
      dst-registry: 123456789012.dkr.ecr.us-east-1.amazonaws.com/client-portal
      dst-aws-account: production
      branch-override: main
```

### Deployment Without CloudFormation

```yaml
name: Deploy without infrastructure

jobs:
  deploy:
    uses: bisnow/github-actions-deploy-to-k8s/.github/workflows/deploy.yaml@main
    with:
      environment: dev
      service-name: my-service
      registry: 000000000000.dkr.ecr.us-east-1.amazonaws.com/my-service
      path-to-k8s-image-tag: .k8s/overlays/dev/kustomization.yaml
      disable-cloudformation: true
```

## Workflow Behavior

### Environment-Specific Behavior

- **Dev Environment**: Deploys infrastructure (CloudFormation) and updates K8s manifests
- **Beta/Prod Environments**: Tags existing dev images and updates K8s manifests

### Change Detection

The workflow automatically detects what needs to be deployed:

1. **K8s Changes**: Updates Flux target branch when `.k8s/` files change
2. **CloudFormation Changes**: Deploys infrastructure when template files change (dev only)
3. **Image Tagging**: For beta/prod, tags existing dev images with new environment tags

### Concurrency

Use the `concurrency` setting to prevent multiple deployments from running simultaneously for the same service and environment.

## Prerequisites

- AWS ECR repository with existing images
- Kubernetes manifests in `.k8s/` directory
- Flux GitOps operator configured
- Proper AWS IAM permissions for ECR and CloudFormation
- GitHub Actions permissions for the required scopes

