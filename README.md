# actions

Shared Github Actions Workflows

## Example

### Github Release

```yaml
name: release
on:
  push:
    branches:
    - main

jobs:

  release:
    uses: CloudNativeEntrepreneur/actions/.github/workflows/github-release.yaml@main
    secrets:
      GH_ORG_TOKEN: ${{ secrets.GH_ORG_TOKEN }}
```

### Node Quality

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