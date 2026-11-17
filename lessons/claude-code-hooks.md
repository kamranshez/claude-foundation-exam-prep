# Claude Code Hooks

Hooks are user-defined commands that Claude Code runs automatically at specific points in its lifecycle, giving you deterministic control instead of relying on the model to remember.

## 1. Why It Matters

Instructions in prompts or `CLAUDE.md` are advisory: Claude may forget or ignore them. Hooks are executed by the harness, so they always run. Use them to enforce formatting, block dangerous commands, log activity or inject context.

## 2. Common Hook Events

| Event | When it fires | Typical use |
|-------|---------------|-------------|
| `PreToolUse` | Before a tool call runs | Block or validate a call (e.g. deny `rm -rf`) |
| `PostToolUse` | After a tool call succeeds | Run a formatter or linter on edited files |
| `UserPromptSubmit` | When the user submits a prompt | Add context or validate the prompt |
| `Stop` | When Claude finishes responding | Run tests, notify the user |
| `SubagentStop` | When a subagent finishes | Post-process subagent results |
| `SessionStart` | When a session begins or resumes | Load environment or project context |
| `Notification` | When Claude sends a notification | Desktop or chat alerts |

## 3. Configuration

Hooks live in `settings.json` (user `~/.claude/settings.json`, project `.claude/settings.json`, or local `.claude/settings.local.json`).

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

- `matcher` filters by tool name (regex-style, e.g. `Bash`, `Edit|Write`). Omit it or use `*` to match all.
- `type: "command"` runs a shell command.
- The hook receives event details as JSON on stdin (tool name, tool input, session info).

## 4. Controlling Claude with Exit Codes

| Exit code | Effect |
|-----------|--------|
| `0` | Success. Stdout is shown to the user (and added as context for some events such as `UserPromptSubmit`). |
| `2` | Blocking error. For `PreToolUse` the tool call is blocked and stderr is fed back to Claude. |
| Other | Non-blocking error. Shown to the user, execution continues. |

Hooks can also print structured JSON to stdout for finer control (e.g. allow, deny or ask for permission, or stop Claude).

Example `PreToolUse` guard (bash):

```bash
#!/bin/bash
cmd=$(jq -r '.tool_input.command')
if echo "$cmd" | grep -q 'rm -rf'; then
  echo "Blocked: destructive command" >&2
  exit 2
fi
```

## 5. Hooks vs Other Mechanisms

| Mechanism | Nature | Best for |
|-----------|--------|----------|
| `CLAUDE.md` | Advisory instructions in context | Conventions, project knowledge |
| Permissions rules | Allow/deny lists for tools | Coarse access control |
| Hooks | Deterministic code run by the harness | Enforcement, automation, validation |

## 6. Security Notes

- Hooks run with your user permissions and execute arbitrary shell commands. Review any hook before adding it, especially in shared project settings.
- Quote variables and validate input to avoid shell injection.
- Keep hooks fast; slow hooks delay every matching action.

## Key Takeaways

- Hooks are deterministic: the harness runs them, not the model.
- `PreToolUse` can block; `PostToolUse` reacts after the fact.
- Exit code `2` blocks and returns stderr to Claude; `0` is success.
- Configure with `matcher` + `hooks` in `settings.json`; input arrives as JSON on stdin.
- Use hooks for enforcement, and `CLAUDE.md` for guidance.

## Practice Questions

1. **Why use a hook instead of a `CLAUDE.md` instruction to run a formatter after every edit?**
   Hooks are executed by the harness every time, so they are guaranteed. `CLAUDE.md` instructions are advisory and Claude may skip them.

2. **Which event and exit code would you use to stop Claude running a dangerous Bash command?**
   A `PreToolUse` hook with matcher `Bash` that exits with code `2`. Stderr is returned to Claude as feedback.

3. **How does a hook learn which tool is being called and with what input?**
   The harness passes event data as JSON on stdin, including the tool name and `tool_input`.

4. **What does the `matcher` field do?**
   It filters which tool names trigger the hook (e.g. `Edit|Write`); omitting it matches all tools.

5. **Name a security concern with hooks.**
   They run arbitrary shell commands with your permissions, so untrusted or poorly written hooks (e.g. in a shared repo) can cause harm or injection.
