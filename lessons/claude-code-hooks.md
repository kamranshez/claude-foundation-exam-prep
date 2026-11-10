# Claude Code Hooks

Hooks are user-defined commands that Claude Code runs automatically at set points in its lifecycle, giving deterministic control instead of relying on the model to remember an instruction.

## 1. Why It Matters

A line in `CLAUDE.md` is a request that Claude may or may not follow. A hook is run by the harness itself, so it happens every time. Use hooks for things that must always happen: formatting, blocking dangerous commands, logging, notifications.

## 2. Common Hook Events

| Event | Fires | Typical use |
|-------|-------|-------------|
| `PreToolUse` | Before a tool call runs | Block or approve a call, validate arguments |
| `PostToolUse` | After a tool call succeeds | Run a formatter or linter on edited files |
| `UserPromptSubmit` | When the user submits a prompt | Add context or validate the prompt |
| `Notification` | When Claude needs attention | Desktop or chat alert |
| `Stop` | When Claude finishes responding | Run tests, force further work |
| `SubagentStop` | When a subagent finishes | Same as Stop, for subagents |
| `SessionStart` | When a session begins or resumes | Load project context |

## 3. Configuration

Hooks live in settings files (`~/.claude/settings.json` for user, `.claude/settings.json` for a shared project, `.claude/settings.local.json` for personal project settings). Each event holds matchers, and each matcher holds commands:

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

- `matcher` filters by tool name (e.g. `Bash`, `Edit|Write`, or `mcp__server__tool` for [MCP](mcp.md) tools). It applies to tool events.
- Use `/hooks` inside a session to view and manage configured hooks.

## 4. Input and Output

The hook command receives JSON on stdin (session id, event name, tool name and tool input, etc.). It communicates back through its exit code:

| Exit code | Meaning |
|-----------|---------|
| 0 | Success; stdout may be shown or used as context for some events |
| 2 | Blocking error; stderr is fed back to Claude (for `PreToolUse` the tool call is blocked) |
| Other | Non-blocking error; shown to the user, execution continues |

Hooks can also print structured JSON to stdout for finer control, such as allowing, denying or asking for permission on a `PreToolUse` call.

Example: block `rm -rf` in a `PreToolUse` hook on `Bash` by reading `tool_input.command` from stdin and exiting with code 2 plus an explanation on stderr.

## 5. Hooks vs Other Features

| Need | Best fit |
|------|----------|
| Guaranteed action every time | Hook |
| Guidance and conventions | `CLAUDE.md` |
| Allow or deny specific tools | Permission rules (hooks can add dynamic logic) |
| Reusable workflow Claude chooses | Skill or slash command |

## 6. Security Considerations

- Hooks run shell commands with your user permissions; review any hook before adding it, especially from a shared repository.
- Quote variables and validate input read from stdin to avoid injection.
- Use absolute paths or `$CLAUDE_PROJECT_DIR` so the hook works from any directory.
- Keep hooks fast; slow hooks stall the session.

## Key Takeaways

- Hooks are deterministic, harness-executed commands tied to lifecycle events.
- `PreToolUse` can block; `PostToolUse` suits formatting and checks.
- Configure them in `settings.json` with an event, a matcher and a command.
- Exit code 2 is a blocking error whose stderr goes back to Claude.
- Use hooks for rules that must always apply; use `CLAUDE.md` for guidance.

## Practice Questions

1. **Why use a hook instead of a `CLAUDE.md` instruction to run a formatter?**
   A hook is executed by the harness every time; a `CLAUDE.md` instruction is only guidance the model may skip.

2. **Which event lets you stop a tool call before it runs?**
   `PreToolUse`; exiting with code 2 (or returning a deny decision in JSON) blocks the call.

3. **What does exit code 2 do in a hook?**
   It is a blocking error: stderr is passed back to Claude, and for `PreToolUse` the tool call is blocked. Other non-zero codes are non-blocking.

4. **How would a hook run only when Claude edits files?**
   Use a `PostToolUse` hook with matcher `Edit|Write`.

5. **Why should you review hooks from a shared repo before trusting them?**
   Hooks run arbitrary shell commands with your user permissions.
