# Claude Code Hooks

Hooks are user-defined commands that Claude Code runs automatically at specific points in its lifecycle, giving deterministic control instead of relying on the model to remember an instruction.

## 1. Why It Matters

Instructions in prompts or `CLAUDE.md` are advisory: Claude may forget or skip them. Hooks are enforced by the harness, so they are the right tool for "always do X" or "never do Y" rules (formatting, blocking dangerous commands, logging, notifications).

## 2. Common Hook Events

| Event | Fires | Typical use |
|-------|-------|-------------|
| `PreToolUse` | Before a tool call runs | Block or modify risky calls (e.g. `rm -rf`, edits to protected files) |
| `PostToolUse` | After a tool call succeeds | Run formatter, linter or tests after edits |
| `UserPromptSubmit` | When the user submits a prompt | Add context or validate the prompt |
| `Notification` | When Claude needs attention | Desktop or Slack alerts |
| `Stop` | When Claude finishes responding | Final checks, force continuation |
| `SubagentStop` | When a subagent finishes | Validate subagent output |
| `SessionStart` | When a session begins or resumes | Load project context |

## 3. Configuration

Hooks live in `settings.json` (user: `~/.claude/settings.json`, project: `.claude/settings.json`, local: `.claude/settings.local.json`).

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

- `matcher` selects which tools the hook applies to (tool name, regex such as `Edit|Write`, or `*` for all). Events without tools ignore it.
- `type: "command"` runs a shell command.
- The hook receives event data as JSON on **stdin** (tool name, tool input, session info).

## 4. Controlling Behavior with Exit Codes

| Exit code | Meaning |
|-----------|---------|
| `0` | Success; stdout is shown to the user (for some events it is added to Claude's context) |
| `2` | Blocking error; stderr is fed back to Claude. For `PreToolUse` the tool call is blocked |
| Other | Non-blocking error; shown to the user, execution continues |

Hooks can also print structured JSON to stdout (for example a `permissionDecision` of `allow`, `deny` or `ask` in `PreToolUse`) for finer control.

## 5. Example: Block Edits to `.env`

```bash
#!/bin/bash
# PreToolUse hook: read JSON from stdin
path=$(jq -r '.tool_input.file_path // empty')
if [[ "$path" == *.env ]]; then
  echo "Editing .env files is not allowed." >&2
  exit 2
fi
exit 0
```

Because the script exits with `2`, the edit is blocked and Claude sees the message and can adjust.

## 6. Hooks vs Other Features

| Feature | Nature | Best for |
|---------|--------|----------|
| `CLAUDE.md` | Advisory context | Conventions, project knowledge |
| Permissions | Allow/deny rules for tools | Coarse access control |
| Hooks | Deterministic code at lifecycle events | Enforcement, automation |
| Skills | On-demand instructions | Reusable task playbooks |

## 7. Best Practices and Safety

- Keep hooks fast; they run synchronously and slow the loop.
- Hooks run with your user permissions, so review them like any script. Never run untrusted hook configs.
- Quote variables and validate input to avoid shell injection.
- Use absolute paths or `$CLAUDE_PROJECT_DIR` for scripts.
- Prefer `PreToolUse` for prevention and `PostToolUse` for cleanup; a post hook cannot undo a tool call.

## Key Takeaways

- Hooks give deterministic, enforced behavior; `CLAUDE.md` is only advisory.
- Configure them in `settings.json` by event, with an optional tool `matcher`.
- Hooks get JSON on stdin; exit code `2` blocks (for `PreToolUse`) and returns stderr to Claude.
- `PreToolUse` prevents, `PostToolUse` reacts, `Stop` and `Notification` handle completion and alerts.
- Hooks execute arbitrary commands with your permissions, so treat them as security-sensitive.

## Practice Questions

1. **Why use a hook instead of a `CLAUDE.md` instruction to run a formatter after every edit?**
   Hooks are executed by the harness and always run; `CLAUDE.md` is advisory and Claude might skip it.

2. **Which event and exit code block a tool call before it runs?**
   `PreToolUse` with exit code `2` (stderr is returned to Claude).

3. **How does a hook receive information about the tool call?**
   As JSON on stdin, including the tool name and tool input.

4. **A `PostToolUse` hook exits with code 1. What happens?**
   It is a non-blocking error: the message is shown to the user and execution continues. The tool call has already happened.

5. **Name a security concern with hooks.**
   They run arbitrary shell commands with your permissions, so untrusted or poorly quoted hooks can cause harm or injection.
