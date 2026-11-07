# Model Context Protocol (MCP)

MCP is an open standard that lets AI applications connect to external tools and data sources through one common protocol, instead of custom integrations for each.

## 1. Why It Matters

Without a standard, every app needs a custom connector for every data source (N x M integrations). With MCP, a data source implements one MCP server and any MCP-capable app can use it. It builds on [tool use](tool-use.md): MCP servers expose tools that Claude calls.

## 2. Architecture

| Component | Role |
|-----------|------|
| Host | The AI application the user interacts with (e.g. Claude Code, Claude Desktop) |
| Client | Connector inside the host; keeps a 1:1 connection to one server |
| Server | Program exposing capabilities from a system (files, GitHub, a database, etc.) |

Messages use JSON-RPC 2.0.

## 3. What Servers Expose

| Primitive | Controlled by | Purpose | Example |
|-----------|---------------|---------|---------|
| Tools | Model | Actions Claude can invoke | `create_issue`, `run_query` |
| Resources | Application | Read-only data/context | A file, a DB schema |
| Prompts | User | Reusable prompt templates | A "review this PR" template |

## 4. Transports

| Transport | Use |
|-----------|-----|
| stdio | Server runs as a local subprocess; best for local tools |
| Streamable HTTP | Remote servers over the network (supports auth such as OAuth) |

An older HTTP+SSE transport exists but is deprecated in favour of Streamable HTTP.

## 5. Using MCP in Claude Code

```bash
# Add a remote HTTP server
claude mcp add --transport http my-server https://example.com/mcp

# Add a local stdio server
claude mcp add my-local -- npx -y some-mcp-server

claude mcp list
claude mcp remove my-server
```

Scopes: `local` (just you, this project), `project` (shared via `.mcp.json` in version control), `user` (all your projects). Inside a session, `/mcp` shows server status and handles authentication.

MCP tools appear to Claude named like `mcp__<server>__<tool>`, which is also how you reference them in permission rules.

## 6. Security Considerations

- Only install servers you trust; a server's tool output is untrusted input and can contain prompt injection.
- Use least privilege: scope credentials and allow only needed tools.
- Project-scoped servers require user approval before first use.
- Keep human confirmation for destructive actions.

## Key Takeaways

- MCP standardises how AI apps connect to tools and data (host, client, server).
- Servers expose tools, resources and prompts.
- stdio is for local servers; Streamable HTTP for remote ones.
- In Claude Code, `claude mcp add` registers servers; scopes are local, project and user.
- Treat MCP server output as untrusted and apply least privilege.

## Practice Questions

1. **What problem does MCP solve?**
   It replaces many custom point-to-point integrations with one standard protocol, so any MCP client can use any MCP server.

2. **Name the three primitives an MCP server can expose and who controls each.**
   Tools (model-controlled), resources (application-controlled), prompts (user-controlled).

3. **When would you choose stdio over Streamable HTTP?**
   For a server running locally as a subprocess on the user's machine; HTTP is for remote, network-hosted servers.

4. **How do you share an MCP server configuration with your whole team in Claude Code?**
   Add it with project scope, which stores it in `.mcp.json` committed to the repository.

5. **Why is MCP tool output a security concern?**
   It comes from an external system and may contain prompt injection, so it must be treated as untrusted input.
