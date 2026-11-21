# Claude Code Hooks

Hooks are user-defined commands that Claude Code runs automatically at specific points in its lifecycle, giving deterministic control instead of relying on the model to remember an instruction.

## 1. Why Hooks Matter

A prompt or CLAUDE.md instruction is a request; a hook is guaranteed. Use hooks when something must always happen (formatting, blocking dangerous commands, logging). See also [MCP](mcp.md) and [Tool Use](tool-use.md), since hooks commonly match on tool calls.

## 2. Common Hook Events

| Event | When it fires | Typical use |
|-------|---------------|-------------|
| `PreToolUse` | Before a tool runs | Block or validate commands |
| `PostToolUse` | After a tool succeeds | Run formatter or linter |
| `UserPromptSubmit` | When the user submits a prompt | Add context, validate input |
| `Notification` | When Claude needs attention | Desktop alerts |
| `Stop` | When Claude finishes responding | Run tests, final checks |
| `SubagentStop` | When a subagent finishes | Check subagent output |
| `SessionStart` | Session begins or resumes | Load project context |

## 3. Configuration

Hooks live in settings files: `~/.claude/settings.json` (user), `.claude/settings.json` (project, shareable) or `.claude/settings.local.json` (personal, not committed). Matchers select which tools trigger the hook.

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

MCP tools can be matched by name, e.g. `mcp__github__.*`. The `/hooks` command lets you view and manage configured hooks.

## 4. Input and Output

- Hooks receive JSON on stdin (session id, event name, tool name and tool input).
- **Exit code 0**: success; stdout may be added as context for some events.
- **Exit code 2**: blocking error; stderr is fed back to Claude. For `PreToolUse` this blocks the tool call.
- **Other exit codes**: non-blocking error; execution continues.
- Hooks can also print structured JSON to control behaviour (e.g. allow, deny or ask for permission).

```bash
#!/bin/bash
# PreToolUse hook: block rm -rf
cmd=$(jq -r '.tool_input.command // ""')
if echo "$cmd" | grep -q 'rm -rf'; then
  echo "Blocked: rm -rf is not allowed" >&2
  exit 2
fi
```

## 5. Security Considerations

- Hooks run with your user's permissions and execute arbitrary shell commands; review any hook before adding it.
- Quote variables, validate input, and avoid touching sensitive files.
- Use absolute paths or `$CLAUDE_PROJECT_DIR` for scripts.
- Hooks are not a substitute for permission rules; use both.

## Key Takeaways

- Hooks give deterministic, automatic control over Claude Code's lifecycle.
- Key events: `PreToolUse`, `PostToolUse`, `UserPromptSubmit`, `Stop`, `Notification`, `SessionStart`.
- Configured in settings JSON files with a matcher and a command.
- Exit code 2 blocks (for `PreToolUse`) and sends stderr back to Claude; 0 is success.
- Hooks run arbitrary commands, so treat them as trusted code.

## Practice Questions

1. **Why use a hook instead of a CLAUDE.md instruction?**
   A hook runs deterministically every time; an instruction is advisory and the model may not follow it.

2. **Which event would you use to auto-format files after Claude edits them?**
   `PostToolUse` with a matcher such as `Edit|Write`.

3. **How can a hook prevent a tool call from happening?**
   Use a `PreToolUse` hook that exits with code 2 (or returns a deny decision in JSON); stderr is shown to Claude.

4. **Where would you put hooks that should be shared with the team?**
   In `.claude/settings.json`, committed to the repo; use `.claude/settings.local.json` for personal hooks.

5. **What is a security risk of hooks?**
   They run arbitrary shell commands with your permissions, so a malicious or careless hook can do harm.
