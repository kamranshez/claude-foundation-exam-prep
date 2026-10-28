# CI Notes: Claude as a CI Worker

Exam-prep notes on using Claude Code in GitHub-based CI.

## 1. Get the GitHub CLI (`gh`)

`gh` is GitHub's official CLI. It is used to clone repos and to work with issues, PRs and Actions runs from the terminal.

### Install

| Platform | Command |
|----------|---------|
| macOS | `brew install gh` |
| Windows | `winget install --id GitHub.cli` |
| Debian/Ubuntu | `sudo apt install gh` (or add the official GitHub apt repo) |
| Fedora | `sudo dnf install gh` |

### Authenticate

```bash
gh auth login      # follow the prompts (browser or token)
gh auth status     # verify
```

### Commands worth knowing

```bash
gh repo clone kamranshez/claude-foundation-exam-prep
gh issue list
gh issue view 4
gh pr create --base main --fill
gh pr checks            # CI status for the current PR
gh run list             # recent workflow runs
gh run view <run-id> --log-failed
```

## 2. CI Concept Using a GitHub Project

CI (Continuous Integration) means every change is automatically built and checked when it is pushed or when a PR is opened. On GitHub this is done with **GitHub Actions**.

### Core concepts

- **Workflow**: a YAML file in `.github/workflows/`.
- **Event / trigger**: what starts the workflow (`push`, `pull_request`, `issue_comment`, ...).
- **Job**: a set of steps that runs on one runner.
- **Step**: a single command or a reusable **action** (`uses:`).
- **Runner**: the machine that runs the job (e.g. `ubuntu-latest`).
- **Secrets**: encrypted values (e.g. tokens) referenced as `${{ secrets.NAME }}`.

### Example from this repo

`.github/workflows/markdown-only.yml` runs on every `push` and `pull_request`, and fails if any tracked file is not Markdown. That is a minimal CI check: trigger, runner, checkout, a validation script, pass/fail result.

### Using a GitHub Project (board) with CI

1. Create a Project (Projects tab, board layout) and add issues or PRs to it.
2. Use columns/status such as `Todo`, `In progress`, `In review`, `Done`.
3. Link PRs to issues (`Closes #4`) so merging moves the item to `Done` automatically.
4. Required status checks on `main` (branch protection) ensure only PRs with passing CI are merged.

Flow: issue on the board, then a branch and PR, then CI runs on the PR, then review, then merge, then the item is closed.

## 3. Live Demo: Claude as a CI Worker

What the demo shows (this very issue is an example):

1. Install the Claude GitHub app / workflow via `/install-github-app` in Claude Code, or add the workflow manually.
2. Add the `CLAUDE_CODE_OAUTH_TOKEN` (or `ANTHROPIC_API_KEY`) repository secret.
3. Open an issue and comment `@claude work on this issue and create a pr to main`.
4. The workflow `claude.yml` triggers, and Claude:
   - posts a single progress comment with a checklist,
   - works on a branch named `claude/issue-<n>-<timestamp>`,
   - commits and pushes changes,
   - links a "Create a PR" URL in its comment.
5. Other workflows (e.g. `markdown-only`) then run on the PR as normal CI.

Things to observe in the demo: the progress checklist, the job run link, the pushed branch, and CI results on the resulting PR.

## 4. In-Depth Configuration as a CI Worker

The action is `anthropics/claude-code-action@v1`.

### Triggers (`on:`)

```yaml
on:
  issue_comment:
    types: [created]
  pull_request_review_comment:
    types: [created]
  issues:
    types: [opened, assigned]
  pull_request_review:
    types: [submitted]
```

An `if:` condition on the job restricts runs to events containing `@claude`.

### Permissions

```yaml
permissions:
  contents: read
  pull-requests: read
  issues: read
  id-token: write
  actions: read   # lets Claude read CI results on PRs
```

Follow least privilege: grant only what the job needs.

### Key `with:` inputs

| Input | Purpose |
|-------|---------|
| `claude_code_oauth_token` / `anthropic_api_key` | Authentication (store as secrets, never inline) |
| `prompt` | Fixed instructions; if omitted, Claude follows the `@claude` comment |
| `claude_args` | CLI flags, e.g. `--allowed-tools`, `--model`, `--max-turns` |
| `additional_permissions` | Extra GitHub permissions, e.g. `actions: read` |
| `trigger_phrase` | Change the default `@claude` mention |

### Tool control

Limit what Claude can run:

```yaml
claude_args: '--allowed-tools "Bash(gh pr *)" --max-turns 10'
```

### CLAUDE.md

Put repo guidance (conventions, build/test commands, constraints) in a root `CLAUDE.md`. Claude reads it on every run.

### Limitations to remember

- Cannot modify files in `.github/workflows/` (GitHub App permission).
- Cannot approve PRs or submit formal reviews.
- Communicates through a single, updated comment.
- Only the triggering comment holds instructions; other comments are context.

### Security notes

- Keep tokens in repo secrets.
- Restrict who can trigger it (write access is required by default).
- Review Claude's PRs like any other contributor's.

### Related workflow in this repo

`claude-code-review.yml` runs automated code review on PRs.
