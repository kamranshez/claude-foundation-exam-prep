# Claude Code Hooks

Hooks are user-defined commands that Claude Code runs automatically at specific points in its lifecycle, giving deterministic control instead of relying on the model to remember.

## 1. Why It Matters

Instructions in prompts or `CLAUDE.md` are advisory: Claude may forget them. Hooks are executed by the harness, so they always run. Use them to enforce formatting, block dangerous commands, log activity or inject context. They pair well with [MCP](mcp.md) and [tool use](tool-use.md), since many hooks match on tool calls.

## 2. Common Hook Events

| Event | When it fires | Typical use |
|-------|---------------|-------------|
| `PreToolUse` | Before a tool call runs | Block or approve a call, validate input |
| `PostToolUse` | After a tool call succeeds | Run a formatter or linter, log results |
| `UserPromptSubmit` | When the user submits a prompt | Add context, validate prompts |
| `Stop` | When Claude finishes responding | Run tests, require follow-up work |
| `SessionStart` | When a session begins or resumes | Load project context |
| `Notification` | When Claude sends a notification | Desktop or chat alerts |

## 3. Configuration

Hooks live in `settings.json` (user `~/.claude/settings.json`, project `.claude/settings.json`, or the local, uncommitted `.claude/settings.local.json`). A `matcher` selects which tools the hook applies to.

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          { "type": "command", "command": "npx prettier --write \"$CLAUDE_FILE\"" }
        ]
      }
    ]
  }
}
```

The hook command receives event details as JSON on stdin (for example `tool_name` and `tool_input`).

## 4. Controlling Behavior with Exit Codes

| Exit code | Meaning |
|-----------|---------|
| `0` | Success; continue normally |
| `2` | Blocking error; stderr is fed back to Claude (for `PreToolUse`, the tool call is blocked) |
| Other | Non-blocking error; shown to the user, execution continues |

Hooks can also print structured JSON to stdout for finer control, such as allow/deny decisions.

## 5. Hooks vs. Other Mechanisms

| Mechanism | Guarantee | Best for |
|-----------|-----------|----------|
| `CLAUDE.md` | Advisory | Conventions and context |
| Permissions | Rule-based allow/deny | Controlling which tools may run |
| Hooks | Always executed | Automation and enforcement |

## 6. Best Practices

- Keep hooks fast; they run synchronously in the loop.
- Treat hook input as untrusted and quote shell variables.
- Hooks run with your user permissions, so review any hook before adopting it.
- Test hooks by running the command manually with sample JSON.

## Key Takeaways

- Hooks are deterministic: the harness runs them, not the model.
- `PreToolUse` can block actions; `PostToolUse` reacts after them.
- Exit code `2` is the blocking signal, and its stderr goes to Claude.
- Configure them in `settings.json` with an event, `matcher` and command.
- Use hooks for guarantees, `CLAUDE.md` for guidance.

## Practice Questions

**1. Why use a hook instead of a `CLAUDE.md` instruction to run a formatter after every edit?**
Hooks always execute; `CLAUDE.md` is advisory and Claude might skip it.

**2. Which event lets you stop a tool call before it runs?**
`PreToolUse`, by exiting with code `2` (or returning a deny decision).

**3. What does a hook's `matcher` do?**
It filters which tools (for example `Edit|Write` or `Bash`) trigger the hook.

**4. A hook exits with code 1. What happens?**
It is a non-blocking error: the user sees it, but Claude's execution continues.

**5. How does a hook receive information about the tool call?**
As JSON on stdin, including fields such as `tool_name` and `tool_input`.
