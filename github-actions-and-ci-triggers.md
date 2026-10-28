# GitHub Actions and CI Triggers for Claude Code

A guide for any agent (or human) that needs to understand how GitHub Actions workflows start, and how Claude Code plugs into them. See also [ci-notes.md](ci-notes.md).

## 1. GitHub Actions in One Minute

| Term | Meaning |
|------|---------|
| Workflow | A YAML file in `.github/workflows/` |
| Event / trigger | What starts a workflow (`on:`) |
| Job | A group of steps that runs on one runner |
| Step | A shell command (`run:`) or a reusable action (`uses:`) |
| Runner | The machine executing the job (`ubuntu-latest`) |
| Secret | Encrypted value, read as `${{ secrets.NAME }}` |

Lifecycle: event fires, GitHub matches it against workflows, the `if:` filter is evaluated, a runner is provisioned, and the steps run. Results show up as checks on the commit or PR.

## 2. Common Triggers

| Trigger | Fires when | Typical use |
|---------|-----------|-------------|
| `push` | Commits are pushed | Build and test every branch |
| `pull_request` | PR opened, synchronized, or reopened | Validate changes before merge |
| `issues` | Issue opened, edited, assigned, labeled | Triage, agent kickoff |
| `issue_comment` | Comment on an issue **or PR** | `@claude` mentions |
| `pull_request_review` | Review submitted | React to review text |
| `pull_request_review_comment` | Inline comment on a diff | `@claude` on specific lines |
| `workflow_dispatch` | Manual run from the UI/CLI | On-demand jobs |
| `schedule` | Cron expression | Nightly checks |
| `workflow_call` | Called by another workflow | Reusable pipelines |

Refine triggers with `types:`, `branches:`, and `paths:`:

```yaml
on:
  pull_request:
    types: [opened, synchronize]
    branches: [main]
    paths: ['**.md']
```

Gotchas:

- `issue_comment` fires for PR comments too, and it runs the workflow file from the **default branch**, so changes to a workflow only take effect for that trigger after merging.
- `pull_request` from forks gets a read-only token and no secrets.
- Events created with the default `GITHUB_TOKEN` do not start new workflow runs (this prevents loops).
- Use an `if:` on the job to filter further, e.g. `contains(github.event.comment.body, '@claude')`.

## 3. How Claude Code Is Triggered

`anthropics/claude-code-action@v1` runs Claude Code inside a workflow in two modes:

1. **Interactive (tag) mode**: no `prompt` is set. A trigger phrase (`@claude`) in an issue, comment, or review makes Claude read the request, work on a branch, and report in one updating comment.
2. **Automation mode**: a `prompt` is set. Claude runs on any trigger (`pull_request`, `schedule`, `workflow_dispatch`) without a mention, e.g. for automatic PR review.

### Interactive example (simplified from this repo's `claude.yml`)

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

jobs:
  claude:
    if: contains(github.event.comment.body, '@claude')   # simplified
    runs-on: ubuntu-latest
    permissions:
      contents: read
      pull-requests: read
      issues: read
      id-token: write
      actions: read
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 1
      - uses: anthropics/claude-code-action@v1
        with:
          claude_code_oauth_token: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
```

The real workflow uses a per-event `if:` because the comment body lives at a different path for each event (`comment.body`, `review.body`, `issue.body`).

### Automation example (PR review on every push)

```yaml
on:
  pull_request:
    types: [opened, synchronize]

jobs:
  review:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      pull-requests: write
      id-token: write
    steps:
      - uses: actions/checkout@v4
      - uses: anthropics/claude-code-action@v1
        with:
          claude_code_oauth_token: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
          prompt: 'Review this PR for bugs and style issues.'
```

## 4. Event-to-Behavior Map

| Event | What Claude does |
|-------|------------------|
| Issue opened/assigned containing `@claude` | Reads the issue, works on a `claude/issue-<n>-<timestamp>` branch, and links a PR URL |
| Comment `@claude ...` on an issue | Treats that comment as the instruction |
| Comment on a PR | Pushes to the existing PR branch |
| Inline review comment | Answers or fixes the specific code |
| `pull_request` + `prompt` | Runs the prompt automatically (review, summary, checks) |

## 5. Configuration Cheat Sheet

| Input | Purpose |
|-------|---------|
| `claude_code_oauth_token` / `anthropic_api_key` | Authentication; always from secrets |
| `prompt` | Switches to automation mode with fixed instructions |
| `claude_args` | CLI flags: `--allowed-tools`, `--model`, `--max-turns` |
| `trigger_phrase` | Replace `@claude` |
| `additional_permissions` | e.g. `actions: read` to read CI results |

Restrict tools: `claude_args: '--allowed-tools "Bash(gh pr *)" --max-turns 10'`.

## 6. Guidance for Any Agent

- Read the repo `CLAUDE.md` first; it holds the conventions and constraints.
- Only the triggering comment contains instructions; other comments are context.
- Work on the provided branch; do not create new ones on issue triggers.
- Cannot edit `.github/workflows/` (GitHub App permission), approve PRs, or post multiple comments.
- Check CI after pushing: `gh pr checks`, `gh run list`, `gh run view <id> --log-failed`.
- Use least-privilege `permissions:`, keep secrets in repo secrets, and review agent PRs like any contributor's.

## 7. Quick Troubleshooting

| Symptom | Likely cause |
|---------|--------------|
| Workflow does not run | Trigger/`types`/`paths` mismatch, or workflow not on the default branch (for comment events) |
| Claude does not respond | No `@claude` in the triggering text, or `if:` filter too strict |
| Auth error | Missing or wrong `CLAUDE_CODE_OAUTH_TOKEN` / `ANTHROPIC_API_KEY` secret |
| Cannot push or comment | `permissions:` too narrow |
| No follow-up workflow after Claude pushes | Events from `GITHUB_TOKEN` do not trigger workflows |
