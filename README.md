# shared-workflows

Reusable GitHub Actions workflows shared across CAD repos.

## Workflows

### Shared

- `ado_comment.yml`
- `frontend_build_and_deploy.yml`
- `backend_build_and_deploy.yml`
- `update-config-fe.yml`
- `update-config-be.yml`
- `get-update-config-be-inputs.yml`
- `deploy.yml`
- `test_deployment.yml`
- `frontend_ci.yml`
- `backend_ci.yml`
- `aws_ci.yml`
- `delete_stack.yml`
- `release.yml`

### Util

- `create_shared_workflows_release.yml`

## Usage

Call reusable workflows from app repo.

### ADO comment

```yaml
jobs:
  post-comment:
    uses: oxctl/shared-workflows/.github/workflows/ado_comment.yml@master
    with:
      ado_organization: oxforduniversity
      ado_project: Canvas
    secrets:
      ado_username: ${{ secrets.ADO_USERNAME }}
      ado_personal_token: ${{ secrets.ADO_PERSONAL_TOKEN }}
```

### Full deployment

The app repository owns the triggers and GitHub environments. The shared
deployment workflow resolves `aws/<env_type>.json` from the app repository,
then enters the selected GitHub environment before either service can change
AWS (waiting for approval when required reviewers are configured). It builds
and deploys both services, updates the backend configuration, and runs
deployment tests on the default or release branch.

```yaml
on:
  workflow_dispatch:
    inputs:
      env_type:
        type: choice
        required: true
        options: [beta, prod]

jobs:
  deploy:
    uses: oxctl/shared-workflows/.github/workflows/deploy.yml@master
    permissions:
      contents: read
      id-token: write
    with:
      env_type: ${{ inputs.env_type }}
    secrets: inherit
```

Set `github_env` if the GitHub environment name differs from `env_type`, or
`aws_parameters` to use another file under `aws/` (for example, `dr` selects
`aws/dr.json`). The called jobs select that GitHub environment to read
`VITE_SENTRY_DSN`, `VITE_SENTRY_ENV`, `CANVAS_HOST`, `TOOL_SUPPORT_URL`,
`DEPLOYMENT_TEST_PATH`, and `BACKEND_URL` from environment variables. Set
`DEPLOYMENT_TESTS_OAUTH_TOKEN`, `CANVAS_TOKEN`, `TOOL_SUPPORT_USERNAME`,
`TOOL_SUPPORT_PASSWORD`, and `SECRET` as secrets in the app
repository/environment as needed. The backend configuration can read the
tool-support credentials from AWS Secrets Manager when they are absent.
The caller must pass secrets (for example with `secrets: inherit`); they are
forwarded to the nested workflows. The calling job grants `id-token: write`
and `contents: read` permissions for the AWS jobs.

### Frontend-only update-config example

```yaml
on:
  workflow_dispatch:
    inputs:
      env_type:
        type: choice
        required: true
        options: [beta, prod]

jobs:
  call-update-config:
    uses: oxctl/shared-workflows/.github/workflows/update-config-fe.yml@master
    with:
      env_type: ${{ inputs.env_type }}
    secrets: inherit
```

### Backend update-config dispatch with resolved inputs

```yaml
on:
  workflow_dispatch:
    inputs:
      env_type:
        type: choice
        required: true
        options: [beta, prod]

jobs:
  resolve-inputs:
    uses: oxctl/shared-workflows/.github/workflows/get-update-config-be-inputs.yml@master
    with:
      env_type: ${{ inputs.env_type }}

  call-update-config:
    needs: [resolve-inputs]
    uses: oxctl/shared-workflows/.github/workflows/update-config-be.yml@master
    permissions:
      contents: read
      id-token: write
    with:
      account_type: ${{ needs.resolve-inputs.outputs.account_type }}
      app_name: ${{ needs.resolve-inputs.outputs.app_name }}
      aws_account_id: ${{ needs.resolve-inputs.outputs.aws_account_id }}
      aws_region: ${{ needs.resolve-inputs.outputs.aws_region }}
      env_type: ${{ needs.resolve-inputs.outputs.env_type }}
      github_env: ${{ needs.resolve-inputs.outputs.github_env }}
    secrets: inherit
```

### Frontend CI wrapper example

```yaml
on:
  pull_request:
    branches: [master]
    paths:
      - 'frontend/**'
      - '.github/workflows/frontend*'

jobs:
  frontend-ci:
    uses: oxctl/shared-workflows/.github/workflows/frontend_ci.yml@master
```

### Backend CI wrapper example

```yaml
on:
  pull_request:
    branches: [master]
    paths:
      - 'backend/**'
      - '.github/workflows/backend*'

jobs:
  backend-ci:
    uses: oxctl/shared-workflows/.github/workflows/backend_ci.yml@master
```

### AWS template validation wrapper example

```yaml
on:
  pull_request:
    branches: [master]

jobs:
  aws-ci:
    uses: oxctl/shared-workflows/.github/workflows/aws_ci.yml@master
    with:
      aws_region: eu-west-1
      aws_account_id: 730335587339
```

### Delete stack wrapper example

```yaml
on:
  workflow_dispatch:
    inputs:
      stack-prefix:
        type: string
        required: true
      account-id:
        type: string
        required: true
      region:
        type: string
        required: true
        default: eu-west-1

jobs:
  delete-stack:
    uses: oxctl/shared-workflows/.github/workflows/delete_stack.yml@master
    with:
      stack_prefix: ${{ inputs['stack-prefix'] }}
      aws_account_id: ${{ inputs['account-id'] }}
      region: ${{ inputs.region }}
```

### Release PR wrapper example

```yaml
on:
  workflow_dispatch:

jobs:
  create-release-pr:
    uses: oxctl/shared-workflows/.github/workflows/release.yml@master
    with:
      base_branch: release
      head_branch: master
```

### Create shared-workflows SemVer release (manual dispatch)

In this repository, run **Actions → "Create shared workflows release"** and choose:

- `bump`: `patch`, `minor`, or `major`

The workflow creates and pushes the next `vX.Y.Z` tag, always moves the matching major floating tag (`vN`) to that new release, and publishes a GitHub Release with generated notes.
