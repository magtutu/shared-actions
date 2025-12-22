# GitHub Environments Configuration Guide

This guide explains how to configure environments in the shared-actions repository to centrally control access and security for all consuming repositories.

## Centralized Security Model

In this setup:
- ✅ **All environments are in shared-actions repo**
- ✅ **All secrets are in shared-actions repo**
- ✅ **All protection rules are in shared-actions repo**
- ✅ **Consumer repos have NO environments or secrets**
- ✅ **One place to manage everything**

## How It Works

When a consumer repo calls a reusable workflow:

1. Consumer repo calls: `uses: YOUR_USERNAME/shared-actions/.github/workflows/deploy.yml@main`
2. The workflow runs in the **shared-actions** repo context
3. Secrets come from **shared-actions** repo environments
4. Protection rules from **shared-actions** repo are enforced
5. Approvals are requested from reviewers configured in **shared-actions**

## Setting Up Environments (shared-actions repo only)

### In the shared-actions repository:

1. Go to **Settings** → **Environments**
2. Create two environments: `staging` and `production`

#### Staging Environment:

**Secrets to add**:
- `DEPLOY_TOKEN`: Your staging deployment token
- `API_KEY`: Your staging API key (optional)

**Protection rules**:
- Deployment branches: All branches (or specific branches if you want)
- Required reviewers: None (for fast iteration)
- Wait timer: None

#### Production Environment:

**Secrets to add**:
- `DEPLOY_TOKEN`: Your production deployment token  
- `API_KEY`: Your production API key (optional)

**Protection rules**:
- ✓ **Required reviewers**: Add yourself or team members (1-6 reviewers)
- ✓ **Deployment branches**: Selected branches → `main` only
- ✓ **Wait timer**: 5-10 minutes (optional, gives you time to cancel)

## Access Control: Who Can Deploy?

This is the key question with centralized environments. You have several options:

### Option 1: Trust All Consumer Repos (Default)
- Any repo that can call your workflow can trigger deployments
- Protection rules still apply (approvals, branch restrictions)
- Good for: Small teams, trusted repos only

### Option 2: Restrict Which Repos Can Call Workflows
In shared-actions repo:
1. Go to **Settings** → **Actions** → **General**
2. Scroll to **"Access"** section
3. Choose:
   - **Accessible from repositories in the 'YOUR_ORG' organization**: Any org repo can use it
   - **Accessible from repositories owned by the user 'YOUR_USERNAME'**: Only your repos
   - **Not accessible**: Only shared-actions repo can use it (for testing)

### Option 3: Use Required Reviewers as Gatekeepers
- Set required reviewers on production environment
- Even if a consumer repo triggers deployment, it pauses for approval
- Reviewers can see which repo triggered it and approve/reject
- Good for: Controlled production access

### Option 4: Branch Protection + Environment Protection
- Require deployments to come from specific branches
- In production environment: Set "Deployment branches" to `main` only
- Consumer repos can only deploy if their workflow runs from main branch
- Good for: Ensuring only merged code deploys

## Security Implications

### Advantages of Centralized Model:
- ✅ One place to manage all secrets
- ✅ One place to update deployment credentials
- ✅ Consistent protection rules across all consumers
- ✅ Easier to audit (all deployments in one repo)
- ✅ Consumer repos don't need any setup

### Considerations:
- ⚠️ All consumer repos share the same secrets
- ⚠️ Any consumer repo can trigger deployments (unless restricted)
- ⚠️ Need to trust all repos that can call your workflows
- ⚠️ Approvers need to verify which repo is deploying

## Best Practices

### 1. Restrict Workflow Access
Don't leave your workflows accessible to all repos. Use the Access settings to limit which repos can call them.

### 2. Use Required Reviewers for Production
Always require approval for production deployments. The reviewer should verify:
- Which consumer repo is deploying
- What version is being deployed
- Whether the deployment is expected

### 3. Monitor Deployment History
Check the Actions tab regularly to see:
- Which repos are using your workflows
- What's being deployed where
- Who approved what

### 4. Use Descriptive App Names
In consumer workflows, pass meaningful app names:
```yaml
with:
  app-name: team-frontend  # Not just "my-app"
```

This helps reviewers know what's being deployed.

### 5. Consider Per-Consumer Environments
If you need isolation between consumers, create environments like:
- `staging-team-a`
- `staging-team-b`
- `production-team-a`
- `production-team-b`

Each with their own secrets and protection rules.

### 6. Use Deployment Logs
The reusable workflow logs which app and version is being deployed. Review these logs to track what's happening.

## Setup Steps

### In shared-actions repo:

1. **Create environments**: Settings → Environments
   - Create `staging` environment
   - Create `production` environment

2. **Add secrets to each environment**:
   - Go to each environment
   - Add `DEPLOY_TOKEN` secret
   - Add `API_KEY` secret (if needed)

3. **Configure production protection**:
   - Add required reviewers
   - Set deployment branches to `main` only
   - Optional: Add wait timer

4. **Restrict workflow access**: Settings → Actions → General → Access
   - Choose appropriate access level
   - Save changes

### In actions-consumer repo:

1. **Update workflow file**: Replace `YOUR_USERNAME` with your GitHub username
2. **That's it!** No environments or secrets needed

## Testing Your Setup

### Test 1: Staging Deployment
```bash
cd actions-consumer
git add .
git commit -m "Test staging deployment"
git push origin main
```
- Should trigger automatically
- Should deploy to staging without approval
- Check Actions tab in both repos

### Test 2: Production Deployment
1. Go to actions-consumer repo → Actions tab
2. Select "Deploy Application" workflow
3. Click "Run workflow"
4. Choose "production" environment
5. Should pause and request approval
6. Approve in shared-actions repo (Settings → Environments → production → View deployment)
7. Should complete deployment

### Test 3: Verify Secrets
Add a temporary step to the deploy workflow:
```yaml
- name: Test secret access
  run: |
    if [ -n "${{ secrets.DEPLOY_TOKEN }}" ]; then
      echo "✓ Secret accessible"
    else
      echo "✗ Secret not accessible"
      exit 1
    fi
```

## Troubleshooting

**Issue**: "Environment not found"
- **Solution**: Create the environment in shared-actions repo (not consumer repo)

**Issue**: "Secret not found"  
- **Solution**: Add secrets to the environment in shared-actions repo

**Issue**: "Deployment not pausing for approval"
- **Solution**: Check protection rules in shared-actions repo's environment

**Issue**: "Consumer repo can't call workflow"
- **Solution**: Check Access settings in shared-actions repo

**Issue**: "Approval request not showing"
- **Solution**: Go to shared-actions repo → Environments → Click on environment → View pending deployments

## Comparison: Centralized vs Distributed

| Aspect | Centralized (This Setup) | Distributed (Previous Setup) |
|--------|-------------------------|------------------------------|
| Secrets location | shared-actions only | Each consumer repo |
| Environments | shared-actions only | Each consumer repo |
| Protection rules | shared-actions only | Each consumer repo |
| Setup complexity | Simple for consumers | Setup needed per consumer |
| Secret isolation | Shared across consumers | Isolated per consumer |
| Access control | Workflow access + reviewers | Each repo controls own |
| Best for | Trusted repos, central control | Multiple teams, isolation |

## Next Steps

1. Create the environments in shared-actions repo
2. Add the required secrets to each environment
3. Configure protection rules for production
4. Restrict workflow access appropriately
5. Test a staging deployment
6. Test a production deployment with approval
7. Review the deployment history and logs
