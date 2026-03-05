# .github
Default community file repository

## To incorporate workflow templates into other repositories, follow these steps:

In the repository where you want to use the workflow template, create a new workflow file in the `.github/workflows` directory.

Use the `uses` keyword to reference the workflow template from the `.github` repository.
Example:

```yml
name: Auto Approve Terraform Lock Files

on:
  pull_request:
    types: [opened, synchronize]

jobs:
  auto-approve:
    uses: ./.github/WORKFLOW_TEMPLATES/auto-approve-terraform-lock-files.yml
```

### Customize Workflow Templates (Optional):
If you need to customize the workflow for a specific repository, you can override or add steps in the workflow file.
Example:

```yml
name: Auto Approve Terraform Lock Files

on:
  pull_request:
    types: [opened, synchronize]

jobs:
  auto-approve:
    uses: ./.github/WORKFLOW_TEMPLATES/auto-approve-terraform-lock-files.yml
    with:
      additional-parameter: value
```

## Shared Workflows

### Deploy Build Pipeline

Deploys Terraform infrastructure for CodePipeline build pipelines.

**How it works:**
1. `terraform-plan` job runs init, validate, and plan
2. `terraform-apply` job waits for manual approval (via GitHub's `prod` environment protection)
3. After approval, apply runs automatically

**Prerequisites:**
- Configure a `prod` environment in your repository (Settings → Environments)
- Add **required reviewers** to the `prod` environment for the approval gate to work
- Without required reviewers, the apply job runs immediately after plan completes

**Terraform Version:** Centrally managed (currently `1.11.4`)

#### Required Secrets
- `TERRAFORM_MODULES_KEY`: Base64-encoded SSH key for private Terraform module access (This is already set as an org-level secret)

#### Required Repository Variables
- `TOOLING_ACCOUNT_ID`: AWS account ID for tooling
- `TOOLING_ACCOUNT_REGION`: AWS region for tooling account

#### Basic Usage

```yml
name: Deploy Build Pipeline

on:
  push:
    branches: [main]
    paths: ['build-pipeline/*']
  workflow_dispatch:

jobs:
  deploy:
    uses: MT-JEDI/.github/.github/workflows/deploy-build-pipeline.yml@main
    secrets: inherit
```

#### Custom Working Directory

```yml
jobs:
  deploy:
    uses: MT-JEDI/.github/.github/workflows/deploy-build-pipeline.yml@main
    with:
      working_directory: './infra/pipeline'
    secrets: inherit
```

---

## PR Size Labeler Example

The PR size labeler automatically labels pull requests based on their size.

### Why bother?

It's a very simple thing that can help teams manage code review workflows more effectively:

- **Review prioritization**: Quickly identify small PRs that can be reviewed quickly vs. large PRs that need more time and attention
- **Reviewer assignment**: Automatically route PRs to appropriate reviewers based on complexity (e.g., senior reviewers for large PRs)
- **Quality gates**: Enforce size limits to prevent overly large PRs that are harder to review and more likely to introduce bugs

By automatically labeling PRs as `size:xs`, `size:s`, `size:m`, `size:l`, or `size:xl`, teams can filter, sort, and prioritize pull requests more efficiently.

### Basic usage with defaults:

```yml
name: PR Size Labeler

on:
  pull_request:
    types: [opened, synchronize]

jobs:
  label-pr-size:
    uses: MT-JEDI/.github/.github/workflows/pr-size-labeler.yml@main
```

### Customized usage with specific thresholds and settings:

```yml
name: PR Size Labeler

on:
  pull_request:
    types: [opened, synchronize]

jobs:
  label-pr-size:
    uses: MT-JEDI/.github/.github/workflows/pr-size-labeler.yml@main
    with:
      xs_max_size: '10'
      s_max_size: '50'
      m_max_size: '250'
      l_max_size: '1000'
      fail_if_xl: 'true'
      files_to_ignore: 'package-lock.json,yarn.lock,pnpm-lock.yaml,*.min.js'
```
