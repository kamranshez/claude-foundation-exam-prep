# Claude Code Hooks

Hooks are user-defined shell commands that Claude Code runs automatically at set points in its lifecycle, giving deterministic control instead of relying on the model to remember.

## 1. Why It Matters

Instructions in prompts or `CLAUDE.md` are advisory: Claude may forget or ignore them. Hooks are executed by the harness, so they always run. Use them to enforce formatting, block dangerous commands, log activity or inject context.

## 2. Common Hook Events

| Event | When it fires | Typical use |
|-------|---------------|-------------|
| `PreToolUse` | Before a tool call runs | Block or approve a call, validate input |
| `PostToolUse` | After a tool call succeeds | Run a formatter or linter, log changes |
| `UserPromptSubmit` | When the user submits a prompt | Add context, validate prompts |
| `Stop` | When Claude finishes responding | Run tests, notify, force continuation |
| `SubagentStop` | When a subagent finishes | Post-process subagent output |
| `SessionStart` | When a session starts or resumes | Load project context or env setup |
| `Notification` | When Claude needs attention or permission | Desktop or chat alerts |

## 3. Configuration

Hooks live in `settings.json` files:

- `~/.claude/settings.json`: user-wide
- `.claude/settings.json`: project, shared via git
- `.claude/settings.local.json`: project, personal, not committed

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

The `matcher` filters by tool name (regex-style, e.g. `Bash` or `Edit|Write`). Events without tools ignore it.

## 4. How Hooks Communicate

The hook receives event data as JSON on **stdin** (session ID, tool name, tool input, etc.). It responds through its exit code and output:

| Exit code | Meaning |
|-----------|---------|
| `0` | Success; stdout may be shown or parsed as JSON |
| `2` | Blocking error; stderr is fed back to Claude (for `PreToolUse` the call is blocked) |
| Other | Non-blocking error; shown to the user, execution continues |

Hooks can also print JSON to stdout for finer control, such as allowing, denying or asking for permission, or stopping the turn.

## 5. Example: Block Dangerous Commands

```bash
#!/bin/bash
# PreToolUse hook for the Bash tool
cmd=$(jq -r '.tool_input.command')
if echo "$cmd" | grep -q "rm -rf /"; then
  echo "Blocked: destructive command" >&2
  exit 2
fi
exit 0
```

## 6. Hooks vs Related Features

| Feature | Nature | Use for |
|---------|--------|---------|
| `CLAUDE.md` | Advisory context | Conventions and guidance |
| Permissions rules | Allow/deny lists | Which tools may run |
| Hooks | Deterministic code | Guaranteed side effects and checks |
| Skills | On-demand instructions | Reusable task playbooks |

## 7. Best Practices and Safety

- Keep hooks fast; they run synchronously and slow the loop.
- Hooks run with your user permissions: review them, quote variables and never run untrusted hook code.
- Validate and sanitize the JSON input; treat it as untrusted.
- Use exit code 2 plus a clear stderr message so Claude can adjust.
- Prefer project hooks in version control so the team shares them.

## Key Takeaways

- Hooks are deterministic commands triggered by lifecycle events; prompts are only advisory.
- `PreToolUse` can block; `PostToolUse` is for follow-up actions like formatting.
- Input arrives as JSON on stdin; exit code `2` blocks and sends stderr to Claude.
- Configure in `settings.json` (user, project or local) with a `matcher` for tool names.
- Hooks run with your privileges, so write and review them carefully.

## Practice Questions

**1. Why use a hook instead of a `CLAUDE.md` instruction to run a formatter after every edit?**
Hooks are executed by the harness every time, while `CLAUDE.md` is guidance Claude may not follow.

**2. Which event lets you stop a tool call before it runs?**
`PreToolUse`, by exiting with code 2 (or returning a deny decision in JSON).

**3. How does a hook receive information about the tool call?**
As JSON on standard input.

**4. What happens when a hook exits with code 2?**
It is a blocking error: stderr is returned to Claude, and for `PreToolUse` the tool call is blocked.

**5. Where would you put a hook that only you want, for one project, not shared with the team?**
`.claude/settings.local.json`.
