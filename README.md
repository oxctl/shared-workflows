# shared-workflows

Reusable GitHub Actions workflows shared across CAD repos.

## Workflows

- `ado_comment.yml`
- `frontend_build_and_deploy.yml`
- `backend_build_and_deploy.yml`
- `update-config-fe.yml`
- `update-config-be.yml`
- `get-update-config-be-inputs.yml`
- `test_deployment.yml`
- `frontend_ci.yml`
- `backend_ci.yml`
- `aws_ci.yml`
- `delete_stack.yml`
- `release.yml`

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

### Frontend/backend example

```yaml
jobs:
  call-frontend-build-and-deploy:
    uses: oxctl/shared-workflows/.github/workflows/frontend_build_and_deploy.yml@master
    with:
      account_type: ${{ fromJSON(needs.aws-params.outputs.json).accountType }}
      app_name: ${{ fromJSON(needs.aws-params.outputs.json).appName }}
      aws_account_id: ${{ fromJSON(needs.aws-params.outputs.json).awsAccountId }}
      aws_parameters: ${{ (startsWith(github.ref_name, 'dr-')) && 'dr' || github.ref_name }}
      aws_region: ${{ fromJSON(needs.aws-params.outputs.json).awsRegion }}
      env_type: ${{ fromJSON(needs.aws-params.outputs.json).envType }}
      github_env: ${{ fromJSON(needs.aws-params.outputs.json).envType }}
      ref: ${{ github.ref }}
      sha: ${{ github.sha }}
      execution-environment: ${{ fromJSON(needs.aws-params.outputs.json).envType }}
    secrets: inherit

  call-backend-build-and-deploy:
    uses: oxctl/shared-workflows/.github/workflows/backend_build_and_deploy.yml@master
    with:
      account_type: ${{ fromJSON(needs.aws-params.outputs.json).accountType }}
      app_name: ${{ fromJSON(needs.aws-params.outputs.json).appName }}
      aws_account_id: ${{ fromJSON(needs.aws-params.outputs.json).awsAccountId }}
      aws_parameters: ${{ (startsWith(github.ref_name, 'dr-')) && 'dr' || github.ref_name }}
      aws_region: ${{ fromJSON(needs.aws-params.outputs.json).awsRegion }}
      env_type: ${{ fromJSON(needs.aws-params.outputs.json).envType }}
      github_env: ${{ fromJSON(needs.aws-params.outputs.json).envType }}
      ref: ${{ github.ref }}
      sha: ${{ github.sha }}
    secrets: inherit

  call-update-config:
    uses: oxctl/shared-workflows/.github/workflows/update-config-be.yml@master
    needs: [aws-params, call-frontend-build-and-deploy, call-backend-build-and-deploy]
    with:
      account_type: ${{ fromJSON(needs.aws-params.outputs.json).accountType }}
      app_name: ${{ fromJSON(needs.aws-params.outputs.json).appName }}
      aws_account_id: ${{ fromJSON(needs.aws-params.outputs.json).awsAccountId }}
      aws_region: ${{ fromJSON(needs.aws-params.outputs.json).awsRegion }}
      env_type: ${{ fromJSON(needs.aws-params.outputs.json).envType }}
      github_env: ${{ fromJSON(needs.aws-params.outputs.json).envType }}
    secrets: inherit

  call-deployment-tests:
    uses: oxctl/shared-workflows/.github/workflows/test_deployment.yml@master
    needs: [aws-params, call-frontend-build-and-deploy, call-backend-build-and-deploy, call-update-config]
    with:
      execution-environment: ${{ fromJSON(needs.aws-params.outputs.json).envType }}
    secrets: inherit
```

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
    with:
      working_directory: frontend
      node_version_file: frontend/.nvmrc
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
    with:
      working_directory: backend
      java_version_file: backend/.java-version
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
      stack_prefix: ${{ inputs.stack-prefix }}
      aws_account_id: ${{ inputs.account-id }}
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