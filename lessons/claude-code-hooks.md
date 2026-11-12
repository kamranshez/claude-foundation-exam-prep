# Claude Code Hooks

Hooks are user-defined commands that Claude Code runs automatically at specific points in its lifecycle, giving deterministic control instead of relying on the model to remember an instruction.

## 1. Why It Matters

Instructions in prompts or `CLAUDE.md` are advisory; the model may skip them. A hook is executed by the harness, so it happens every time. Use hooks to enforce formatting, block dangerous commands, log activity or send notifications. Hooks complement [MCP](mcp.md) tools and permission rules.

## 2. Common Hook Events

| Event | Fires | Typical use |
|-------|-------|-------------|
| `PreToolUse` | Before a tool call runs | Block or validate a command |
| `PostToolUse` | After a tool call succeeds | Run a formatter or linter |
| `UserPromptSubmit` | When the user submits a prompt | Add context or validate input |
| `Notification` | When Claude needs attention | Desktop or chat alert |
| `Stop` | When Claude finishes responding | Run tests, final checks |
| `SessionStart` | When a session begins or resumes | Load project context |

## 3. Configuration

Hooks live in `settings.json`: `~/.claude/settings.json` (user), `.claude/settings.json` (project, shared) or `.claude/settings.local.json` (project, personal). A `matcher` selects which tools the hook applies to.

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          { "type": "command", "command": "npx prettier --write \"$CLAUDE_PROJECT_DIR\"" }
        ]
      }
    ]
  }
}
```

`/hooks` inside a session lets you view and manage configured hooks.

## 4. Input and Output

- The hook receives JSON on stdin (session id, event name, tool name and tool input).
- Exit code `0`: success; the action proceeds.
- Exit code `2`: blocking error; for `PreToolUse` the tool call is blocked and stderr is fed back to Claude.
- Other non-zero codes: non-blocking error shown to the user.
- A hook can also print structured JSON to control decisions (for example allow, deny or add context).

Example `PreToolUse` guard (bash script, blocks `rm -rf`):

```bash
#!/bin/bash
cmd=$(jq -r '.tool_input.command')
if echo "$cmd" | grep -q 'rm -rf'; then
  echo "Blocked: rm -rf is not allowed" >&2
  exit 2
fi
```

## 5. Hooks vs Other Mechanisms

| Mechanism | Nature |
|-----------|--------|
| `CLAUDE.md` | Advisory guidance the model may follow |
| Permission rules | Allow/ask/deny which tools may run |
| Hooks | Deterministic code run at lifecycle events |

## 6. Security Considerations

- Hooks run shell commands with your user's permissions; review them before adding.
- Never pipe untrusted input into shell commands unquoted.
- Project hooks in a shared repo run for everyone who uses it, so review changes to them like code.

## Key Takeaways

- Hooks give deterministic, harness-enforced behaviour at lifecycle events.
- `PreToolUse` can block; `PostToolUse` suits formatting and linting.
- Configured in `settings.json` with an event, a `matcher` and a command.
- Exit code 2 blocks and returns stderr to Claude; 0 means success.
- Hooks execute arbitrary commands, so treat them as security-sensitive.

## Practice Questions

1. **Why use a hook rather than a `CLAUDE.md` instruction to run a formatter after every edit?**
   Hooks are executed by the harness every time, whereas `CLAUDE.md` is advisory and may be ignored.

2. **Which event would you use to prevent Claude from running a dangerous shell command?**
   `PreToolUse`, with a script that exits with code 2 to block the call.

3. **What does exit code 2 from a `PreToolUse` hook do?**
   It blocks the tool call and sends the hook's stderr back to Claude as feedback.

4. **How do you restrict a hook to file-editing tools only?**
   Set the `matcher` to something like `Edit|Write`.

5. **Where can you share hooks with your team?**
   In the project's `.claude/settings.json`, committed to the repository.
