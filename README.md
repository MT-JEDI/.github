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

## Linting Workflows

Changes under `.github/workflows/` are checked by [actionlint](https://github.com/rhysd/actionlint) (with shellcheck for `run:` blocks) via `actionlint.yml`. Run `actionlint` locally before pushing; config and narrow ignores live in `.github/actionlint.yml`.

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

---

## Node Lint

Runs `npm ci` in each npm package directory (both `package.json` and `package-lock.json` required), then ESLint and Prettier check (`npm run lint`, `npm run format:check`) — same runner so `node_modules` from install is available for lint. Nested packages such as `functions/<name>/` are discovered automatically. A manifest without a sibling lockfile fails the job. If no install roots exist, the job fails (the lint gate does not pass vacuously).

Optional **CodeArtifact**: pass `aws-role` plus `codeartifact-domain`, `codeartifact-domain-owner`, and `codeartifact-repository` (same contract as `python-unit-tests.yml`, but `aws codeartifact login --tool npm`). The reusable job assumes the role, logs npm into CodeArtifact, runs `npm ci`, then clears assumed-role env vars before lint so lint does not run with live AWS credentials. Leave `aws-role` empty for public-registry-only repos.

**Branch protection:** the caller job id must be `lint` and this workflow's job id is `lint`, so the required status context is `lint / lint`.

**Node version:** optional `node-version` input; otherwise root `.nvmrc`, root `package.json` `engines.node`, a single agreed value across packages, or `24`.

**Permissions:** callers grant `contents: read`. If you pass `aws-role`, also grant `id-token: write` (OIDC). Do not hardcode IAM ARNs in this public repo — use a caller repo org variable or secret.

### Basic usage

```yml
name: Node Lint

on:
  pull_request:
    branches: [main]

permissions:
  contents: read

jobs:
  lint:
    uses: MT-JEDI/.github/.github/workflows/node-lint.yml@v1
```

### Pin Node version

```yml
jobs:
  lint:
    uses: MT-JEDI/.github/.github/workflows/node-lint.yml@v1
    with:
      node-version: '24'
```

### CodeArtifact (optional)

```yml
name: Node Lint

on:
  pull_request:
    branches: [main]

permissions:
  contents: read
  id-token: write

jobs:
  lint:
    uses: MT-JEDI/.github/.github/workflows/node-lint.yml@v1
    with:
      aws-role: ${{ vars.GH_ACTION_OIDC_ROLE_ARN }}
      aws-region: us-east-1
      codeartifact-domain: mtjedi
      codeartifact-domain-owner: '<account-id>'
      codeartifact-repository: eda-packages
```

---

## Node Unit Tests

Runs `npm ci` and `npm test` per package that owns tests (`*.test.*`, `*.spec.*`, or files under `__tests__/`; files in nested packages belong to the nested package). Repos with no matching tests pass with a notice (Jest is not invoked). Optional `test-script` replaces the per-package test loop but still runs `npm ci` in discovered packages first.

Optional **CodeArtifact:** same inputs as Node Lint (`aws-role`, `aws-region`, `codeartifact-domain`, `codeartifact-domain-owner`, `codeartifact-repository`). Install uses `aws codeartifact login --tool npm` and `npm ci`; assumed-role credentials are cleared before `npm test` so the test step keeps fake `AWS_*` isolation for live-call guards.

Combines Jest `coverage/coverage-summary.json` line counts (**requires `json-summary` in the Jest `coverageReporters`**, e.g. `["text", "json-summary"]`; without it no summary is written and the ratchet is skipped with a notice; an unreadable or invalid summary fails the job) across packages and compares the repo-wide percentage to `coverage-baseline.txt` at the repo root (missing baseline = advisory notice only). Publishes JUnit to the PR when `junit.xml` or `test-results/*.xml` exists at any depth outside `node_modules` (configure `jest-junit` in the repo's Jest config if you want PR test summaries).

**Branch protection:** caller job id `unit-tests`; required context `unit-tests / unit-tests`.

**Permissions:** callers must grant `contents: read`, `checks: write`, and `pull-requests: write` (same as the Python unit-test workflow). Grant `id-token: write` **if and only if** you pass `aws-role`.

### Basic usage

```yml
name: Node Unit Tests

on:
  pull_request:
    branches: [main]

permissions:
  contents: read
  checks: write
  pull-requests: write

jobs:
  unit-tests:
    uses: MT-JEDI/.github/.github/workflows/node-unit-tests.yml@v1
```

### Custom test runner script

```yml
jobs:
  unit-tests:
    uses: MT-JEDI/.github/.github/workflows/node-unit-tests.yml@v1
    with:
      test-script: scripts/run-unit-tests.sh
```

### CodeArtifact (optional)

```yml
permissions:
  contents: read
  checks: write
  pull-requests: write
  id-token: write

jobs:
  unit-tests:
    uses: MT-JEDI/.github/.github/workflows/node-unit-tests.yml@v1
    with:
      aws-role: ${{ vars.GH_ACTION_OIDC_ROLE_ARN }}
      aws-region: us-east-1
      codeartifact-domain: mtjedi
      codeartifact-domain-owner: '<account-id>'
      codeartifact-repository: eda-packages
```

---

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

### Node linting and unit tests (Telemetry)

Telemetry Node.js repos use ESLint + Prettier via `npm run lint` and `npm run format:check`, and Jest via `npm test` (typically `jest --ci --coverage`). Config lives in each package directory (`eslint.config.js`, `.prettierrc`, `jest.config.js`). CI ratchets combined line coverage against `coverage-baseline.txt` at the repo root. Run locally with the same scripts as the shared `node-lint.yml` and `node-unit-tests.yml` workflows.
