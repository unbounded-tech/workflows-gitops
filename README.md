# ArgoCD Promote Helm Workflow

This repository contains a reusable GitHub Actions workflow for promoting Helm charts to different environments using ArgoCD. The workflow automates the process of generating ArgoCD Application manifests from Helm charts and committing them to environment-specific GitOps repositories.

## Overview

The `argocd-promote-helm` workflow is designed to streamline the deployment of applications across multiple environments (e.g., local, staging, production) by leveraging ArgoCD's GitOps approach. It supports both promotion workflows (e.g., on version tags) and preview environments (e.g., on pull requests).

Key features:
- Generates ArgoCD Application manifests from Helm charts
- Supports merging existing values with new ones
- Handles pull request creation for manual approval
- Provides preview environment support with automatic commenting
- Integrates with ArgoCD for automated deployments

## Prerequisites

- ArgoCD installed and configured in your cluster
- GitOps repositories for each environment
- Promotion and Preview Helm charts prepared in your application repository
- GitHub repository with appropriate permissions
- Personal Access Token (PAT) with repo and packages write access

## Inputs

| Input | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `name` | string | false | Repository name | Name of the application |
| `promotion_chart_path` | string | false | `.gitops/promote/helm` | Path to the Helm chart for promotion |
| `destination_path` | string | false | `.gitops/deploy/helm/templates` | Path in the environment repo where manifests are written |
| `project` | string | true | - | ArgoCD project name |
| `environment_name` | string | true | - | Environment name for GitHub environment protection |
| `environment_repository` | string | true | - | GitOps repository for the environment |
| `create_pull_request` | boolean | false | - | Whether to create a PR instead of direct push |
| `values` | string | false | "" | Additional Helm values as YAML string |
| `preview` | boolean | false | false | Enable preview mode for PR environments |
| `preview_mode` | string | false | `pr` | Preview mode: `pr` for pull request previews, `push` for branch previews |
| `comment` | string | false | Default preview comment | Comment body for preview PRs |
| `comment_body_includes_text` | string | false | "argocd-promote-helm-preview" | Text to find existing comments for replacement |
| `dry_run` | boolean | false | false | If true, skip commit and push steps (useful for testing) |

## Secrets

| Secret | Required | Description |
|--------|----------|-------------|
| `GH_PAT` | true | GitHub Personal Access Token with repo and packages write permissions |

## Usage

### Promotion Workflow (On Version Tags)

Use this workflow to promote releases to different environments when a new version tag is pushed.

```yaml
name: on-version-tag
on:
  push:
    tags:
    - v*.*.*

permissions:
  contents: write
  packages: write
  issues: write
  pull-requests: write

jobs:
  publish:
    uses: unbounded-tech/workflows-containers/.github/workflows/publish.yaml@v1.1.1
    with:
      dockerfiles: |
        [
          {
            "prefix": "",
            "dockerfile": "./Dockerfile",
            "postfix": ""
          }
        ]

  release:
    needs: publish
    uses: unbounded-tech/workflow-simple-release/.github/workflows/workflow.yaml@v1.3.0
    with:
      tag: ${{ github.ref_name }}
      name: ${{ github.ref_name }}

  promote:
    needs: release
    uses: unbounded-tech/workflows-gitops/.github/workflows/argocd-promote-helm.yaml@v1.0.0
    secrets:
      GH_PAT: ${{ secrets.GH_ORG_ACTIONS_REPO_WRITE_PACKAGES }}
    with:
      environment_name: your-env
      environment_repository: your-org/your-env
      destination_path: .gitops/deploy
      project: your-env
      create_pull_request: false
      name: your-app
```

### Preview Workflow - PR Mode (On Pull Requests)

Use this workflow to create preview environments for pull requests. The workflow will comment on the PR with the preview status.

