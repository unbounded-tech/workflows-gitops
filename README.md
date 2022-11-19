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