# actions

Shared Github Actions Workflows

Most jobs expect a Personal Access Token to be set as an organizational secret named `GH_ORG_TOKEN` which has permissions for:
* repo
* write:packages

You can generate this token here: https://github.com/settings/tokens/new?scopes=repo,read:packages

## Example

### Github Release

*.github/workflows/release.yaml*
```yaml
name: release
on:
  push:
    branches:
    - main

jobs:

  release:
    uses: CloudNativeEntrepreneur/actions/.github/workflows/github-release.yaml@main
    secrets: inherit
    with:
      docker: true # optional, set to build/release a Dockerfile
      helm: true # optional, set to release helm chart
```

### Gitops Preview + Helm Quality

*.github/workflows/pr.yaml*
```yaml
name: pr

on:

  pull_request:
    branches:
      - main

jobs:

  helm-quality:
    uses: CloudNativeEntrepreneur/actions/.github/workflows/helm-quality.yaml@main
    secrets: inherit
    with:
      helm_path: helm
  
  preview-helm-quality:
    uses: CloudNativeEntrepreneur/actions/.github/workflows/helm-quality.yaml@main
    secrets: inherit
    with:
      helm_path: preview/helm

  preview:
    needs:
    - helm-quality
    - preview-helm-quality
    uses: CloudNativeEntrepreneur/actions/.github/workflows/gitops-preview.yaml@main
    secrets: inherit
    with:
      environment_repository: CloudNativeEntrepreneur/example-preview-envs
      comment: |
        Your preview environment has been published! :rocket:

        It may take a few minutes to spin up.

        You can view the Preview contents with `kubectl`:

        ```bash
        kubectl get ns ${{ github.event.repository.name }}-pr-${{ github.event.pull_request.number }}-preview
        kubectl get sa -n ${{ github.event.repository.name }}-pr-${{ github.event.pull_request.number }}-preview
        kubectl get externalsecret -n ${{ github.event.repository.name }}-pr-${{ github.event.pull_request.number }}-preview
        ```
```

### Gitops Preview Cleanup

*.github/workflows/pr-close.yaml*

```yaml
name: pr-close
on:
  pull_request:
    types: [ closed ]

jobs:
  
  preview-cleanup:
    uses: CloudNativeEntrepreneur/actions/.github/workflows/gitops-preview-cleanup.yaml@main
    secrets: inherit
    with:
      environment_repository: CloudNativeEntrepreneur/example-preview-envs
```

### Node Quality

*.github/workflows/pr.yaml*
```yaml
name: Pull Request

on:
  pull_request:
    branches: [ main ]

jobs:

  quality:
    uses: CloudNativeEntrepreneur/actions/.github/workflows/node-quality.yaml@main
```

### Node Semantic Release

Release an NPM library with semantic-release.

*.github/workflows/release.yaml*
```yaml
name: Release

on:
  push:
    branches:
      - main

jobs:
  
  release:
    uses: CloudNativeEntrepreneur/actions/.github/workflows/node-semantic-release.yaml@main
    secrets: inherit
```

