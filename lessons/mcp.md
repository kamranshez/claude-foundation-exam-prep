# Model Context Protocol (MCP)

MCP is an open standard that lets AI applications connect to external tools, data and prompts through one common protocol, instead of custom integrations for each service.

## 1. Why It Matters

Without a standard, every app needs a custom connector for every data source (an N x M problem). With MCP, a service implements an **MCP server** once and any **MCP client** (Claude Code, Claude Desktop, your own agent) can use it. MCP builds on [tool use](tool-use.md).

## 2. Architecture

| Role | Description | Example |
|------|-------------|---------|
| Host | The AI application the user interacts with | Claude Code, Claude Desktop |
| Client | Connector inside the host; keeps one connection per server | Created by the host |
| Server | Program exposing capabilities over MCP | GitHub, filesystem, database server |

Messages use JSON-RPC 2.0.

## 3. What Servers Expose (Primitives)

| Primitive | Controlled by | Purpose | Example |
|-----------|---------------|---------|---------|
| Tools | Model | Actions Claude can call | `create_issue`, `run_query` |
| Resources | Application | Read-only data/context | File contents, DB schema |
| Prompts | User | Reusable prompt templates | `/review-pr` command |

## 4. Transports

| Transport | Use |
|-----------|-----|
| stdio | Local server launched as a subprocess of the client |
| Streamable HTTP | Remote servers over HTTP (supports OAuth for auth) |

## 5. Using MCP in Claude Code

```bash
# Add a remote HTTP server
claude mcp add --transport http github https://example.com/mcp

# Add a local stdio server
claude mcp add my-server -- npx -y some-mcp-server

claude mcp list
```

Scopes: **local** (you, this project), **project** (shared via `.mcp.json` in the repo), **user** (you, all projects). Tools appear as `mcp__<server>__<tool>` and are subject to the normal permission system.

## 6. Security Considerations

- Only install servers you trust; tool output can contain prompt injection.
- Grant least privilege and require approval for destructive tools.
- Keep secrets in environment variables, not in committed `.mcp.json`.

## Key Takeaways

- MCP standardizes how AI apps connect to tools and data (host, client, server).
- Servers expose tools, resources and prompts.
- Transports: stdio for local, Streamable HTTP for remote.
- Claude Code configures servers with `claude mcp add`; project scope uses `.mcp.json`.
- Treat third-party servers and their output as untrusted.

## Practice Questions

**1. What problem does MCP solve?**
It replaces many custom point-to-point integrations with one standard protocol between AI clients and servers.

**2. Name the three primitives an MCP server can expose.**
Tools, resources and prompts.

**3. Which transport suits a local server, and which suits a remote one?**
stdio for local; Streamable HTTP for remote.

**4. How do you share an MCP server configuration with your whole team in Claude Code?**
Use project scope, which stores it in `.mcp.json` in the repository.

**5. Why is prompt injection a concern with MCP servers?**
Tool results and resources are external content that may contain instructions; untrusted servers can try to manipulate Claude, so use trusted servers and least privilege.
