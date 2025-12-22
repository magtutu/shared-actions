# AWS OIDC Configuration for Reusable Workflows

This guide explains how to configure AWS IAM with GitHub OIDC to ensure that consumer repos can only deploy to their own resources.

## The Challenge

When `actions-consumer` calls a reusable workflow in `shared-actions`:
- The workflow runs in the **shared-actions** repository context
- But GitHub includes the **calling repository** in the OIDC token claims
- We need to use these claims to restrict what each consumer can deploy

## OIDC Token Claims

When a reusable workflow is called, the OIDC token includes:

```json
{
  "repository": "YOUR_USERNAME/shared-actions",
  "repository_owner": "YOUR_USERNAME",
  "job_workflow_ref": "YOUR_USERNAME/shared-actions/.github/workflows/deploy.yml@refs/heads/main",
  "workflow_ref": "YOUR_USERNAME/actions-consumer/.github/workflows/deploy-app.yml@refs/heads/main",
  "repository_id": "123456789",
  "repository_owner_id": "987654321"
}
```

Key claims:
- `repository`: The repo where the workflow is defined (shared-actions)
- `workflow_ref`: The repo that called the workflow (actions-consumer) ⭐ **This is what we need!**

## AWS IAM Setup

### Step 1: Create OIDC Identity Provider

In AWS IAM Console:

1. Go to **IAM** → **Identity providers** → **Add provider**
2. Provider type: **OpenID Connect**
3. Provider URL: `https://token.actions.githubusercontent.com`
4. Audience: `sts.amazonaws.com`
5. Click **Add provider**

Or via AWS CLI:
```bash
aws iam create-open-id-connect-provider \
  --url https://token.actions.githubusercontent.com \
  --client-id-list sts.amazonaws.com \
  --thumbprint-list 6938fd4d98bab03faadb97b34396831e3780aea1
```

## Dynamic Role Selection with Branch Restrictions

### Step 2: Create IAM Roles with Dynamic Naming Convention

For each consumer repository, create a role following the pattern: `deployment-role-{repo-name}`

#### Example: deployment-role-actions-consumer

**Trust Policy** (`deployment-role-actions-consumer-trust-policy.json`):
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::YOUR_ACCOUNT_ID:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
          "token.actions.githubusercontent.com:sub": "repo:YOUR_USERNAME/actions-consumer:ref:refs/heads/main"
        }
      }
    }
  ]
}
```

**Key Security Features**:
- ✅ Only the `actions-consumer` repository can assume this role
- ✅ Only deployments from the `main` branch are allowed
- ✅ Feature branches cannot deploy (will fail at AWS role assumption)
- ✅ The shared-actions workflow dynamically constructs the role name from the calling repo

**Permissions Policy** (`deployment-role-actions-consumer-permissions.json`):
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowECRAccess",
      "Effect": "Allow",
      "Action": [
        "ecr:GetAuthorizationToken",
        "ecr:BatchCheckLayerAvailability",
        "ecr:GetDownloadUrlForLayer",
        "ecr:BatchGetImage"
      ],
      "Resource": "*"
    },
    {
      "Sid": "AllowECRPush",
      "Effect": "Allow",
      "Action": [
        "ecr:PutImage",
        "ecr:InitiateLayerUpload",
        "ecr:UploadLayerPart",
        "ecr:CompleteLayerUpload"
      ],
      "Resource": "arn:aws:ecr:us-east-1:YOUR_ACCOUNT_ID:repository/actions-consumer"
    },
    {
      "Sid": "AllowECSDeployToSpecificService",
      "Effect": "Allow",
      "Action": [
        "ecs:UpdateService",
        "ecs:DescribeServices",
        "ecs:DescribeTaskDefinition",
        "ecs:RegisterTaskDefinition"
      ],
      "Resource": [
        "arn:aws:ecs:us-east-1:YOUR_ACCOUNT_ID:service/*/actions-consumer-service",
        "arn:aws:ecs:us-east-1:YOUR_ACCOUNT_ID:task-definition/actions-consumer:*"
      ]
    },
    {
      "Sid": "AllowPassRole",
      "Effect": "Allow",
      "Action": "iam:PassRole",
      "Resource": "arn:aws:iam::YOUR_ACCOUNT_ID:role/ecsTaskExecutionRole",
      "Condition": {
        "StringEquals": {
          "iam:PassedToService": "ecs-tasks.amazonaws.com"
        }
      }
    }
  ]
}
```

Create the role:
```bash
# Create the role with trust policy
aws iam create-role \
  --role-name deployment-role-actions-consumer \
  --assume-role-policy-document file://deployment-role-actions-consumer-trust-policy.json \
  --description "Deployment role for actions-consumer repo (main branch only)"

# Attach the permissions policy
aws iam put-role-policy \
  --role-name deployment-role-actions-consumer \
  --policy-name DeploymentPermissions \
  --policy-document file://deployment-role-actions-consumer-permissions.json
```

### How the Dynamic Role Selection Works

1. **Consumer repo calls the workflow**:
   ```yaml
   uses: YOUR_USERNAME/shared-actions/.github/workflows/deploy.yml@main
   with:
     app-name: actions-consumer
   ```

2. **Shared-actions workflow extracts the repo name**:
   - Uses `inputs.app-name` (which should match the repo name)
   - Constructs role name: `deployment-role-actions-consumer`

3. **Workflow assumes the role**:
   ```yaml
   role-to-assume: arn:aws:iam::${{ secrets.AWS_ACCOUNT_ID }}:role/deployment-role-${{ inputs.app-name }}
   ```

4. **AWS validates**:
   - Is the OIDC token from `actions-consumer` repo? ✓
   - Is it from the `main` branch? ✓
   - Does the role name match? ✓
   - Allow assumption

