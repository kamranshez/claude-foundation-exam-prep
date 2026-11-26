# Claude Code Hooks

Hooks are user-defined commands that Claude Code runs automatically at specific points in its lifecycle, giving deterministic control instead of relying on the model to remember an instruction.

## 1. Why It Matters

Instructions in prompts or CLAUDE.md are advisory: Claude may forget or skip them. Hooks are executed by the harness, so they always run. Use them to enforce formatting, block dangerous commands, log activity or inject context.

## 2. Common Hook Events

| Event | When it fires | Typical use |
|-------|---------------|-------------|
| `PreToolUse` | Before a tool call runs | Block or validate commands, protect files |
| `PostToolUse` | After a tool call succeeds | Run formatters, linters, tests |
| `UserPromptSubmit` | When the user submits a prompt | Add context, validate prompts |
| `Notification` | When Claude needs attention | Desktop alerts |
| `Stop` | When Claude finishes responding | Final checks, notifications |
| `SubagentStop` | When a subagent finishes | Post-process subagent results |
| `SessionStart` | When a session begins or resumes | Load project context |

## 3. Configuration

Hooks live in settings files: `~/.claude/settings.json` (user), `.claude/settings.json` (project, shared) or `.claude/settings.local.json` (project, personal).

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

- `matcher` filters by tool name (regex-style; omit or use `*` to match all).
- Each hook receives event data as JSON on stdin.

## 4. Controlling Behavior with Exit Codes

| Exit code | Effect |
|-----------|--------|
| `0` | Success; continue normally |
| `2` | Blocking error; stderr is fed back to Claude (for `PreToolUse` the tool call is blocked) |
| Other | Non-blocking error; shown to the user, execution continues |

Hooks can also print structured JSON to stdout for finer control (for example allow, deny or ask decisions).

## 5. Example: Block Dangerous Commands

A `PreToolUse` hook with matcher `Bash` can read the command from stdin, and if it contains `rm -rf /`, write a reason to stderr and `exit 2`. Claude sees the reason and adjusts.

## 6. Hooks vs Other Features

| Need | Use |
|------|-----|
| Guaranteed, deterministic action | Hook |
| Guidance and conventions | CLAUDE.md |
| Reusable instructions on demand | Skill / slash command |
| Access to external tools or data | MCP server |

## 7. Security Notes

- Hooks run shell commands with your user permissions. Review hooks from untrusted repos before use.
- Quote variables and validate input to avoid injection.
- Keep hooks fast; they run in the agent loop.

## Key Takeaways

- Hooks are deterministic: the harness runs them, not the model.
- `PreToolUse` can block; `PostToolUse` is for follow-up actions such as formatting.
- Exit code `2` blocks and sends stderr back to Claude; `0` means success.
- Configure in `settings.json` with an event, optional `matcher` and a command.
- Prefer hooks over CLAUDE.md when an action must always happen.

## Practice Questions

1. **Why use a hook instead of a CLAUDE.md instruction to run a formatter after every edit?**
   Hooks are executed automatically and deterministically; CLAUDE.md is only guidance Claude may not follow.

2. **Which event lets you prevent a tool call from running?**
   `PreToolUse`, by exiting with code 2 (or returning a deny decision).

3. **What does exit code 2 from a hook do?**
   It signals a blocking error; stderr is passed back to Claude, and for `PreToolUse` the tool call is blocked.

4. **How do you make a hook run only for file edits?**
   Set `matcher` to the relevant tool names, e.g. `"Edit|Write"`.

5. **Where would you put a hook that should be shared with your whole team?**
   In the project's `.claude/settings.json`, committed to the repo.
