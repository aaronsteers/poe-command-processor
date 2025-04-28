# Poe Command Processor

A GitHub Action to execute [Poe the Poet](https://github.com/nat-n/poethepoet) commands in your repository, designed for use with slash commands in PR comments. This action can automatically commit changes back to the pull request branch or create a new PR if needed.

## Features

- ✅ Runs any Poe command in your repository.
- ✅ Supports triggering via slash commands in PR and issue comments.
- ✅ Can auto-commit changes to the PR branch, or open a new PR if no PR exists.
- ✅ Posts status updates and results as comments on the originating comment.
- ✅ Infers commands from comment bodies for seamless gitops/chatops workflows.

## Inputs

| Name           | Description                                                                 | Required | Default  |
|----------------|-----------------------------------------------------------------------------|----------|----------|
| `command`      | Poe command to run. If not provided, inferred from the body of the specified `comment-id`. | false    |          |
| `pr`           | Pull Request number.                                                        | false    |          |
| `comment-id`   | Comment ID (for reply chaining and command inference).                      | false    |          |
| `github-token` | GitHub Token. Required for CI to run after commits are pushed.              | false    |          |
| `no-commit`    | Disable auto-commit step.                                                   | false    | `false`  |

## Usage

### Basic Example

```yaml
- name: Run Poe Command
  uses: aaronsteers/poe-command-processor@v1
  with:
    command: "lint"
    github-token: ${{ secrets.GITHUB_TOKEN }}
    pr: ${{ github.event.pull_request.number }}
```

### Slash Command Example

This action is designed to work with slash commands in PR comments. If you omit the `command` input and provide a `comment-id`, the action will extract the command from the comment body.

```yaml
- name: Run Poe Command from Comment
  uses: aaronsteers/poe-command-processor@v1
  with:
    comment-id: ${{ github.event.comment.id }}
    pr: ${{ github.event.issue.number }}
    github-token: ${{ secrets.GITHUB_TOKEN }}
```

If the comment body is `/poe lint`, the action will run `poe lint`.

### Auto-Commit and PR Creation

- If changes are made and `no-commit` is not set to `true`, the action will auto-commit changes to the PR branch.
- If no PR is provided, the action will create a new draft PR with the results.
- Status and result comments are posted back to the PR or issue thread.

## Requirements

- Your repository must use [Poe the Poet](https://github.com/nat-n/poethepoet) and have a valid `pyproject.toml`,  `poe_tasks.toml`, or similar config file containing the project's Poe tasks.
- The action sets up Python 3.11 and installs dependencies using [uv](https://github.com/astral-sh/uv).
- Your project should have a poe task named `install` which will run before any other requested command.
- The `github-token` input is required for committing changes and posting comments.
- If a `.tool-versions` file exists in the root of your repository, this action will automatically use it to determine the versions of `poetry`, `python`, and `uv`, provided matching entries are found. This behavior is powered by the [marocchino/tool-versions-action](https://github.com/marocchino/tool-versions-action). No additional configuration is required to enable this feature.

## Sample Workflows

### Sample Poe Slash Command (Generic)

<details>
<summary>Show/Hide Sample Poe Workflow Files</summary>

```yaml
# .github/workflows/poe-command.yml:
name: On-Demand Poe Task

on:
  workflow_dispatch:
    inputs:
      pr:
        description: "PR Number. If omitted, a new PR will be created."
        type: string
        required: false
      comment-id:
        description: "Comment ID (Optional)"
        type: string
        required: false

permissions:
  contents: write
  pull-requests: write
  issues: write

jobs:
  run-poe-command:
    env:
      SOME_ENV_VAR: ${{ secrets.some_value }}
    runs-on: ubuntu-latest
    steps:
      - name: Run Poe Slash Command Processor
        uses: aaronsteers/poe-command-processor@v1
        with:
          pr: ${{ github.event.inputs.pr }}
          comment-id: ${{ github.event.inputs.comment-id }}
          github-token: ${{ secrets.MY_GH_PAT_TOKEN }}
```

</details>

### Sample Auto-Format Slash Command

<details>
<summary>Show/Hide Auto-Format Workflow File</summary>

```yaml
# .github/workflows/format-command.yml:
name: On-Demand Format Task

on:
  workflow_dispatch:
    inputs:
      pr:
        description: "PR Number. If omitted, a new PR will be created."
        type: string
        required: false
      comment-id:
        description: "Comment ID (Optional)"
        type: string
        required: false

permissions:
  contents: write
  pull-requests: write
  issues: write

jobs:
  run-poe-command:
    env:
      SOME_ENV_VAR: ${{ secrets.some_value }}
    runs-on: ubuntu-latest
    steps:
      - name: Run Poe Slash Command Processor
        uses: aaronsteers/poe-command-processor@v1
        with:
          command: format
          pr: ${{ github.event.inputs.pr }}
          comment-id: ${{ github.event.inputs.comment-id }}
          github-token: ${{ secrets.MY_GH_PAT_TOKEN }}
```

</details>


### Sample On-Demand Test Slash Command

<details>
<summary>Show/Hide Sample Test Workflow File</summary>

```yaml
# .github/workflows/test-command.yml:
name: On-Demand Test Task

on:
  workflow_dispatch:
    inputs:
      pr:
        description: "PR Number. If omitted, a new PR will be created."
        type: string
        required: false
      comment-id:
        description: "Comment ID (Optional)"
        type: string
        required: false

permissions:
  pull-requests: write
  issues: write

jobs:
  run-poe-command:
    env:
      SOME_ENV_VAR: ${{ secrets.some_value }}
    runs-on: ubuntu-latest
    steps:
      - name: Run Poe Slash Command Processor
        uses: aaronsteers/poe-command-processor@v1
        with:
          command: test
          no-commit: "true"  # No changes expected from 'test' task
          pr: ${{ github.event.inputs.pr }}
          comment-id: ${{ github.event.inputs.comment-id }}
          github-token: ${{ secrets.MY_GH_PAT_TOKEN }}
```

</details>


### Sample Slash Command Dispatch Workflow

<details>
<summary>Show/Hide Sample Slash Command Dispatch File</summary>


```yaml
# .github/workflows/slash-command-dispatch.yml:
name: Slash Command Dispatch

on:
  issue_comment:
    types: [created]

jobs:
  slashCommandDispatch:
    # Only allow slash commands on pull request (not on issues)
    runs-on: ubuntu-latest
    steps:
      - name: Slash Command Dispatch
        id: dispatch
        uses: peter-evans/slash-command-dispatch@v4
        with:
          repository: ${{ github.repository }}
          token: ${{ github.token }}
          dispatch-type: workflow
          issue-type: both
          commands: |
            poe
            format
            test
          static-args: |
            comment-id=${{ github.event.comment.id }}
            pr=${{ github.event.issue.pull_request != null && github.event.issue.number || '' }}
          # Only run for users with 'write' permission on the main repository
          permission: write

      - name: Edit comment with error message
        if: steps.dispatch.outputs.error-message
        uses: peter-evans/create-or-update-comment@v4
        with:
          comment-id: ${{ github.event.comment.id }}
          body: |
            > Error: ${{ steps.dispatch.outputs.error-message }}

```

<details>

## License

This project is licensed under the terms of the [MIT License](LICENSE).
