# Prompt Caching

Prompt caching lets the Claude API reuse a previously processed prompt prefix, cutting latency and cost for repeated context.

## 1. Why It Matters

Many requests repeat the same large prefix: a long system prompt, tool definitions, reference documents or conversation history. Without caching, Claude reprocesses all of it every call. With caching, the prefix is read from cache at a fraction of the normal input price and with lower latency.

## 2. How It Works

You mark a point in the prompt with a `cache_control` breakpoint. Everything **up to and including** that block is cached as a prefix.

```json
{
  "model": "claude-sonnet-5-5",
  "max_tokens": 1024,
  "system": [
    {
      "type": "text",
      "text": "<long instructions or reference document>",
      "cache_control": {"type": "ephemeral"}
    }
  ],
  "messages": [{"role": "user", "content": "Question about the document"}]
}
```

Prompt order for caching: **tools, then system, then messages**. A change at any level invalidates that level and everything after it.

## 3. Key Facts

| Topic | Detail |
|-------|--------|
| Cache match | Exact prefix match; any change before the breakpoint is a cache miss |
| Default lifetime (TTL) | About 5 minutes, refreshed each time the cache is hit |
| Longer TTL | An optional 1-hour TTL is available at a higher write price |
| Minimum size | The prefix must exceed a model-specific minimum token count to be cached |
| Breakpoints | Multiple allowed per request (a small fixed maximum) |
| Write cost | Cache writes cost more than normal input tokens |
| Read cost | Cache reads cost much less than normal input tokens |
| Usage fields | `cache_creation_input_tokens` and `cache_read_input_tokens` in the response |

Check current limits and prices in the official docs before relying on exact numbers.

## 4. Designing for Cache Hits

- Put **stable content first** (tools, system prompt, documents) and **variable content last** (the user's new question).
- Do not inject timestamps, request IDs or other changing values into the cached prefix.
- Keep tool definitions and their order identical between calls.
- For multi-turn chats, move the breakpoint to the latest turn so history is cached incrementally.
- Verify via the usage fields: if `cache_read_input_tokens` stays 0, the prefix is changing or is below the minimum size.

## 5. When to Use It

| Good fit | Poor fit |
|----------|----------|
| Long system prompt reused across many calls | One-off prompts |
| Q&A over a large document | Short prompts under the minimum size |
| Agents with many tool definitions | Prefix that changes every request |
| Long multi-turn conversations | Calls spaced further apart than the TTL |

## Key Takeaways

- Caching reuses an identical prompt **prefix**; it is enabled with `cache_control` breakpoints.
- Order matters: tools, system, messages. Stable first, variable last.
- Reads are cheap, writes cost a premium, and the default TTL is about 5 minutes (refreshed on use).
- Any change before a breakpoint breaks the cache.
- Monitor `cache_creation_input_tokens` and `cache_read_input_tokens` to confirm it works.

## Practice Questions

1. **What does a `cache_control` breakpoint cache?**
   The entire prompt prefix up to and including the marked block.

2. **Your cache read tokens are always 0. Name two likely causes.**
   The prefix changes between calls (e.g. a timestamp or reordered tools), or it is below the model's minimum cacheable size. Calls spaced beyond the TTL also cause misses.

3. **Where should the user's changing question go relative to the cached content?**
   After it. Stable content goes first and variable content last.

4. **What is the default cache TTL and what extends it?**
   About 5 minutes; each cache hit refreshes it. A longer 1-hour TTL is optionally available.

5. **Is caching always cheaper?**
   No. Writes cost more than normal input, so it pays off only when the prefix is reused enough times within the TTL.
