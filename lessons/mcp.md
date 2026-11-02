# Model Context Protocol (MCP)

MCP is an open protocol that lets AI applications connect to external tools and data sources through one standard interface.

## 1. Why It Matters

Without a standard, every app needs a custom integration for every tool (an M x N problem). With MCP, a tool is wrapped once as an **MCP server** and any **MCP client** (Claude Code, Claude Desktop, your own app) can use it. It builds directly on [tool use](tool-use.md).

## 2. Architecture

| Role | Description | Example |
|------|-------------|---------|
| Host | The AI application the user interacts with | Claude Code, Claude Desktop |
| Client | Connector inside the host; one per server connection | Built into the host |
| Server | Program exposing capabilities over MCP | GitHub, filesystem, database server |

Messages use JSON-RPC 2.0.

## 3. What Servers Expose (Primitives)

| Primitive | Controlled by | Purpose | Example |
|-----------|---------------|---------|---------|
| Tools | Model | Actions Claude can call | `create_issue`, `run_query` |
| Resources | Application | Read-only data/context | A file, a DB schema |
| Prompts | User | Reusable prompt templates | A `/review-pr` command |

## 4. Transports

| Transport | Use case |
|-----------|----------|
| stdio | Local server launched as a subprocess on your machine |
| Streamable HTTP | Remote, hosted servers (supports OAuth-based auth) |

Older SSE-based remote transport is deprecated in favor of Streamable HTTP.

## 5. Using MCP in Claude Code

```bash
# Add a remote HTTP server
claude mcp add --transport http github https://example.com/mcp

# Add a local stdio server
claude mcp add my-server -- npx -y some-mcp-server

claude mcp list      # show configured servers
/mcp                 # inside a session: status and authentication
```

Scopes: **local** (you, this project), **project** (shared via `.mcp.json` in the repo), **user** (you, all projects). MCP tools appear to Claude named like `mcp__<server>__<tool>`, and are subject to the normal permission system.

## 6. Security Considerations

- Only install servers you trust; a server's tool descriptions and results go into Claude's context.
- Untrusted content returned by a tool can contain prompt injection.
- Grant least privilege (read-only credentials where possible) and review tool permissions.
- Never commit secrets in `.mcp.json`; use environment variable expansion.

## Key Takeaways

- MCP standardizes how hosts connect to tools and data via clients and servers.
- Servers expose tools (model-controlled), resources (app-controlled) and prompts (user-controlled).
- Use stdio for local servers and Streamable HTTP for remote ones.
- In Claude Code, `claude mcp add` registers servers; scopes are local, project and user.
- Treat MCP servers as a trust boundary: least privilege and watch for prompt injection.

## Practice Questions

**1. What problem does MCP solve?**
It replaces many custom point-to-point integrations with one standard protocol, so any MCP client can use any MCP server.

**2. Name the three main primitives a server can expose and who controls each.**
Tools (model), resources (application), prompts (user).

**3. Which transport suits a local server, and which a remote one?**
stdio for local subprocess servers; Streamable HTTP for remote servers.

**4. How do you share an MCP server configuration with your whole team in Claude Code?**
Use project scope, which stores it in `.mcp.json` in the repository (and avoid committing secrets).

**5. Why is connecting an untrusted MCP server risky?**
Its tool descriptions and results enter Claude's context and can carry prompt injection or trigger harmful actions.
