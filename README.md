# Electric Book Deploy Workflow Plugin

A deploy workflow plugin used by Electric Book projects and media repos with automated deploys to an Electric Book Server instance and S3 buckets.

## Creating the OIDC provider on AWS S3 for GitHub Action connections

### 1. Create the OIDC Identity Provider

1. In the AWS console for relevant account, Go to IAM > Identity providers > Create provider
2. Select OpenID Connect for type
3. Enter https://token.actions.githubusercontent.com for Provider URL
4. Enter sts.amazonaws.com for Audience
5. Click Add provider.

### 2. Create the IAM Role

1. Go to IAM > Roles > Create role.
2. Select Web identity for Trusted entity type.
3. Select token.actions.githubusercontent.com for Identity provider.
4. Select sts.amazonaws.com for Audience.
5. Enter electricbookworks for GitHub organization (leave repo and branch options blank) and click Next.
6. Add the AmazonS3FullAccess and CloudFrontFullAccess permission policies and click Next.
7. Enter GitHubActionsS3Access for Role name and click Create role.
8. Copy the Role ARN and add this to the aws.role-arn property in `deploy.config.json` (see below).

## Adding the workflow to your project repository

### 1. Add the consuming workflow file

Create `.github/workflows/deploy.yml` in your Electric Book project and/or media repo:

```yaml
name: Deploy to Electric Book Server and S3

on:
  push: # Trigger on any push, but branch validation will filter
  workflow_dispatch: # Allow manual triggering

jobs:
  deploy:
    uses: electricbookworks/electric-book-deploy-workflow/.github/workflows/deploy.yml@{release-version}
    secrets: inherit
```

Replace `{release-version}` with the latest release (e.g. `v2.0.2`), unless you want to use an older version.

You can also provide the following optional inputs, under the `with` property:

```yaml
name: Deploy to Electric Book Server and S3

on:
  push: # Trigger on any push, but branch validation will filter
  workflow_dispatch: # Allow manual triggering

jobs:
  deploy:
    uses: electricbookworks/electric-book-deploy-workflow/.github/workflows/deploy.yml@{release-version}
    secrets: inherit
    with:
      config-file: '.github/workflows/deploy.config.json' # optional - defaults to '.github/workflows/deploy.config.json'
      node-version: '22' # optional - defaults to 22
      runner: 'Linux-4core-16gb-ram-150gb-sso' # optional - defaults to 'ubuntu-latest'
      repo-owner: 'electricbookworks' # optional - defaults to 'electricbookworks' - if not a match, workflow will be skipped
```

### 2. Create your deployment configuration

Create `.github/workflows/deploy.config.json`:

```json
{
  "aws": {
    "role-arn": "arn:aws:iam::{account_id}:role/GitHubActionsS3Access",
    "region": "eu-west-2"
  },
  "book-server": {
    "bucket-comment": "The bucket name is suffixed with -live or -staging depending on the branch.",
    "bucket": "ebt-books",
    "configs-comment": "All configs are relative to the _configs folder.",
    "configs": ["_config.live.yml"],
    "builds": [
      {
        "configs": [],
        "dir": "electric-book"
      },
      {
        "configs": ["_config.student.yml"],
        "dir": "audience/student"
      },
      {
        "configs": ["_config.teacher.yml"],
        "dir": "audience/teacher"
      }
    ]
  },
  "media" : {
    "cloudfront-comment": "The CloudFront ID is used to invalidate the cache after deployment.",
    "cloudfront-id-staging": "E7XZICS0M3W1S",
    "cloudfront-id-live": "E2KTTVJ1I6I7D8",
    "syncs": [
      {
        "bucket-comment": "The bucket name is suffixed with -live or -staging depending on the branch.",
        "bucket": "electric-book",
        "source": "assets/images/web",
        "destination": "assets/images/web",
        "options": ""
      }
    ]
  },
  "vercel": {
    "trigger": {
      "live": "https://api.vercel.com/v1/integrations/deploy/{IDs}",
      "staging": "https://api.vercel.com/v1/integrations/deploy/{IDs}"
    }
  }
}
```

In `deploy.config.json` configure the following:

1. The S3 bucket to sync the built files to.
2. Deployment configs to be used on all builds.
3. The separate builds that need to be deployed. Each build has a deployment directory that will be uploaded to the S3 bucket. If the directory already exists, it will be replaced entirely by the new deployment. You can also configure build-specific configs for each.
4. Media sync commands with S3.
5. The Vercel deploy triggers. These are specific to the project and branch of the book server your project deploys to.
