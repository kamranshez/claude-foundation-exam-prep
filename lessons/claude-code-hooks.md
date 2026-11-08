# Claude Code Hooks

Hooks are user-defined commands that Claude Code runs automatically at specific points in its lifecycle, giving deterministic control instead of relying on the model to remember an instruction.

## 1. Why It Matters

Instructions in prompts or `CLAUDE.md` are advisory: Claude may forget or ignore them. Hooks are executed by the harness, so they always run. Use them to enforce formatting, block dangerous commands, log activity or inject context.

## 2. Where Hooks Are Configured

Hooks live in settings files under a `hooks` key:

| File | Scope |
|------|-------|
| `~/.claude/settings.json` | All your projects (user) |
| `.claude/settings.json` | One project, shared via git |
| `.claude/settings.local.json` | One project, personal, not committed |

## 3. Common Hook Events

| Event | When it fires | Typical use |
|-------|---------------|-------------|
| `PreToolUse` | Before a tool call runs | Block or validate a command |
| `PostToolUse` | After a tool call succeeds | Run a formatter or linter |
| `UserPromptSubmit` | When the user submits a prompt | Add context, validate input |
| `Notification` | When Claude needs attention | Desktop alert |
| `Stop` | When Claude finishes responding | Run tests, final checks |
| `SubagentStop` | When a subagent finishes | Verify subagent output |
| `SessionStart` | Session begins or resumes | Load project context |

## 4. Example Configuration

Run Prettier after every file edit:

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

- `matcher` is matched against the tool name (regex-style, e.g. `Bash`, `Edit|Write`). Omit it or use `*` to match all.
- The hook receives event data as JSON on stdin (tool name, tool input, session info).

## 5. Controlling Behaviour with Exit Codes

| Exit code | Effect |
|-----------|--------|
| `0` | Success; continue normally |
| `2` | Blocking error; stderr is fed back to Claude (for `PreToolUse` the tool call is blocked) |
| Other non-zero | Non-blocking error; shown to the user, execution continues |

Hooks can also print structured JSON to stdout for finer control, such as allowing, denying or asking for permission on a tool call.

Example guard that blocks `rm -rf` (reads JSON from stdin):

```bash
#!/bin/bash
cmd=$(jq -r '.tool_input.command')
if echo "$cmd" | grep -q 'rm -rf'; then
  echo "Blocked: rm -rf is not allowed" >&2
  exit 2
fi
exit 0
```

## 6. Hooks vs Other Features

| Feature | Nature | Best for |
|---------|--------|----------|
| `CLAUDE.md` | Advisory context | Conventions, project knowledge |
| Hooks | Deterministic, harness-run | Rules that must always happen |
| Permissions | Allow/deny rules for tools | Controlling what Claude may do |
| MCP | External tools and data | Extending capabilities |

## 7. Security Notes

- Hooks run shell commands with your user's permissions, so review them before enabling.
- Never trust hook input blindly: quote variables and validate paths.
- Project hooks come from the repo, so treat an untrusted repo's `.claude/settings.json` with caution.

## Key Takeaways

- Hooks are deterministic automation run by the harness, not by the model.
- Key events: `PreToolUse`, `PostToolUse`, `UserPromptSubmit`, `Stop`, `Notification`, `SessionStart`.
- A `matcher` selects which tools trigger a hook; data arrives as JSON on stdin.
- Exit code `2` blocks (and sends stderr to Claude); `0` passes.
- Configure in user, project or local settings files.
- Use hooks for "must always happen" rules; use `CLAUDE.md` for guidance.

## Practice Questions

**1. Why use a hook instead of writing "always run the formatter" in `CLAUDE.md`?**
Hooks are executed by the harness and always run; `CLAUDE.md` is advisory and Claude may skip it.

**2. Which event and exit code would you use to prevent a dangerous Bash command from running?**
A `PreToolUse` hook with matcher `Bash` that exits with code `2`. Stderr is returned to Claude.

**3. How does a hook learn which tool is being called and with what input?**
The harness passes event data as JSON on stdin, including the tool name and `tool_input`.

**4. Where would you put a hook that only you want in one project, not committed to git?**
In `.claude/settings.local.json`.

**5. A hook exits with code 1. What happens?**
It is a non-blocking error: the user sees it, but execution continues. Only exit code `2` blocks.
