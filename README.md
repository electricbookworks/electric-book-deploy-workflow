# Electric Book Deploy Workflow Plugin

This deploy workflow plugin gets used by Electric Book projects and media repos with automated deploys to an Electric Book Server instance and/or S3 buckets.

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

Replace `{release-version}` with the latest release (e.g. `v1.0.1`), unless you want to use an older version.

You can also add optional inputs to customise the Ruby and Node versions, as well as the config file location:

```yaml
name: Deploy to Electric Book Server and S3

on:
  push: # Trigger on any push, but branch validation will filter
  workflow_dispatch: # Allow manual triggering

jobs:
  deploy:
    uses: electricbookworks/electric-book-deploy-workflow/.github/workflows/deploy.yml@{release-version}
    with:
      config-file: '.github/workflows/deploy.config.json' # optional - defaults to '.github/workflows/deploy.config.json'
      node-version: '22' # optional - defaults to 22
      ruby-version: '3.2' # optional - defaults to 3.2
    secrets: inherit
```

### 2. Create your deployment configuration

Create `.github/workflows/deploy.config.json`:

```json
{
  "book-server": {
    "repo": "electricbookworks/electric-book-server-template",
    "branches": [
      "staging",
      "live"
    ],
    "configs-comment": "All configs are relative to the _configs folder.",
    "configs": ["_config.live.yml"],
    "builds": [
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
    "branches-comment": "Only live and staging branches will trigger these syncs. This is not configurable.",
    "region": "eu-west-2",
    "syncs": [
      {
        "bucket-comment": "The bucket name is suffixed with -live or -staging depending on the branch.",
        "bucket": "electric-book",
        "source": "assets/images/web",
        "destination": "assets/images/web",
        "options": ""
      }
    ]
  }
}
```

In `deploy.config.json` configure the following:

1. The book server repo. This is an [Electric Book Server](https://github.com/electricbookworks/electric-book-server-template) instance (e.g. `core-book-server`) that serves projects from its `public` folder and is configured for its own continuous deployment. 
2. The branches that will trigger deployments on new commits. The workflow will push to the same branch on the deployment repo, matching `main` to `master` and vica versa.
3. Deployment configs to be used on all builds.
4. The separate builds that need to be deployed. Each build has a deployment directory that will be pushed to the deployment repo's `public` directory. If the directory already exists, it will be replaced entirely by the new deployment. You can also configure build-specific configs for each.
5. Media sync commands with S3.
