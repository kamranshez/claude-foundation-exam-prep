# Claude Code Hooks

Hooks are user-defined commands that Claude Code runs automatically at specific points in its lifecycle, giving you deterministic control instead of relying on the model to remember.

## 1. Why It Matters

Instructions in prompts or `CLAUDE.md` are advisory: Claude may forget or skip them. A hook is executed by the harness, so it always runs. Use hooks to enforce formatting, block dangerous commands, log activity or inject context.

## 2. Common Hook Events

| Event | Fires when | Typical use |
|-------|-----------|-------------|
| `PreToolUse` | Before a tool call runs | Block or approve a call, validate input |
| `PostToolUse` | After a tool call succeeds | Run a formatter or linter, log changes |
| `UserPromptSubmit` | User submits a prompt | Add context, validate or filter prompts |
| `Stop` | Claude finishes responding | Run tests, notify, force more work |
| `SubagentStop` | A subagent finishes | Post-process subagent results |
| `SessionStart` | A session starts or resumes | Load project context or env setup |
| `Notification` | Claude needs attention | Desktop or chat alerts |

## 3. Configuration

Hooks live in settings files: `~/.claude/settings.json` (user), `.claude/settings.json` (project, shareable) or `.claude/settings.local.json` (personal, not committed).

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          { "type": "command", "command": "npx prettier --write \"$CLAUDE_FILE_PATH\"" }
        ]
      }
    ]
  }
}
```

- `matcher` selects tools by name (regex-style, e.g. `Bash`, `Edit|Write`); omit or use `*` for all.
- The hook receives event details as JSON on stdin.

## 4. Controlling Behavior with Exit Codes

| Exit code | Effect |
|-----------|--------|
| `0` | Success; stdout may be shown or added as context |
| `2` | Blocking error; stderr is fed back to Claude (for `PreToolUse` the call is blocked) |
| Other | Non-blocking error; shown to the user, execution continues |

Hooks can also print JSON to give finer control, such as allowing, denying or asking for permission on a tool call.

## 5. Best Practices and Safety

- Keep hooks fast; they run synchronously in the loop.
- Hooks run with your user permissions, so review them like any script.
- Quote variables and validate input to avoid injection.
- Prefer project hooks in `.claude/settings.json` so the team shares them.
- Use hooks for rules that must always hold; use `CLAUDE.md` for guidance.

## Key Takeaways

- Hooks are deterministic, harness-run commands tied to lifecycle events.
- `PreToolUse` can block; `PostToolUse` reacts after the fact.
- Exit code `2` blocks and sends stderr back to Claude.
- Configure in `settings.json` with an event, optional `matcher`, and a command.
- Use hooks for guarantees, `CLAUDE.md` for guidance.

## Practice Questions

**1. Why use a hook instead of a `CLAUDE.md` instruction to run a formatter after edits?**
Hooks run deterministically every time; `CLAUDE.md` instructions are advisory and may be ignored.

**2. Which event lets you stop a dangerous `Bash` command before it runs?**
`PreToolUse`, with a `Bash` matcher, exiting with code `2` (or returning a deny decision).

**3. What does exit code 2 from a hook do?**
It signals a blocking error: stderr is returned to Claude, and a `PreToolUse` call is blocked.

**4. Where do you put a hook the whole team should share?**
In the project's `.claude/settings.json`, committed to the repo.

**5. How does a hook receive information about the tool call?**
As JSON on standard input.
