# Claude Code Hooks

Hooks are user-defined commands that Claude Code runs automatically at specific points in its lifecycle, giving deterministic control instead of relying on the model to remember an instruction.

## 1. Why It Matters

Prompts and `CLAUDE.md` guide Claude, but the model may not always follow them. Hooks are executed by the Claude Code harness, so they always run. Use them to enforce rules (block dangerous commands), automate chores (format after edits) and add context (inject project state).

## 2. Hook Events

| Event | Fires when | Typical use |
|-------|-----------|-------------|
| `SessionStart` | A session begins or resumes | Load context, set env vars |
| `UserPromptSubmit` | User submits a prompt, before Claude sees it | Validate or add context |
| `PreToolUse` | Before a tool call runs | Block or approve calls, modify policy |
| `PostToolUse` | After a tool call succeeds | Run formatters, linters, tests |
| `Notification` | Claude needs attention or permission | Desktop or chat alerts |
| `Stop` | Claude finishes responding | Check work, force continuation |
| `SubagentStop` | A subagent finishes | Validate subagent output |
| `PreCompact` | Before context compaction | Save or adjust state |
| `SessionEnd` | Session ends | Cleanup, logging |

## 3. Configuration

Hooks live in settings files (`~/.claude/settings.json`, `.claude/settings.json`, `.claude/settings.local.json`). A `matcher` selects which tools the hook applies to (tool names or regex; omit or use `*` for all).

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          { "type": "command", "command": "prettier --write \"$CLAUDE_PROJECT_DIR\"" }
        ]
      }
    ]
  }
}
```

Use `/hooks` in Claude Code to view and edit hooks interactively.

## 4. Input and Output

- Input: the hook receives JSON on **stdin** (session ID, event name, `tool_name`, `tool_input`, working directory, and so on).
- Output is controlled by **exit code**:

| Exit code | Meaning |
|-----------|---------|
| `0` | Success; stdout may be shown or added as context (for `UserPromptSubmit` and `SessionStart`) |
| `2` | Blocking error; stderr is fed back to Claude, and `PreToolUse` blocks the tool call |
| Other | Non-blocking error; shown to the user, execution continues |

- Hooks can instead print structured JSON on stdout for finer control, such as a permission decision (`allow`, `deny`, `ask`) or `"continue": false` to stop processing.

Example `PreToolUse` guard (blocks `rm -rf`):

```bash
#!/bin/bash
cmd=$(jq -r '.tool_input.command // ""')
if echo "$cmd" | grep -q 'rm -rf'; then
  echo "Blocked: rm -rf is not allowed" >&2
  exit 2
fi
exit 0
```

## 5. Hooks vs Other Mechanisms

| Mechanism | Nature | Enforced? |
|-----------|--------|-----------|
| `CLAUDE.md` | Instructions the model reads | No, advisory |
| Permissions (`allow`/`deny`) | Static rules on tools | Yes |
| Hooks | Custom code at lifecycle events | Yes, deterministic |
| Skills | Task instructions loaded on demand | No, advisory |

## 6. Security Considerations

- Hooks run shell commands with **your user's permissions**; review them before adding, especially from shared repos.
- Treat the stdin JSON as untrusted: quote variables, validate paths and avoid `eval`.
- Use absolute paths or `$CLAUDE_PROJECT_DIR` to avoid running the wrong script.
- Keep hooks fast; slow hooks stall the session (they have a timeout).

## Key Takeaways

- Hooks give deterministic, harness-enforced automation at lifecycle events.
- `PreToolUse` can block actions; `PostToolUse` reacts after them.
- Exit code `2` is the blocking signal and feeds stderr back to Claude; `0` is success.
- Configure in `settings.json` with an event, optional `matcher` and a command.
- Use hooks for rules that must always happen; use `CLAUDE.md` for guidance.

## Practice Questions

**1. Which hook event lets you stop a tool call before it runs?**
`PreToolUse`, by exiting with code 2 (or returning a `deny` decision in JSON).

**2. How does a hook receive information about the tool call?**
As JSON on stdin, including fields such as `tool_name` and `tool_input`.

**3. You want code auto-formatted after every file edit. Which event and matcher?**
`PostToolUse` with matcher `Edit|Write`, running a formatter command.

**4. Why use a hook instead of writing "always run the linter" in `CLAUDE.md`?**
`CLAUDE.md` is advisory and the model may skip it; a hook is executed by the harness every time.

**5. What does exit code 2 do in a hook?**
It signals a blocking error: stderr is sent back to Claude, and for `PreToolUse` the tool call is blocked.
