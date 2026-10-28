# Tool Use

Tool use lets Claude request actions, such as calling functions or APIs, that your application (or Anthropic) executes, then continue with the results.

## 1. Why It Matters

Claude only generates text. Tools let it fetch live data, run calculations, query systems and take actions. Tool use is the foundation of agents and of MCP.

## 2. The Tool Use Loop

1. You send a request with `tools` definitions and a user message.
2. Claude decides a tool is needed and responds with `stop_reason: "tool_use"` and a `tool_use` block (`id`, `name`, `input`).
3. Your code runs the tool.
4. You send a new request that appends Claude's response and a `tool_result` block in a `user` message, with a matching `tool_use_id`.
5. Claude produces a final answer (`stop_reason: "end_turn"`) or requests more tools. Repeat until done.

Claude never executes client tools itself. Your application does.

## 3. Defining a Tool

```json
{
  "name": "get_weather",
  "description": "Get the current weather for a city. Use when the user asks about current conditions.",
  "input_schema": {
    "type": "object",
    "properties": {
      "city": {"type": "string", "description": "City name, e.g. Paris"}
    },
    "required": ["city"]
  }
}
```

The `description` is the most important field: Claude uses it to decide when and how to call the tool. Be specific about purpose, inputs and limits.

## 4. Key Facts

| Topic | Detail |
|-------|--------|
| Client tools | Run in your app; you return the `tool_result` |
| Server tools | Run on Anthropic's infrastructure (e.g. web search); no result handling needed |
| `tool_choice: auto` | Claude decides whether to use a tool (default) |
| `tool_choice: any` | Claude must use some tool |
| `tool_choice: tool` | Claude must use a specific named tool |
| `tool_choice: none` | Claude may not use tools |
| Parallel calls | Claude may return several `tool_use` blocks in one turn; return all results together in one user message |
| Errors | Set `is_error: true` in the `tool_result` so Claude can recover or retry |
| Structured output | Forcing a tool with a JSON schema is a common way to get reliable JSON |
| Caching | Tool definitions come first in the prompt, so they are good cache prefixes |

## 5. Best Practices

- Write clear names and detailed descriptions; add parameter descriptions and enums.
- Keep tool sets small and focused; too many similar tools cause wrong choices.
- Validate tool inputs; treat them as untrusted.
- Return concise, relevant results to save context tokens.
- Require confirmation for destructive or irreversible actions.

## Key Takeaways

- Claude requests tools; your code executes client tools and returns `tool_result` blocks.
- The loop continues while `stop_reason` is `tool_use`.
- Tool descriptions drive tool selection, so write them carefully.
- Use `tool_choice` to control whether and which tools are called.
- Report failures with `is_error: true` instead of hiding them.

## Practice Questions

**1. What `stop_reason` indicates Claude wants to call a tool?**
`tool_use`.

**2. Where must the result of a client tool be sent, and how is it linked to the call?**
In a `user` message as a `tool_result` block whose `tool_use_id` matches the `id` of the `tool_use` block.

**3. How do you force Claude to call one specific tool?**
Set `tool_choice` to `{"type": "tool", "name": "<tool_name>"}`.

**4. A tool fails with a timeout. How should you tell Claude?**
Return a `tool_result` with `is_error: true` and a helpful error message so Claude can retry or explain.

**5. Which tool definition field most influences whether Claude picks the tool?**
The `description`.
