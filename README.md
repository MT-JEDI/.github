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