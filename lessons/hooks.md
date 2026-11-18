# Claude Code Hooks

Hooks are user-defined commands that Claude Code runs automatically at specific points in its lifecycle, giving deterministic control instead of relying on the model to remember an instruction.

## 1. Why Hooks Matter

Instructions in prompts or `CLAUDE.md` are advisory: the model may forget or skip them. A hook is executed by the harness, so it always runs. Use hooks to enforce rules (formatting, blocking dangerous commands, logging, notifications). Hooks complement [tool use](tool-use.md) and [MCP](mcp.md) tools, since they can match both built-in and `mcp__<server>__<tool>` tools.

## 2. Common Hook Events

| Event | Fires when | Typical use |
|-------|-----------|-------------|
| `PreToolUse` | Before a tool call runs | Block or approve a call, validate input |
| `PostToolUse` | After a tool call succeeds | Run a formatter or linter, log activity |
| `UserPromptSubmit` | User submits a prompt | Add context, validate prompts |
| `Notification` | Claude needs attention | Desktop or chat alert |
| `Stop` | Claude finishes responding | Run tests, force continuation |
| `SubagentStop` | A subagent finishes | Post-process subagent work |
| `SessionStart` | A session begins or resumes | Load project context |

## 3. Configuration

Hooks live in `settings.json` files:

| File | Scope |
|------|-------|
| `~/.claude/settings.json` | All your projects |
| `.claude/settings.json` | Project, shareable via version control |
| `.claude/settings.local.json` | Project, just you (not committed) |

Example: run Prettier after every file edit.

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

The `matcher` selects tools by name (regex-style, e.g. `Edit|Write`, or `mcp__github__.*`). Use `/hooks` inside a session to view and manage them.

## 4. Input and Output

- The hook receives JSON on stdin describing the event (session ID, tool name, tool input, and so on).
- The exit code controls behaviour:

| Exit code | Effect |
|-----------|--------|
| `0` | Success; stdout may be shown or added as context |
| `2` | Blocking error; for `PreToolUse` the tool call is blocked and stderr is fed back to Claude |
| Other | Non-blocking error; shown to the user, execution continues |

- A hook can instead print structured JSON to stdout for finer control (for example allow, deny or ask on a `PreToolUse` decision).

Example: a `PreToolUse` hook script that blocks `rm -rf`.

```bash
#!/bin/bash
cmd=$(jq -r '.tool_input.command // empty')
if echo "$cmd" | grep -q 'rm -rf'; then
  echo "Blocked: rm -rf is not allowed" >&2
  exit 2
fi
```

## 5. Security Considerations

- Hooks run shell commands with your user's permissions; review any hook before adding it, especially from shared project settings.
- Quote variables and validate inputs to avoid injection.
- Prefer absolute paths for scripts and keep hooks fast, since they run inline.

## Key Takeaways

- Hooks give deterministic, harness-enforced automation, unlike advisory prompts.
- Key events: `PreToolUse`, `PostToolUse`, `UserPromptSubmit`, `Notification`, `Stop`, `SessionStart`.
- Configure them in `settings.json` with an event, a `matcher` and a command.
- Exit code `2` blocks (for `PreToolUse`) and returns stderr to Claude; `0` means success.
- Hooks execute arbitrary commands, so treat them as trusted code.

## Practice Questions

1. **Why use a hook rather than a `CLAUDE.md` instruction to run a formatter?**
   Hooks are executed by the harness every time, while `CLAUDE.md` guidance is advisory and may be ignored.

2. **Which event lets you block a tool call before it runs?**
   `PreToolUse`; exiting with code 2 (or returning a deny decision) blocks the call.

3. **What does a hook's `matcher` do?**
   It filters which tools trigger the hook, for example `Edit|Write` or an MCP tool pattern like `mcp__github__.*`.

4. **Where would you put a hook that should apply to your whole team?**
   In `.claude/settings.json`, committed to the repository.

5. **What is a security risk of hooks?**
   They run arbitrary shell commands with your permissions, so untrusted or poorly quoted hooks can be dangerous.
