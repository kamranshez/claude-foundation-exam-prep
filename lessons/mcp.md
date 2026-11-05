# Model Context Protocol (MCP)

MCP is an open standard that lets AI applications connect to external tools, data and prompts through one common protocol, instead of custom integrations for each service.

## 1. Why It Matters

Without a standard, every app needs a bespoke connector for every data source (an N x M problem). With MCP, a service exposes one MCP server and any MCP-capable client (Claude Code, Claude Desktop, your own app) can use it.

## 2. Architecture

| Component | Role |
|-----------|------|
| Host | The AI application the user interacts with (e.g. Claude Code) |
| Client | Connector inside the host; keeps a 1:1 connection to one server |
| Server | Program that exposes capabilities (GitHub, a database, a file system) |

Messages use JSON-RPC 2.0.

## 3. What Servers Expose (Primitives)

| Primitive | Controlled by | Purpose | Example |
|-----------|---------------|---------|---------|
| Tools | Model | Actions Claude can call | `create_issue`, `run_query` |
| Resources | Application | Read-only data/context | A file, a DB schema |
| Prompts | User | Reusable prompt templates | A `/review` command |

Tools in MCP map directly onto the tool use loop (see [Tool Use](tool-use.md)): the client lists the server's tools, Claude requests a call, and the client forwards it to the server.

## 4. Transports

| Transport | Use |
|-----------|-----|
| stdio | Local server launched as a subprocess; simple, no network |
| Streamable HTTP | Remote server over HTTP; supports auth such as OAuth |
| SSE | Older remote transport, being deprecated in favour of HTTP |

## 5. Using MCP in Claude Code

```bash
# Add a remote HTTP server
claude mcp add --transport http github https://example.com/mcp

# Add a local stdio server
claude mcp add my-db -- npx -y some-mcp-server

claude mcp list
claude mcp remove my-db
```

Scopes control where a server is configured:

| Scope | Stored in | Shared? |
|-------|-----------|---------|
| local | User's private project settings | No (default) |
| project | `.mcp.json` in the repo | Yes, via version control |
| user | User config | Across all your projects |

MCP tools appear to Claude with names like `mcp__<server>__<tool>`, which is also how you write permission rules for them.

## 6. Security

- Only install servers you trust; tool results can contain prompt injection.
- Grant least privilege (read-only credentials where possible).
- Review tool calls, and require approval for destructive actions.
- Project-scoped servers from `.mcp.json` require user approval before first use.

## Key Takeaways

- MCP standardises how AI apps connect to tools, resources and prompts.
- Host contains clients; each client talks to one server over JSON-RPC.
- Tools are model-controlled, resources app-controlled, prompts user-controlled.
- Transports: stdio for local, Streamable HTTP for remote.
- In Claude Code, scope (local, project, user) decides who shares a server; `.mcp.json` is the shared project file.
- Treat MCP servers as a trust and prompt-injection boundary.

## Practice Questions

1. **What problem does MCP solve?**
   It replaces many custom point-to-point integrations with one standard protocol between AI apps and external systems.

2. **Name the three primitives a server can expose and who controls each.**
   Tools (model), resources (application), prompts (user).

3. **Which transport suits a local server, and which a remote one?**
   stdio for local subprocess servers; Streamable HTTP for remote servers.

4. **How do you share an MCP server with your whole team in Claude Code?**
   Use project scope, which stores it in `.mcp.json` in the repo and is committed to version control.

5. **Why is connecting an untrusted MCP server risky?**
   Its tool descriptions and results are fed to Claude and can contain prompt injection or trigger harmful actions, so only trusted servers and least-privilege credentials should be used.
