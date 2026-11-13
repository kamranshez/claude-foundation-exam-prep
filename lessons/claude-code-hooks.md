# Claude Code Hooks

Hooks are user-defined commands that Claude Code runs automatically at specific points in its lifecycle, giving deterministic control instead of relying on the model to remember an instruction.

## 1. Why It Matters

Instructions in prompts or `CLAUDE.md` are advisory: Claude may forget or ignore them. Hooks are executed by the harness, so they always run. Use them to enforce formatting, block dangerous commands, log activity or inject context.

## 2. Hook Events

| Event | When it fires | Typical use |
|-------|---------------|-------------|
| `PreToolUse` | Before a tool call runs | Block or modify risky calls (e.g. `rm -rf`) |
| `PostToolUse` | After a tool call succeeds | Run formatter, linter or tests |
| `UserPromptSubmit` | When the user submits a prompt | Add context, validate prompts |
| `Notification` | When Claude sends a notification | Desktop or Slack alerts |
| `Stop` | When Claude finishes responding | Final checks, force continuation |
| `SubagentStop` | When a subagent finishes | Validate subagent output |
| `SessionStart` | When a session begins or resumes | Load project context |

## 3. Configuration

Hooks live in `settings.json` (user `~/.claude/settings.json`, project `.claude/settings.json`, or local `.claude/settings.local.json`).

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

- `matcher` selects tools by name (regex such as `Edit|Write`, or `Bash`). Omit or use `*` to match all.
- The hook receives event data as JSON on stdin (session id, tool name, tool input).

## 4. Exit Codes and Control

| Exit code | Effect |
|-----------|--------|
| `0` | Success; stdout is shown to the user (and added as context for some events) |
| `2` | Blocking error; stderr is fed back to Claude. For `PreToolUse` the tool call is blocked |
| Other | Non-blocking error; shown to the user, execution continues |

Hooks can also print JSON to stdout for finer control (e.g. approve, deny or ask for a tool call, or stop Claude).

## 5. Hooks vs Other Features

| Feature | Nature | Use for |
|---------|--------|---------|
| `CLAUDE.md` | Advisory instructions | Conventions, project knowledge |
| Permissions | Allow/deny rules per tool | Coarse access control |
| Hooks | Deterministic code | Guaranteed side effects and checks |
| Skills / slash commands | Reusable prompts | Repeatable workflows |

## 6. Security Notes

- Hooks run with your user permissions; review any hook before enabling it.
- Quote shell variables and validate input to avoid injection.
- Project hooks come from the repo, so treat untrusted repos with caution.

## Key Takeaways

- Hooks are deterministic: the harness runs them, not the model.
- `PreToolUse` can block actions; `PostToolUse` suits formatting and tests.
- Exit code `2` is the blocking signal and feeds stderr back to Claude.
- Configure in `settings.json` using event names and tool `matcher` patterns.
- Use hooks for must-happen rules; use `CLAUDE.md` for guidance.

## Practice Questions

1. **Why use a hook instead of a `CLAUDE.md` instruction to run a formatter after every edit?**
   Hooks are executed by the harness every time, so they are guaranteed. `CLAUDE.md` is only advisory.

2. **Which event and exit code block a dangerous Bash command before it runs?**
   A `PreToolUse` hook with matcher `Bash` that exits with code `2`. Its stderr is returned to Claude.

3. **How does a hook receive information about the tool call?**
   As JSON on stdin, including the tool name and tool input.

4. **What happens if a hook exits with code 1?**
   It is a non-blocking error: the message is shown to the user and Claude continues.

5. **Where can hooks be defined?**
   In user, project or local `settings.json` files under the `hooks` key.
