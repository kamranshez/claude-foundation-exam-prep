# Claude Code Hooks

Hooks are user-defined commands that Claude Code runs automatically at specific points in its lifecycle, giving you deterministic control instead of relying on the model to remember an instruction.

## 1. Why It Matters

A line in a prompt or `CLAUDE.md` is a request; Claude may not always follow it. A hook is executed by the harness itself, so it **always** runs. Use hooks to enforce rules (block dangerous commands), automate chores (format after every edit) and add context. See also [Tool Use](tool-use.md) and [MCP](mcp.md), since hooks commonly match tool calls, including `mcp__<server>__<tool>` names.

## 2. Common Hook Events

| Event | Fires | Typical use |
|-------|-------|-------------|
| `PreToolUse` | Before a tool call runs | Block or approve a call, validate input |
| `PostToolUse` | After a tool call succeeds | Run formatter/linter/tests |
| `UserPromptSubmit` | When the user submits a prompt | Add context, validate prompts |
| `Stop` | When Claude finishes responding | Force more work, run final checks |
| `SubagentStop` | When a subagent finishes | Same as `Stop` for subagents |
| `Notification` | When Claude Code sends a notification | Desktop or chat alert |
| `SessionStart` | When a session starts or resumes | Load project context |
| `PreCompact` | Before context compaction | Save state |

The set of events grows over time; check the official docs for the current list.

## 3. Configuration

Hooks live in a settings file under a `hooks` key. Each event has a list of matchers, and each matcher has one or more hooks.

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

| Setting file | Scope |
|--------------|-------|
| `~/.claude/settings.json` | All your projects |
| `.claude/settings.json` | This project, shared via version control |
| `.claude/settings.local.json` | This project, just you (not committed) |

- `matcher` is a tool name or regex (`Bash`, `Edit|Write`, `mcp__github__.*`); `*` or empty matches everything. Events without tools ignore it.
- The `/hooks` command shows and edits configured hooks.

## 4. Input and Output

The hook command receives JSON on **stdin** (session ID, event name, and for tool events the `tool_name` and `tool_input`).

| Exit code | Meaning |
|-----------|---------|
| `0` | Success; stdout may be shown or added as context for some events |
| `2` | **Blocking error**; stderr is fed back to Claude (for `PreToolUse` the tool call is blocked) |
| Other | Non-blocking error; shown to the user, execution continues |

A hook can also print structured JSON for finer control, e.g. a `PreToolUse` hook can return a permission decision of allow, deny or ask.

Example `PreToolUse` script that blocks `rm -rf`:

```bash
#!/bin/bash
cmd=$(jq -r '.tool_input.command' < /dev/stdin)
if echo "$cmd" | grep -q 'rm -rf'; then
  echo "Blocked: rm -rf is not allowed" >&2
  exit 2
fi
exit 0
```

## 5. Hooks vs Other Features

| Feature | Nature | Use when |
|---------|--------|----------|
| Hook | Deterministic, runs outside the model | A rule must always happen |
| `CLAUDE.md` | Guidance Claude reads | Conventions and preferences |
| Permission rules | Allow/deny lists for tools | Simple tool access control |
| Skill / slash command | Model- or user-invoked instructions | Reusable workflows |

## 6. Security Considerations

- Hooks run shell commands with **your user permissions**; review any hook before adding it, especially from a shared repo.
- Quote variables and validate input from stdin to avoid command injection.
- Use absolute paths or `$CLAUDE_PROJECT_DIR` for scripts.
- Keep hooks fast; slow hooks stall the session (they have a timeout).

## Key Takeaways

- Hooks give deterministic, always-on automation at lifecycle events; prompts only suggest.
- `PreToolUse` can block or approve; `PostToolUse` reacts after a tool succeeds.
- Configure under `hooks` in user, project or local settings; use `matcher` to target tools.
- Input arrives as JSON on stdin; exit code `2` blocks and sends stderr to Claude.
- Hooks execute with your permissions, so treat them as trusted code and review them.

## Practice Questions

1. **Why use a hook instead of a `CLAUDE.md` instruction to run a formatter after edits?**
   A hook is executed by the harness every time, so it is deterministic. A `CLAUDE.md` instruction is guidance the model might skip.

2. **Which event and exit code would you use to stop Claude running a dangerous Bash command?**
   A `PreToolUse` hook with matcher `Bash` that exits with code `2` (stderr explains the reason to Claude); alternatively return a JSON deny decision.

3. **Where do you put a hook that should apply to the whole team but not to your other projects?**
   In `.claude/settings.json` in the repo, committed to version control. Personal-only hooks go in `.claude/settings.local.json`.

4. **How does a hook script learn which tool is being called and with what input?**
   Claude Code passes JSON on stdin containing fields such as `tool_name` and `tool_input`.

5. **What is the main security risk of hooks?**
   They run arbitrary shell commands with your user permissions, so a malicious or careless hook (for example from a cloned repo) can do harm. Review hooks before trusting them.
