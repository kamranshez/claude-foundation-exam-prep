# Claude Code Hooks

Hooks are user-defined commands that Claude Code runs automatically at specific points in its lifecycle, giving you deterministic control instead of relying on the model to remember an instruction.

## 1. Why It Matters

A line in `CLAUDE.md` is guidance that Claude may or may not follow. A hook is code that the harness always runs. Use hooks when something must happen every time: formatting, blocking dangerous commands, logging, notifications. See also [MCP](mcp.md) and [Tool Use](tool-use.md), since hooks commonly match on tool calls.

## 2. Common Hook Events

| Event | Fires when | Typical use |
|-------|-----------|-------------|
| `SessionStart` | A session begins or resumes | Load context, set env vars |
| `UserPromptSubmit` | User submits a prompt, before Claude processes it | Validate or add context |
| `PreToolUse` | Before a tool call runs | Block or approve calls |
| `PostToolUse` | After a tool call succeeds | Auto-format, run linters or tests |
| `Notification` | Claude needs attention (e.g. permission prompt) | Desktop or chat alert |
| `Stop` | Claude finishes responding | Final checks, force continuation |
| `SubagentStop` | A subagent finishes | Check subagent output |
| `PreCompact` | Before context compaction | Back up the transcript |
| `SessionEnd` | Session ends | Cleanup, logging |

## 3. Configuration

Hooks live in a settings file under a `hooks` key:

| File | Scope |
|------|-------|
| `~/.claude/settings.json` | All your projects |
| `.claude/settings.json` | This project, shared via version control |
| `.claude/settings.local.json` | This project, just you (not committed) |

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

- `matcher` filters by tool name (for tool events). It can be an exact name, a regex such as `Edit|Write`, or `*`. MCP tools are matched as `mcp__<server>__<tool>`.
- Events with no tool (e.g. `Stop`) take no matcher.
- `/hooks` inside a session lets you view and edit hooks.

## 4. Input and Output

The hook command receives JSON on **stdin** (fields such as `session_id`, `hook_event_name`, `tool_name`, `tool_input`). It communicates back through its exit code:

| Exit code | Meaning |
|-----------|---------|
| `0` | Success; stdout is processed (for some events it is added to context) |
| `2` | Blocking error; stderr is fed back to Claude, and for `PreToolUse` the tool call is blocked |
| Other | Non-blocking error; shown to the user, execution continues |

A hook can instead print structured JSON on stdout for finer control (for example a `PreToolUse` permission decision of allow, deny or ask).

### Example: block edits to `.env`

```bash
#!/bin/bash
# PreToolUse hook: reads JSON from stdin
path=$(jq -r '.tool_input.file_path // empty')
if [[ "$path" == *.env ]]; then
  echo "Editing .env files is not allowed." >&2
  exit 2
fi
exit 0
```

## 5. Hooks vs Other Features

| Need | Best fit |
|------|----------|
| Guaranteed, deterministic action | Hook |
| Style or convention guidance | `CLAUDE.md` |
| Allow or deny specific tools | Permission rules in settings |
| Reusable instructions loaded on demand | Skills or slash commands |

## 6. Security Considerations

- Hooks run with **your user's permissions** and can execute arbitrary commands, so review any hook before adding it, especially from a shared repo.
- Validate and quote inputs from stdin; never interpolate them unsafely into shell commands.
- Use absolute paths or `$CLAUDE_PROJECT_DIR` so hooks do not depend on the working directory.
- Keep hooks fast; they run inline and slow hooks delay the session.

## Key Takeaways

- Hooks are shell commands triggered by lifecycle events, so they are deterministic, unlike prompt instructions.
- Key events: `PreToolUse`, `PostToolUse`, `UserPromptSubmit`, `Stop`, `Notification`, `SessionStart`.
- Configure them in `settings.json` (user, project or local) with an optional tool `matcher`.
- Hooks get JSON on stdin; exit code `2` blocks and returns stderr to Claude.
- Treat hooks as trusted code: they run with your permissions.

## Practice Questions

1. **Why use a hook instead of a `CLAUDE.md` instruction to run a formatter after every edit?**
   A hook always runs, while a `CLAUDE.md` instruction is guidance the model might skip.

2. **Which event lets you stop a tool call before it executes?**
   `PreToolUse`; exiting with code `2` (or returning a deny decision) blocks the call.

3. **How does a hook receive information about the tool call?**
   Claude Code passes JSON on stdin, including `tool_name` and `tool_input`.

4. **How would you make a hook apply only to file edits?**
   Set the `matcher` to `Edit|Write` (plus any other editing tools) on a `PreToolUse` or `PostToolUse` entry.

5. **Where should you put a hook that your whole team should share?**
   In `.claude/settings.json`, committed to the repository. Use `.claude/settings.local.json` for personal hooks.
