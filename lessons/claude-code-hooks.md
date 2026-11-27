# Claude Code Hooks

Hooks are user-defined commands that Claude Code runs automatically at specific points in its lifecycle, giving deterministic control instead of relying on the model to remember an instruction.

## 1. Why It Matters

A line in `CLAUDE.md` is a request the model may or may not follow. A hook is executed by the harness, so it always runs. Use hooks for things that must happen every time: formatting, blocking dangerous commands, logging, notifications. Related: [MCP](mcp.md) tool names (`mcp__<server>__<tool>`) can be matched by hooks too.

## 2. Common Hook Events

| Event | Fires when | Typical use |
|-------|-----------|-------------|
| `PreToolUse` | Before a tool call runs | Block or validate commands |
| `PostToolUse` | After a tool call succeeds | Run formatter, linter, tests |
| `UserPromptSubmit` | User submits a prompt | Add context, validate input |
| `Notification` | Claude needs attention | Desktop/Slack alert |
| `Stop` | Claude finishes responding | Final checks, summaries |
| `SubagentStop` | A subagent finishes | Post-process subagent work |
| `SessionStart` | A session begins or resumes | Load project context |

## 3. Configuration

Hooks live in settings files: `~/.claude/settings.json` (user), `.claude/settings.json` (project, shared), `.claude/settings.local.json` (project, personal). A `matcher` selects which tools the hook applies to.

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

The `/hooks` command in a session lets you view and manage configured hooks.

## 4. Input and Control

The hook receives JSON on stdin (session id, tool name, tool input, etc.). It controls Claude through its exit code:

| Exit code | Effect |
|-----------|--------|
| `0` | Success; proceed (stdout may be added as context for some events) |
| `2` | Blocking error; stderr is fed back to Claude, and for `PreToolUse` the tool call is blocked |
| Other | Non-blocking error; shown to the user, execution continues |

Hooks can also print structured JSON to stdout for finer decisions (e.g. allow, deny, or ask for a tool call).

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

| Mechanism | Deterministic? | Best for |
|-----------|----------------|----------|
| Hook | Yes | Must-always-happen automation and guardrails |
| `CLAUDE.md` | No (guidance) | Conventions and context |
| Permission rules | Yes | Allow/deny tool access |

## 6. Security Considerations

- Hooks run with your user permissions and can execute arbitrary commands; review any hook before adding it, especially from shared repos.
- Quote shell variables and validate input from stdin to avoid injection.
- Use absolute paths or `$CLAUDE_PROJECT_DIR` for scripts.
- Keep hooks fast; they run synchronously in the agent loop.

## Key Takeaways

- Hooks give deterministic, harness-executed behavior at lifecycle events.
- `PreToolUse` can block; `PostToolUse` is for follow-up actions like formatting.
- Exit code `2` blocks and sends stderr back to Claude; `0` succeeds.
- Configure in user, project, or local settings JSON; use `matcher` to target tools.
- Use hooks for rules that must always run; use `CLAUDE.md` for soft guidance.

## Practice Questions

1. **Why choose a hook over a `CLAUDE.md` instruction for running a formatter after every edit?**
   A hook is executed by the harness every time, so it is deterministic; a `CLAUDE.md` instruction relies on the model following it.

2. **Which event and exit code would you use to stop Claude from running a forbidden shell command?**
   A `PreToolUse` hook matching the Bash tool that exits with code `2`, writing the reason to stderr.

3. **Where can you define a hook that is shared with your team through version control?**
   `.claude/settings.json` in the project (committed). Personal-only hooks go in `.claude/settings.local.json` or `~/.claude/settings.json`.

4. **How does a hook receive information about the tool call being made?**
   As JSON on stdin, including the tool name and tool input.

5. **What is a security risk of hooks?**
   They run arbitrary commands with your user permissions, so untrusted or unreviewed hooks can be harmful.
