# 🎭 Staging Environment Workflows

This document explains how to set up **automatic staging environments** that deploy preview instances for every Pull Request.

## 📋 Overview

The staging system creates temporary Cloud Run services for each PR branch:

- **🚀 Deploy**: `staging-reusable.yaml` creates a new service when PR is opened
- **🧹 Cleanup**: `cleanup-staging.yaml` deletes the service when PR is closed/merged

## 🏗️ Service Naming Convention

Staging services follow this pattern:

```
{base-service-name}-staging-{clean-branch-name}
```

Examples:

- Branch `feature/user-auth` → Service `krv-web-staging-feature-user-auth`
- Branch `fix/bug-123` → Service `krv-web-staging-fix-bug-123`
- Branch `main` → Service `krv-web-staging-main`

## 🔧 Setup Instructions

### 1️⃣ Create Staging Workflow

In your repository, create `.github/workflows/staging.yaml`:

```yaml
name: 🎭 Deploy Staging Environment

on:
  pull_request:
    types: [opened, synchronize, reopened]
    branches: [main]

jobs:
  deploy-staging:
    uses: Krv-Analytics/.github-private/.github/workflows/staging-reusable.yaml@main
    with:
      base_service_name: krv-web # Your base service name
      region: us-central1 # GCP region
    secrets:
      GCP_PROJECT_ID: ${{ secrets.GCP_PROJECT_ID }}
      PROJECT_NUMBER: ${{ secrets.PROJECT_NUMBER }}
      ARTIFACT_REGISTRY_REPO: ${{ secrets.ARTIFACT_REGISTRY_REPO }}
```

### 2️⃣ Create Cleanup Workflow

Create `.github/workflows/cleanup-staging.yaml`:

```yaml
name: 🧹 Cleanup Staging Environment

on:
  pull_request:
    types: [closed]
    branches: [main]

jobs:
  cleanup-staging:
    uses: Krv-Analytics/.github-private/.github/workflows/cleanup-staging.yaml@main
    with:
      base_service_name: krv-web # Must match staging deploy
      region: us-central1 # Must match staging deploy
    secrets:
      GCP_PROJECT_ID: ${{ secrets.GCP_PROJECT_ID }}
      PROJECT_NUMBER: ${{ secrets.PROJECT_NUMBER }}
```

## 🎯 How It Works

### **When PR is Opened/Updated:**

1. **🏷️ Name Generation**: Creates unique service name from branch
2. **🚀 Build & Deploy**: Builds container and deploys to Cloud Run
3. **⚙️ Configuration**: Sets staging-specific settings (lower resources, staging env vars)
4. **💬 PR Comment**: Posts the preview URL as a comment
5. **🏷️ Labels**: Tags service with branch, repo, and PR metadata

### **When PR is Closed/Merged:**

1. **🔍 Service Check**: Verifies staging service exists
2. **🗑️ Service Deletion**: Removes the Cloud Run service
3. **🧼 Image Cleanup**: Deletes staging container images
4. **💬 Confirmation**: Comments on PR about successful cleanup

## ⚙️ Staging Service Configuration

Staging environments are configured with:

- **CPU**: 1 vCPU (lower than production)
- **Memory**: 512Mi (resource-efficient)
- **Instances**: 0-2 (scales to zero when not used)
- **Environment**: `NODE_ENV=staging`, `STAGING=true`
- **Labels**: `environment=staging`, `branch=branch-name`, `pr=123`

## 🎨 Customization Options

### Custom Build Files

Use local Dockerfile or cloudbuild.yaml:

```yaml
deploy-staging:
  uses: Krv-Analytics/.github-private/.github/workflows/staging-reusable.yaml@main
  with:
    base_service_name: my-app
    region: us-central1
    dockerfile_path: ./staging/Dockerfile # Custom Dockerfile
    cloudbuild_path: ./staging/cloudbuild.yaml # Custom build config
  secrets:
    # ... secrets
```

### Custom Template Branch

Use different template branch:

```yaml
deploy-staging:
  uses: Krv-Analytics/.github-private/.github/workflows/staging-reusable.yaml@main
  with:
    base_service_name: my-app
    region: us-central1
    template_branch: dev # Use templates from 'dev' branch
  secrets:
    # ... secrets
```

### Different Triggers

Deploy staging for different events:

```yaml
# Only deploy staging for specific labels
on:
  pull_request:
    types: [opened, synchronize, labeled]
    branches: [main]

jobs:
  deploy-staging:
    if: contains(github.event.pull_request.labels.*.name, 'deploy-staging')
    uses: Krv-Analytics/.github-private/.github/workflows/staging-reusable.yaml@main
    # ... rest of config
```

## 🔍 Monitoring & Debugging

### View Staging Services

List all staging environments:

```bash
gcloud run services list \
  --region=us-central1 \
  --filter="metadata.labels.environment=staging" \
  --format="table(metadata.name,status.url,metadata.labels.branch,metadata.labels.pr)"
```

### Check Service Logs

```bash
gcloud logging read "resource.type=cloud_run_revision AND resource.labels.service_name=krv-web-staging-feature-xyz" \
  --limit=50 \
  --format="table(timestamp,textPayload)"
```

### Manual Cleanup

If automatic cleanup fails, manually delete:

```bash
# Delete staging service
gcloud run services delete krv-web-staging-feature-xyz --region=us-central1

# Delete staging images
gcloud artifacts docker images list \
  us-central1-docker.pkg.dev/PROJECT/REPO/krv-web \
  --filter="tags:staging-*" \
  --format="value(IMAGE)" | xargs gcloud artifacts docker images delete
```

## 🚨 Troubleshooting

| Issue                 | Cause                                        | Solution                                             |
| --------------------- | -------------------------------------------- | ---------------------------------------------------- |
| Service name too long | Branch name creates >63 char service name    | Workflow auto-truncates and adds hash for uniqueness |
| Cleanup doesn't work  | Service name mismatch between deploy/cleanup | Ensure `base_service_name` matches exactly           |
| No PR comment         | Missing `pull-requests: write` permission    | Add permission to workflow job                       |
| Build fails           | Missing templates or custom files            | Check `dockerfile_path`/`cloudbuild_path` exist      |
| Service doesn't start | Resource limits too low                      | Increase CPU/memory in staging workflow              |

## 📊 Example Repository Structure

```
my-app/
├── .github/
│   └── workflows/
│       ├── deploy.yaml           # Production deployment
│       ├── staging.yaml          # PR staging deployment
│       └── cleanup-staging.yaml  # PR cleanup
├── staging/                      # Optional: staging-specific configs
│   ├── Dockerfile
│   └── cloudbuild.yaml
├── src/
└── package.json
```

## 🎉 Benefits

- **🔍 Easy Testing**: Every PR gets its own URL for testing
- **🔄 Automatic Lifecycle**: Created and destroyed automatically
- **💰 Cost Efficient**: Scales to zero when not used
- **👥 Team Visibility**: PR comments keep everyone informed
- **🏷️ Well Organized**: Clear labeling and naming conventions
- **🧹 No Waste**: Automatic cleanup prevents orphaned resources

## 🚀 Example Workflow

1. Developer creates PR for `feature/new-login`
2. Staging workflow triggers automatically
3. Service `krv-web-staging-feature-new-login` is created
4. Bot comments: "🎭 Staging Environment Ready! URL: https://krv-web-staging-feature-new-login-abc123.run.app"
5. Team tests the feature
6. PR is merged
7. Cleanup workflow deletes the staging service
8. Bot comments: "🧹 Staging Environment Cleaned Up"

This gives you a complete **preview environment system** with zero manual intervention! 🎉
