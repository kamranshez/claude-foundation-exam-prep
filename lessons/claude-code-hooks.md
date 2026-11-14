# Claude Code Hooks

Hooks are user-defined shell commands that Claude Code runs automatically at set points in its lifecycle, giving deterministic control instead of relying on the model to remember an instruction.

## 1. Why It Matters

A line in a prompt or `CLAUDE.md` is a request, and Claude may not follow it every time. A hook is code that the harness always runs. Use hooks for things that must happen every time: formatting, blocking dangerous commands, logging, notifications. See [tool use](tool-use.md) for the tool calls that hooks intercept.

## 2. Common Hook Events

| Event | Fires | Typical use |
|-------|-------|-------------|
| `PreToolUse` | Before a tool call runs | Block or approve a call, validate input |
| `PostToolUse` | After a tool call succeeds | Run a formatter or linter, log changes |
| `UserPromptSubmit` | When the user submits a prompt | Add context, validate prompts |
| `Notification` | When Claude Code sends a notification | Desktop or chat alerts |
| `Stop` | When Claude finishes responding | Run checks, force it to continue |
| `SubagentStop` | When a subagent finishes | Same as `Stop`, for subagents |
| `SessionStart` | When a session begins or resumes | Load project context |
| `PreCompact` | Before context is compacted | Save or adjust state |

## 3. Configuration

Hooks live in a settings file under the `hooks` key. `matcher` selects tools by name (it applies to tool events such as `PreToolUse` and `PostToolUse`).

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

| File | Scope |
|------|-------|
| `~/.claude/settings.json` | All your projects (user) |
| `.claude/settings.json` | One project, shared via git |
| `.claude/settings.local.json` | One project, personal, not committed |

The `/hooks` command shows the configured hooks.

## 4. Input and Control

Claude Code passes event data to the hook as JSON on **stdin** (for example `tool_name` and `tool_input`). The hook controls what happens next through its **exit code**:

| Exit code | Meaning |
|-----------|---------|
| `0` | Success; stdout may be shown or used as context for some events |
| `2` | Blocking error; stderr is fed back to Claude, and for `PreToolUse` the tool call is blocked |
| Other | Non-blocking error; shown to the user, execution continues |

A hook can instead print structured JSON to stdout for finer control, such as allowing, denying or asking about a tool call.

Example `PreToolUse` guard (blocks `rm -rf`):

```bash
#!/bin/bash
cmd=$(jq -r '.tool_input.command // empty')
if echo "$cmd" | grep -q 'rm -rf'; then
  echo "Blocked: rm -rf is not allowed" >&2
  exit 2
fi
exit 0
```

## 5. Hooks vs Other Features

| Need | Use |
|------|-----|
| Guaranteed action on an event | Hook |
| Guidance Claude should usually follow | `CLAUDE.md` / prompt |
| Reusable instructions loaded when relevant | Skill |
| External tools and data | [MCP](mcp.md) |

## 6. Security

Hooks run shell commands with your user's permissions. Review hooks from any source before enabling them, quote variables, and avoid touching sensitive files.

## Key Takeaways

- Hooks give deterministic, always-run behavior; prompts and `CLAUDE.md` do not.
- `PreToolUse` runs before a tool and can block it; `PostToolUse` runs after it.
- Hooks receive JSON on stdin; exit code `2` blocks and sends stderr to Claude.
- Configure them under `hooks` in `settings.json`, with a `matcher` to target tools.
- Project, user and local settings files set the scope; hooks run with your permissions, so review them.

## Practice Questions

**1. Why use a hook instead of writing "always run the formatter" in `CLAUDE.md`?**
A hook is executed by the harness every time, while `CLAUDE.md` is guidance the model may not follow.

**2. Which event lets you stop a dangerous Bash command before it runs?**
`PreToolUse`, with a hook that exits with code `2` (or returns a deny decision).

**3. How does a hook receive information about the tool call?**
As JSON on stdin, including fields such as `tool_name` and `tool_input`.

**4. What happens when a `PreToolUse` hook exits with code 2?**
The tool call is blocked and the hook's stderr is passed back to Claude so it can adjust.

**5. Where do you put a hook that only applies to you in one project?**
`.claude/settings.local.json`, which is not committed to git.
