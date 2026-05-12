# CI/CD Setup for Grafana Quickwit Image

This repository includes a GitHub Actions workflow that automatically builds and pushes a Grafana Docker image with the patched Quickwit datasource plugin to ECR.

## Required GitHub Secrets

The workflow requires the following secret to be configured in the repository:

### `AWS_ROLE_ARN`
AWS IAM Role ARN with permissions to push to ECR.

**Example format**: `arn:aws:iam::337909757619:role/github-actions-ecr-push`

**Required Permissions**:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ecr:GetAuthorizationToken",
        "ecr:BatchCheckLayerAvailability",
        "ecr:GetDownloadUrlForLayer",
        "ecr:BatchGetImage",
        "ecr:PutImage",
        "ecr:InitiateLayerUpload",
        "ecr:UploadLayerPart",
        "ecr:CompleteLayerUpload"
      ],
      "Resource": [
        "arn:aws:ecr:us-east-1:337909757619:repository/grafana-quickwit"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "ecr:GetAuthorizationToken"
      ],
      "Resource": "*"
    }
  ]
}
```

## Setting up the Secret

1. Go to the repository on GitHub: https://github.com/Iterable/quickwit-datasource
2. Navigate to **Settings** → **Secrets and variables** → **Actions**
3. Click **New repository secret**
4. Name: `AWS_ROLE_ARN`
5. Value: The ARN of your IAM role (e.g., `arn:aws:iam::337909757619:role/github-actions-ecr-push`)
6. Click **Add secret**

## IAM Role Trust Policy

The IAM role must trust GitHub Actions from the Iterable organization:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::337909757619:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:Iterable/quickwit-datasource:*"
        }
      }
    }
  ]
}
```

## Workflow Triggers

The workflow runs on:
- **Push** to `disable-field-caps-all-fields` branch
- **Push** to `main` branch
- **Tags** matching `v*` pattern
- **Manual** trigger via workflow_dispatch

## Image Tags

Images are tagged as:
- `<grafana-version>-quickwit-<plugin-version>-<short-sha>` for branch builds
- `<grafana-version>-quickwit-<version>` for tag builds
- `latest` for main branch or tag builds

**Example**: `12.4.0-quickwit-0.6.0-patched-a1b2c3d`

## Target ECR Repository

- **Repository**: `grafana-quickwit`
- **Region**: `us-east-1`
- **Registry**: `337909757619.dkr.ecr.us-east-1.amazonaws.com`

## Verifying the Workflow

After setting up the secret, the workflow will run automatically on the next push. You can also trigger it manually:

1. Go to **Actions** tab
2. Select **Build and Push Grafana with Quickwit Plugin**
3. Click **Run workflow**
4. Select the branch and click **Run workflow**

## Troubleshooting

**Error: Unable to locate credentials**
- Verify the `AWS_ROLE_ARN` secret is set correctly
- Check that the IAM role exists and the ARN is correct

**Error: AccessDenied**
- Verify the IAM role has the correct permissions policy
- Verify the IAM role's trust policy allows GitHub Actions from this repository

**Error: Repository does not exist**
- Verify the ECR repository `grafana-quickwit` exists in `us-east-1`
- Check the repository name in the workflow matches exactly