```yaml
name: on-pr

on:
  pull_request:
    branches:
      - main
    types:
      - labeled
      - opened
      - reopened
      - synchronize

jobs:
  publish-containers:
    uses: unbounded-tech/workflows-containers/.github/workflows/publish.yaml@v1.1.1
    permissions:
      packages: write
      contents: read
      pull-requests: write
    with:
      dockerfiles: |
        [
          {
            "prefix": "",
            "dockerfile": "./Dockerfile",
            "postfix": ""
          }
        ]

  preview:
    needs:
      - publish-containers
    if: contains(github.event.pull_request.labels.*.name, 'preview')
    uses: unbounded-tech/workflows-gitops/.github/workflows/argocd-promote-helm.yaml@v1
    secrets:
      GH_PAT: ${{ secrets.GH_ORG_ACTIONS_REPO_WRITE_PACKAGES }}
    permissions:
      packages: write
      contents: write
      issues: write
      pull-requests: write
    with:
      promotion_chart_path: .gitops/preview/helm
      name: ${{ github.event.repository.name }}
      environment_repository: your-org/your-previews-env
      environment_name: your-previews-env
      project: your-previews-env
      preview: true
      preview_mode: pr
      create_pull_request: true
      comment: |
        <!-- argocd-promote-helm-preview -->
        Your preview environment has been deployed! :rocket:

        Access it at: https://your-app.${{ github.event.repository.name }}-pr-${{ github.event.pull_request.number }}.your-domain.com

        The current tag is: `pr-${{ github.event.pull_request.number }}-${{ github.event.pull_request.head.sha }}`
```

### Preview Workflow - Push Mode (On Branch Push)

Use this workflow to create preview environments when pushing to feature branches. This is useful when you want previews without requiring a pull request. Note that PR comments are not available in push mode.

```yaml
name: on-push

on:
  push:
    branches:
      - '**'
      - '!main'

jobs:
  publish-containers:
    uses: unbounded-tech/workflows-containers/.github/workflows/publish.yaml@v1.1.1
    permissions:
      packages: write
      contents: read
    with:
      dockerfiles: |
        [
          {
            "prefix": "",
            "dockerfile": "./Dockerfile",
            "postfix": ""
          }
        ]

  preview:
    needs:
      - publish-containers
    uses: unbounded-tech/workflows-gitops/.github/workflows/argocd-promote-helm.yaml@v1
    secrets:
      GH_PAT: ${{ secrets.GH_ORG_ACTIONS_REPO_WRITE_PACKAGES }}
    permissions:
      packages: write
      contents: write
    with:
      promotion_chart_path: .gitops/preview/helm
      environment_repository: your-org/your-previews-env
      environment_name: your-previews-env
      project: your-previews-env
      preview: true
      preview_mode: push
      create_pull_request: true
```

**Key differences between PR and Push modes:**

| Feature | PR Mode (`preview_mode: pr`) | Push Mode (`preview_mode: push`) |
|---------|------------------------------|----------------------------------|
| Trigger | Pull request events | Push to branch |
| Naming | `{repo}-pr-{number}` | `{repo}-{sanitized-branch}` |
| PR Comments | Yes | No |
| Image Tag | `pr-{number}-{sha}` | `{sanitized-branch}-{sha}` |

### Dry Run Mode

Use `dry_run: true` to test the workflow without making any changes. In PR mode, a summary comment will be posted showing what would have happened.

```yaml
  test-workflow:
    uses: unbounded-tech/workflows-gitops/.github/workflows/argocd-promote-helm.yaml@v1
    secrets:
      GH_PAT: ${{ secrets.GITHUB_TOKEN }}
    with:
      project: test-project
      environment_name: test
      environment_repository: ${{ github.repository }}
      preview: true
      preview_mode: pr
      create_pull_request: true
      dry_run: true
```

## Notes

- Replace `your-org`, `your-env-*`, `your-domain.com`, and other placeholders with your actual values.
- Ensure your Helm charts are structured correctly and contain the necessary ArgoCD Application templates.
- **Image Tag Handling**: In preview modes, the workflow automatically sets a sanitized `image.tag` value. Do not pass `image.tag` in the `values` input for preview modes as it will be overridden. The workflow sanitizes branch names to produce valid Docker tags (lowercase, no slashes, max 63 chars).
- For preview environments, the workflow automatically generates unique namespaces and application names based on the PR number.
- The workflow merges existing Helm values with new ones to preserve environment-specific configurations.
- When `create_pull_request` is true, changes are committed to a branch and a PR is created for review before merging.
