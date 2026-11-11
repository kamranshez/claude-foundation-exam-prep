# Claude Code Hooks

Hooks are user-defined shell commands that Claude Code runs automatically at specific points in its lifecycle, giving deterministic control instead of relying on the model to remember an instruction.

## 1. Why It Matters

Instructions in prompts or `CLAUDE.md` are followed by the model, which can be inconsistent. Hooks are executed by the harness, so they always run. Use them to enforce formatting, block dangerous commands, log activity or inject context.

## 2. Common Hook Events

| Event | When it fires | Typical use |
|-------|---------------|-------------|
| `PreToolUse` | Before a tool call runs | Validate or block a call |
| `PostToolUse` | After a tool call succeeds | Run a formatter or linter |
| `UserPromptSubmit` | When the user submits a prompt | Add context, validate input |
| `Notification` | When Claude Code sends a notification | Desktop or chat alert |
| `Stop` | When Claude finishes responding | Run tests, final checks |
| `SubagentStop` | When a subagent finishes | Post-process subagent work |
| `SessionStart` | When a session begins or resumes | Load project context |

## 3. Configuration

Hooks live in `settings.json` (user, project or local). A matcher selects which tools trigger the hook.

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          { "type": "command", "command": "npx prettier --write \"$CLAUDE_FILE_PATH\"" }
        ]
      }
    ]
  }
}
```

| Setting file | Scope |
|--------------|-------|
| `~/.claude/settings.json` | All your projects |
| `.claude/settings.json` | Shared with the team (committed) |
| `.claude/settings.local.json` | Personal, not committed |

## 4. How Hooks Communicate

- The hook receives event details as JSON on standard input.
- **Exit code 0**: success; the action proceeds.
- **Exit code 2**: blocking error; for `PreToolUse` the tool call is blocked and stderr is fed back to Claude.
- **Other exit codes**: non-blocking error; execution continues.
- Hooks can also print structured JSON to control decisions (for example allow, deny or add context).

## 5. Best Practices

- Keep hooks fast; they run synchronously in the loop.
- Treat hook input as untrusted and quote variables to avoid shell injection.
- Use `PreToolUse` for guardrails and `PostToolUse` for cleanup.
- Hooks run with your user permissions, so review project hooks before trusting a repo.
- Use hooks, not prompt text, for anything that must happen every time.

## Key Takeaways

- Hooks give deterministic, harness-enforced behavior around Claude Code's lifecycle.
- `PreToolUse` can block actions; `PostToolUse` reacts to completed ones.
- Exit code 2 blocks and returns stderr to Claude; 0 means success.
- Configure them in `settings.json`, with matchers to target specific tools.
- Hooks execute arbitrary commands, so security review matters.

## Practice Questions

**1. Why use a hook instead of a `CLAUDE.md` instruction to run a formatter?**
Hooks are executed by the harness every time; a `CLAUDE.md` instruction depends on the model following it.

**2. Which event lets you stop a tool call before it runs?**
`PreToolUse`, by exiting with code 2 (or returning a deny decision).

**3. How does a hook receive information about the event?**
As JSON on standard input.

**4. What happens when a `PreToolUse` hook exits with code 2?**
The tool call is blocked and the hook's stderr is shown to Claude so it can adjust.

**5. Where would you put a hook that the whole team should share?**
In the project's `.claude/settings.json`, committed to the repository.