5. **Deployment proceeds** with permissions scoped to `actions-consumer-service`

### Step 3: Store AWS Account Info in GitHub Secrets

In the **shared-actions** repository environments:

1. Go to **Settings** → **Environments** → **staging**
2. Add secrets:
   - `AWS_ACCOUNT_ID` = Your AWS account ID (e.g., `123456789012`)
   - `AWS_REGION` = Your AWS region (e.g., `us-east-1`)
   - `ECS_CLUSTER_NAME` = Your ECS cluster name (e.g., `my-cluster`)

3. Repeat for **production** environment

**Note**: You do NOT need to store role ARNs as secrets. The workflow dynamically constructs the role ARN from:
- `AWS_ACCOUNT_ID` secret
- `app-name` input (which becomes the role name suffix)

## Updated Workflow

The `shared-actions/.github/workflows/deploy.yml` workflow now:

1. **Extracts the calling repository name** from the `app-name` input
2. **Dynamically constructs the role name**: `deployment-role-{app-name}`
3. **Assumes the role** using AWS OIDC
4. **Deploys to ECS** with the assumed role's permissions

Key workflow steps:

```yaml
- name: Extract calling repository name
  id: repo-info
  run: |
    CALLING_REPO="${{ inputs.app-name }}"
    echo "calling-repo=$CALLING_REPO" >> $GITHUB_OUTPUT
    echo "role-name=deployment-role-$CALLING_REPO" >> $GITHUB_OUTPUT

- name: Configure AWS credentials
  uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: arn:aws:iam::${{ secrets.AWS_ACCOUNT_ID }}:role/${{ steps.repo-info.outputs.role-name }}
    aws-region: ${{ secrets.AWS_REGION }}
    role-session-name: github-actions-${{ inputs.app-name }}-${{ inputs.environment }}
```

The workflow is already updated in your repository!

## Multiple Consumer Repos

For each consumer repo, create a role following the naming pattern:

### Example: deployment-role-other-app

**Trust Policy**:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::YOUR_ACCOUNT_ID:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
          "token.actions.githubusercontent.com:sub": "repo:YOUR_USERNAME/other-app:ref:refs/heads/main"
        }
      }
    }
  ]
}
```

**Permissions Policy**: Same structure, but resources limited to:
- ECR: `other-app` repository
- ECS: `other-app-service` service
- Task definitions: `other-app` family

The workflow automatically uses the correct role based on the `app-name` input.

## Naming Convention Requirements

For this dynamic approach to work, you must follow these naming conventions:

### Repository Names
- Consumer repo: `actions-consumer`
- Another consumer: `other-app`

### IAM Role Names
- For actions-consumer: `deployment-role-actions-consumer`
- For other-app: `deployment-role-other-app`

### ECS Service Names
- For actions-consumer: `actions-consumer-service`
- For other-app: `other-app-service`

### ECR Repository Names
- For actions-consumer: `actions-consumer`
- For other-app: `other-app`

### Workflow Input
The `app-name` input should match the repository name:
```yaml
with:
  app-name: actions-consumer  # Must match repo name
```

This ensures:
- Role name is correctly constructed: `deployment-role-actions-consumer`
- Service name is correctly constructed: `actions-consumer-service`
- ECR repository matches: `actions-consumer`
- Trust policy validates the correct repo

## Security Benefits

This setup ensures:

✅ **actions-consumer** can only assume `deployment-role-actions-consumer`
✅ `deployment-role-actions-consumer` can only deploy to `actions-consumer-service`
✅ Only deployments from the **main branch** can assume the role
✅ Feature branches cannot deploy (AWS denies role assumption)
✅ Even if actions-consumer tries to deploy to another service, AWS denies it
✅ Even if another repo tries to use actions-consumer's role, GitHub OIDC denies it
✅ No hardcoded role ARNs - everything is derived from the repo name
✅ All deployments are audited in CloudTrail with the calling repo information

## Branch Protection Strategy

The trust policy enforces `ref:refs/heads/main`, which means:

- ✅ Deployments from `main` branch: **Allowed**
- ❌ Deployments from `feature/new-feature` branch: **Denied by AWS**
- ❌ Deployments from `develop` branch: **Denied by AWS**
- ❌ Deployments from pull requests: **Denied by AWS**

This provides an additional security layer beyond GitHub environment protection rules.

## Testing

1. **Test successful deployment**:
   - Trigger deployment from actions-consumer
   - Should successfully deploy to actions-consumer-service

2. **Test unauthorized service** (should fail):
   - Modify workflow to try deploying to `other-service`
   - AWS should deny with "Access Denied"

3. **Test unauthorized repo** (should fail):
   - Try calling the workflow from a different repo
   - GitHub OIDC should deny role assumption

## Troubleshooting

**Error**: "Not authorized to perform sts:AssumeRoleWithWebIdentity"
- Check the trust policy `sub` claim matches your repo name
- Verify the OIDC provider is configured correctly
- Check that `permissions: id-token: write` is set in the workflow

**Error**: "Access Denied" when updating ECS service
- Check the permissions policy allows the specific service ARN
- Verify the service name matches the pattern in the policy
- Check CloudTrail logs for the exact denied action

**Error**: "Role ARN not found"
- Verify the secret is added to the correct environment in shared-actions
- Check the environment name matches what the workflow is using

## References

- [GitHub OIDC with AWS](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services)
- [Reusable Workflows and OIDC](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/about-security-hardening-with-openid-connect#updating-your-actions-for-oidc)
- [AWS IAM OIDC Identity Providers](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_providers_create_oidc.html)
