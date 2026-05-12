# CI/CD Setup for Grafana Quickwit Image

This repository includes a GitHub Actions workflow that automatically builds and pushes a Grafana Docker image with the patched Quickwit datasource plugin to ECR.

## Overview

The workflow uses the **`gha-runner-ecr-publish`** self-hosted runner which already has AWS credentials configured. **No additional secrets are required.**

## Workflow Triggers

The workflow runs and **publishes to ECR** on:
- **Push** to `disable-field-caps-all-fields` or `main` branches
- **PR merge** to these branches  
- **Manual** workflow dispatch with `force_publish` option

The workflow **builds but does not publish** on:
- Pull request events (for testing)
- Other branch pushes

## Image Details

### Target ECR Repository
- **Repository**: `grafana-quickwit`
- **Region**: `us-east-1`
- **Registry**: `337909757619.dkr.ecr.us-east-1.amazonaws.com`

### Image Tags

Images use **immutable tags only** (no `latest` tag for production safety):

**For Git Tags** (e.g., `v0.6.0-patched-1`):
```
12.4.0-quickwit-0.6.0-patched-1
```

**For Untagged Commits**:
```
12.4.0-quickwit-0.6.0-patched-a1b2c3d
```

Where:
- `12.4.0` = Grafana version
- `0.6.0-patched` = Quickwit plugin version (patched)
- `a1b2c3d` = Short git SHA (7 chars)

### Image Contents
- **Base**: Grafana 12.4.0
- **Plugin**: Quickwit datasource v0.6.0 (patched to disable field_caps)
- **Platform**: linux/amd64

## What the Workflow Does

1. **Build Plugin**
   - Installs Node.js and Go dependencies
   - Builds frontend (TypeScript → JavaScript)
   - Builds backend (Go binaries for Linux)
   - Removes signature files (since plugin is patched)
   - Packages as ZIP

2. **Build Docker Image**
   - Creates Dockerfile dynamically
   - Copies patched plugin into Grafana base image
   - Configures unsigned plugin loading
   - Adds metadata labels

3. **Publish to ECR** (conditional)
   - Authenticates to ECR using runner's AWS credentials
   - Tags image with git hash and `latest`
   - Pushes both tags to ECR
   - Generates build summary

## Running the Workflow

### Automatic (Recommended)
Just push commits to `disable-field-caps-all-fields` branch:
```bash
git push origin disable-field-caps-all-fields
```

The workflow will automatically build and push to ECR.

### Manual Trigger
1. Go to the **Actions** tab in GitHub
2. Select **Build and Push Grafana with Quickwit Plugin**
3. Click **Run workflow**
4. Select branch: `disable-field-caps-all-fields`
5. Check **force_publish** if you want to publish to ECR
6. Click **Run workflow**

## Verifying the Build

After the workflow completes:

1. **Check GitHub Actions**: The workflow summary will show the published image tags
2. **Check ECR**: 
   ```bash
   aws ecr describe-images \
     --repository-name grafana-quickwit \
     --region us-east-1 \
     --query 'sort_by(imageDetails,& imagePushedAt)[-5:]' \
     --output table
   ```

## Using the Image

### Deployment Strategy

1. **Find the latest tag** from the workflow output or ECR
2. **Deploy to preprod** for testing
3. **Promote to prod** after validation

```yaml
# Preprod - test new builds
image: 337909757619.dkr.ecr.us-east-1.amazonaws.com/grafana-quickwit:12.4.0-quickwit-0.6.0-patched-a1b2c3d

# Prod - promote after preprod validation
image: 337909757619.dkr.ecr.us-east-1.amazonaws.com/grafana-quickwit:12.4.0-quickwit-0.6.0-patched-a1b2c3d
```

### Creating Release Tags

To create a versioned release:

```bash
# Create and push a version tag
git tag -a v0.6.0-patched-1 -m "Release v0.6.0-patched-1"
git push origin v0.6.0-patched-1

# This will create image tag: 12.4.0-quickwit-0.6.0-patched-1
```

## Troubleshooting

### Build Fails on Plugin Build
- Check Node.js and Go versions in the workflow match requirements
- Review build logs for npm or go errors

### Docker Build Fails
- Verify the Grafana base image version exists
- Check that plugin ZIP was created successfully

### ECR Push Fails
- Verify the `gha-runner-ecr-publish` runner has ECR write permissions
- Check that the ECR repository `grafana-quickwit` exists
- Verify AWS credentials on the runner are valid

### Workflow Doesn't Trigger
- Ensure you're pushing to the correct branch
- Check workflow file syntax in `.github/workflows/build-and-push.yml`
- Verify GitHub Actions are enabled for the repository

## Comparing with Backstage Setup

This workflow follows the same pattern as `Iterable/backstage`:
- Uses `gha-runner-ecr-publish` runner
- Authenticates with `aws ecr get-login-password`
- Conditionally publishes based on event type
- Generates summary with published tags

No IAM roles or GitHub secrets are required because the self-hosted runner already has the necessary AWS permissions.
