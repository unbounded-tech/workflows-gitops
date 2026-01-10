# ArgoCD Promote Helm Workflow

This repository contains a reusable GitHub Actions workflow for promoting Helm charts to different environments using ArgoCD. The workflow automates the process of generating ArgoCD Application manifests from Helm charts and committing them to environment-specific GitOps repositories.

## Overview

The `argocd-promote-helm` workflow is designed to streamline the promotion of applications across multiple environments (e.g., local, staging, production, previews) by leveraging ArgoCD's GitOps approach with the "Application of Applications" pattern.

Environments are often ephemeral and imperative. This is a code smell! It is an important part of your domain and should be modeled explicitly!

This workflow was made with Trunk Based Development in mind, but it's not limited to that. For deeper context and alignment with how everything fits together, check out:
- [A Guide to Git with Trunk Based Development](https://patrickleet.medium.com/a-guide-to-git-with-trunk-based-development-1e6345c65c78) - In this guide, Patrick Lee Scott describes how to use repositories to model the concepts of environments explicitly.

### What is an Environment?

In this workflow, an **environment** is a directory in a GitOps repository that ArgoCD syncs using the [App of Apps pattern](https://argo-cd.readthedocs.io/en/stable/operator-manual/cluster-bootstrapping/#app-of-apps-pattern). Each environment directory contains ArgoCD Application manifests that define what should be deployed.

**Permanent environments** (e.g., staging, production) typically use dedicated repositories where the root directory serves as the environment:
```
your-org/staging-env/
├── applications/
│   ├── app-a.yaml
│   ├── app-b.yaml
│   └── ...
```

**Preview environments** typically share a single "previews" repository, where each preview gets its own directory:
```
your-org/previews-env/
├── app-a-pr-123/
│   └── app-a.yaml
├── app-b-pr-456/
│   └── app-b.yaml
├── app-a-feature-branch/
│   └── app-a.yaml
└── ...
```

### How Promotion Works

Each application repository contains a **promotion chart** (default: `.gitops/promote/helm`) that defines how the application should be represented as an ArgoCD Application manifest. This chart is a Helm template that generates the Application resource.

When you **promote** an application, this workflow:
1. Renders the promotion chart with the specific version/tag being promoted
2. Commits the resulting Application manifest to the environment repository

This adds your application (at a specific version) to the array of applications that make up that environment. ArgoCD then syncs the environment, deploying the promoted version.

```
┌─────────────────────────┐         ┌─────────────────────────┐
│   Application Repo      │         │   Environment Repo      │
│                         │         │                         │
│  .gitops/promote/helm/  │  ──►    │  applications/          │
│    └── templates/       │ render  │    ├── app-a.yaml (v1.2)│
│        └── app.yaml     │ & commit│    ├── app-b.yaml (v3.0)│
│                         │         │    └── app-c.yaml (v2.1)│
└─────────────────────────┘         └─────────────────────────┘
```

### Learn More

### Key Features
- Generates ArgoCD Application manifests from Helm charts
- Supports merging existing values with new ones
- Handles pull request creation for manual approval
- Provides preview support with automatic commenting
- Integrates with ArgoCD for automated sync

## Terminology

The workflow uses the following concepts:

| Concept | Options | Description |
|---------|---------|-------------|
| **Type** | `Release` / `Preview` | `Release` (preview=false) promotes a versioned release. `Preview` (preview=true) promotes a preview with extra resources (usually a dynamically created environment). |
| **Method** | `Promotion` / `Promotion PR` | `Promotion` pushes directly to the environment repo. `Promotion PR` creates a PR for review before merging. |
| **Event Mode** | `PR Event` / `Push Event` | Auto-detected from `github.event_name`. Affects naming and PR commenting for previews. |

### Configuration Matrix

| Type | Method | Use Case |
|------|--------|----------|
| Release | Promotion | Production releases that auto-sync |
| Release | Promotion PR | Production releases requiring approval |
| Preview | Promotion | Previews that auto-sync |
| Preview | Promotion PR | Previews requiring approval |

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
| `promotion_pr` | boolean | false | false | If true, creates a PR in the environment repository instead of pushing directly (Promotion PR method) |
| `values` | string | false | "" | Additional Helm values as YAML string |
| `preview` | boolean | false | false | If true, promotes a preview (Preview type). Event mode is auto-detected. |
| `comment` | string | false | Default preview comment | Comment body for previews (PR Event only) |
| `dry_run` | boolean | false | false | If true, skip commit and push steps (useful for testing) |

## Secrets

| Secret | Required | Description |
|--------|----------|-------------|
| `GH_PAT` | true | GitHub Personal Access Token with repo and packages write permissions |

## Usage

### [Release][Promotion] - Promote on Version Tag

Use this workflow to promote releases directly to an environment when a new version tag is pushed. Changes are pushed directly to the environment repository.

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
    name: "[Release][Promotion]"
    needs: release
    uses: unbounded-tech/workflows-gitops/.github/workflows/argocd-promote-helm.yaml@v1
    secrets:
      GH_PAT: ${{ secrets.GH_ORG_ACTIONS_REPO_WRITE_PACKAGES }}
    with:
      environment_name: your-env
      environment_repository: your-org/your-env
      destination_path: .gitops/deploy
      project: your-env
      name: your-app
```

### [Release][Promotion PR] - Promote with Approval

Use this workflow when you want releases to require approval before being promoted. A PR is created in the environment repository for review.

```yaml
  promote:
    name: "[Release][Promotion PR]"
    needs: release
    uses: unbounded-tech/workflows-gitops/.github/workflows/argocd-promote-helm.yaml@v1
    secrets:
      GH_PAT: ${{ secrets.GH_ORG_ACTIONS_REPO_WRITE_PACKAGES }}
    with:
      environment_name: your-env
      environment_repository: your-org/your-env
      destination_path: .gitops/deploy
      project: your-env
      promotion_pr: true  # Creates a PR instead of pushing directly
      name: your-app
```

### [Preview][Promotion PR] - Preview on Pull Request

Use this workflow to promote previews for pull requests. The workflow will comment on the PR with the preview status. Event mode is auto-detected as `PR Event`.

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
    name: "[Preview][Promotion PR]"
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
      promotion_pr: true
      comment: |
        Your preview has been promoted!

        Access it at: https://your-app.${{ github.event.repository.name }}-pr-${{ github.event.pull_request.number }}.your-domain.com

        The current tag is: `pr-${{ github.event.pull_request.number }}-${{ github.event.pull_request.head.sha }}`
```

### [Preview][Promotion PR] - Preview on Branch Push

Use this workflow to promote previews when pushing to feature branches. This is useful when you want previews without requiring a pull request. Event mode is auto-detected as `Push Event`, so PR comments are not available.

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
    name: "[Preview][Promotion PR]"
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
      promotion_pr: true
```

### Auto-detected Event Modes for Previews

| Event Mode | Naming | PR Comments | Image Tag |
|------------|--------|-------------|-----------|
| `PR Event` | `{repo}-pr-{number}` | Yes | `pr-{number}-{sha}` |
| `Push Event` | `{repo}-{sanitized-branch}` | No | `{sanitized-branch}-{sha}` |

### [Dry Run] - Testing Without Changes

Use `dry_run: true` to test the workflow without making any changes. When triggered from a pull request, a summary comment will be posted showing what would have happened.

```yaml
  test-workflow:
    name: "[Dry Run][Preview][Promotion PR]"
    uses: unbounded-tech/workflows-gitops/.github/workflows/argocd-promote-helm.yaml@v1
    secrets:
      GH_PAT: ${{ secrets.GITHUB_TOKEN }}
    with:
      project: test-project
      environment_name: test
      environment_repository: ${{ github.repository }}
      preview: true
      promotion_pr: true
      dry_run: true
```

## Notes

- Replace `your-org`, `your-env-*`, `your-domain.com`, and other placeholders with your actual values.
- Ensure your Helm charts are structured correctly and contain the necessary ArgoCD Application templates.
- **Image Tag Handling**: In preview mode, the workflow automatically sets a sanitized `image.tag` value. Do not pass `image.tag` in the `values` input for previews as it will be overridden. The workflow sanitizes branch names to produce valid Docker tags (lowercase, no slashes, max 63 chars).
- For previews, the workflow automatically generates unique namespaces and application names based on the PR number or branch name.
- The workflow merges existing Helm values with new ones to preserve environment-specific configurations.
- When `promotion_pr: true`, changes are committed to a branch and a PR is created for review before merging.
