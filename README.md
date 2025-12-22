# Shared Actions

This repository contains reusable GitHub Actions workflows that can be called from other repositories.

## Public API

### app-merge.yml - Deployment Pipeline on Merge

**This is the only workflow you should call from consumer repositories.**

Orchestrates the complete deployment pipeline: staging → approval → production.

**Usage**:
```yaml
jobs:
  deploy:
    uses: YOUR_USERNAME/shared-actions/.github/workflows/app-merge.yml@main
    with:
      app-name: your-repo-name
```

**Inputs**:
- `app-name` (required): Name of your application/repository

**What it does**:
1. Deploys to staging environment
2. Waits for approval (if configured)
3. Deploys to production environment

**Requirements**:
- Your repository name should match the `app-name` input
- IAM role must exist: `deployment-role-{app-name}`
- ECS service must exist: `{app-name}-service`

## Internal Workflows

The following workflows are used internally by `app-merge.yml` and should not be called directly:

- `app-deploy.yml` - Handles deployment to a specific environment

While this is technically callable, it is not part of the public API and may change without notice.

## Environment Configuration

See [ENVIRONMENTS.md](./ENVIRONMENTS.md) for detailed information about:
- Setting up GitHub environments
- Configuring protection rules
- Managing secrets securely
- Access control patterns
- Best practices

## AWS OIDC Configuration

See [AWS_OIDC_SETUP.md](./AWS_OIDC_SETUP.md) for:
- Setting up AWS OIDC identity provider
- Creating IAM roles with proper trust policies
- Configuring dynamic role selection
- Security best practices

## Security Model

- **All secrets are stored in the shared-actions repository**
- **All environment protection rules are in the shared-actions repository**
- This repo only contains the workflow logic
- Consumer repos just call the workflows - no setup needed
- Access control is managed via workflow access settings in this repo

## Repository Access

To control which repositories can use these workflows:
1. Go to Settings → Actions → General
2. Under "Access", configure which repos can call these workflows
3. Options: Organization-wide, specific repos, or private only
