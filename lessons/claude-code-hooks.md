# Claude Code Hooks

Hooks are user-defined commands that Claude Code runs automatically at specific points in its lifecycle, giving you deterministic control instead of relying on the model to remember.

## 1. Why It Matters

Instructions in prompts or `CLAUDE.md` are advice: Claude may forget or skip them. Hooks are enforced by the harness, so they always run. Use them to format code, block dangerous commands, log activity, or send notifications.

## 2. Hook Events

| Event | Fires when | Typical use |
|-------|-----------|-------------|
| `PreToolUse` | Before a tool call runs | Block or modify risky calls |
| `PostToolUse` | After a tool call succeeds | Run formatters, linters, tests |
| `UserPromptSubmit` | User submits a prompt | Add context, validate input |
| `Notification` | Claude needs attention or permission | Desktop or Slack alerts |
| `Stop` | Claude finishes responding | Final checks, force more work |
| `SubagentStop` | A subagent finishes | Validate subagent output |
| `SessionStart` | Session begins or resumes | Load project context |
| `PreCompact` | Before context compaction | Save state |

## 3. Configuration

Hooks live in `settings.json`:

| File | Scope |
|------|-------|
| `~/.claude/settings.json` | All your projects |
| `.claude/settings.json` | One project, shared via git |
| `.claude/settings.local.json` | One project, personal, not committed |

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          { "type": "command", "command": "npx prettier --write \"$CLAUDE_FILE_PATHS\"" }
        ]
      }
    ]
  }
}
```

- `matcher` filters by tool name (regex-style, e.g. `Bash`, `Edit|Write`). Omit it, or use `*`, to match all.
- `command` is any shell command. It receives event data as JSON on stdin.
- Use `/hooks` in Claude Code to view and edit hooks.

## 4. Controlling Behavior with Exit Codes

| Exit code | Effect |
|-----------|--------|
| `0` | Success; stdout is shown (or added as context for some events) |
| `2` | Blocking error; stderr is fed back to Claude. On `PreToolUse` the tool call is blocked |
| Other | Non-blocking error; shown to the user, execution continues |

Hooks can also print structured JSON (for example a permission decision of allow, deny or ask) for finer control.

Example: block `rm -rf` in a `PreToolUse` hook on `Bash`. The script reads the JSON from stdin, checks `tool_input.command`, and exits with code `2` plus an explanation on stderr.

## 5. Hooks vs Other Features

| Feature | Nature | Use for |
|---------|--------|---------|
| `CLAUDE.md` | Guidance Claude may follow | Conventions, commands |
| Permissions (`allow`/`deny`) | Static rules for tools | Simple allow and deny lists |
| Hooks | Deterministic code at lifecycle events | Enforcement, automation, custom logic |
| MCP | Adds new tools | Access to external systems |

## 6. Security and Best Practices

- Hooks run with your user permissions and can run any command. Review hooks from others before enabling them.
- Quote shell variables and validate the stdin input; treat it as untrusted.
- Keep hooks fast, since they run inline and slow ones delay the session.
- Prefer `PreToolUse` for prevention and `PostToolUse` for cleanup.
- Commit project hooks in `.claude/settings.json` so the team shares them.

## Key Takeaways

- Hooks are deterministic, harness-enforced commands tied to lifecycle events.
- `PreToolUse` can block a tool call; `PostToolUse` runs after it, for example to format.
- Exit code `2` is a blocking error and its stderr is sent back to Claude.
- `matcher` selects which tools a hook applies to.
- Hooks are configured in `settings.json` at user, project or local scope.
- Use hooks for rules that must always happen; use `CLAUDE.md` for guidance.

## Practice Questions

**1. Why use a hook instead of a `CLAUDE.md` instruction to run a formatter after every edit?**
Hooks are executed by the harness and always run, while `CLAUDE.md` instructions are only guidance Claude might skip.

**2. Which event and exit code would you use to prevent Claude from running a dangerous Bash command?**
A `PreToolUse` hook with matcher `Bash` that exits with code `2`. Its stderr is returned to Claude.

**3. What does the `matcher` field do?**
It filters which tool names trigger the hook, such as `Edit|Write`. Omitting it matches all tools.

**4. Where would you put a hook that every teammate should get?**
In the project's `.claude/settings.json`, committed to git. Personal-only hooks go in `.claude/settings.local.json`.

**5. What is the main security concern with hooks?**
They execute arbitrary shell commands with your user permissions, so untrusted hooks or unvalidated input can cause harm.
