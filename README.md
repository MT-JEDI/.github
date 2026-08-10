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

## Versioning

Shared workflows are versioned with [SemVer](https://semver.org/) tags. Consumer repos should pin to the **major alias** (currently `@v1`), not `@main`:

```yml
uses: MT-JEDI/.github/.github/workflows/<workflow>.yml@v1
```

| Change type | Example commit | Bump |
|-------------|----------------|------|
| Breaking (required input removed/renamed, behavior change) | `feat!: ...` or `feat(scope)!: ...` | major (`v2.0.0`, alias `v2`) |
| New workflow or optional input | `feat: add timeout input to draft gate` | minor (`v1.1.0`) |
| Fix or non-breaking tweak | `fix: paginate review thread query` | patch (`v1.0.1`) |

On every conventional-commit squash merge to `main`, [release.yml](.github/workflows/release.yml):

1. Creates an annotated tag `vX.Y.Z`
2. Force-moves the major alias `vX` to the same commit (so `@v1` always resolves to the latest v1.x release)
3. Publishes a GitHub Release with generated notes

PR titles must follow conventional commits (same types as `semantic-pr-title.yml`); merges with non-conventional subjects do not produce a release.

**Squash-merge is required.** [release.yml](.github/workflows/release.yml) reads the merge commit subject on `main`. A regular merge commit (`Merge pull request #N ...`) or a rebase merge does not match conventional commits and skips the release. This repo must allow **squash merge only**, with the squash commit title set to the **PR title** (not the underlying commit message).

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
    uses: MT-JEDI/.github/.github/workflows/deploy-build-pipeline.yml@v1
    secrets: inherit
```

#### Custom Working Directory

```yml
jobs:
  deploy:
    uses: MT-JEDI/.github/.github/workflows/deploy-build-pipeline.yml@v1
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
    uses: MT-JEDI/.github/.github/workflows/pr-size-labeler.yml@v1
```

### Customized usage with specific thresholds and settings:

```yml
name: PR Size Labeler

on:
  pull_request:
    types: [opened, synchronize]

jobs:
  label-pr-size:
    uses: MT-JEDI/.github/.github/workflows/pr-size-labeler.yml@v1
    with:
      xs_max_size: '10'
      s_max_size: '50'
      m_max_size: '250'
      l_max_size: '1000'
      fail_if_xl: 'true'
      files_to_ignore: 'package-lock.json,yarn.lock,pnpm-lock.yaml,*.min.js'
```

## Team Conventions (DIoTS)

### PR titles: conventional commits (enforced by CI)

PR titles must follow [Conventional Commits](https://www.conventionalcommits.org/): `<type>(optional-scope): description`

Allowed types: `feat` `fix` `chore` `docs` `test` `refactor` `ci` `perf` `build` `revert`

Examples:
- `feat: firmware download-url discovery API`
- `fix(ble-scanner): apply shadow config before first scan`
- `feat(ADO-238821): move user_provided location to config shadow`

We squash-merge, so the PR title becomes the commit subject on `main`. The `Semantic PR Title` check enforces this on every PR in iot-* repos.

### Branch naming (guidance, not enforced)

`<type>/<ticket-or-slug>`, e.g. `feat/ADO-238821-config-shadow`, `fix/ble-scanner-boot-config`, `chore/sdlc-standardization`.

### PR size

The PR size labeler tags every PR (`size:xs` through `size:xl`, xl = >900 changed lines excluding lockfiles). Keep PRs under 900 lines; split stacked work into numbered PRs (see iot-firmware-management's `(1/4)`...`(4/4)` distro series for the pattern). **Planned for mid-August 2026:** `size:xl` PRs will fail the check and block merge.

### ADO linkage

Reference the ADO work item in the PR body (the template prompts for it). Optionally use the scope for it: `feat(ADO-12345): ...`. Not CI-enforced.

### Python linting

All iot-* repos lint with Ruff (rules `E`, `F`, `I`; `ruff format` for style). Config is `ruff.toml` at each repo root. Run locally with:

```
pip install ruff==0.15.6
ruff check . && ruff format --check .
```
