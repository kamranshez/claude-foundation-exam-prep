# Model Context Protocol (MCP)

MCP is an open standard that lets AI applications connect to external tools, data and prompts through one common protocol, instead of custom integrations for each service.

## 1. Why It Matters

Without MCP, every app needs a bespoke integration for every tool (N x M integrations). With MCP, a service exposes one MCP server and any MCP-capable client (Claude Code, Claude Desktop, your own app) can use it.

## 2. Architecture

| Role | Description |
|------|-------------|
| Host | The AI application the user runs (e.g. Claude Code, Claude Desktop) |
| Client | A connector inside the host; holds a 1:1 connection to one server |
| Server | A program exposing capabilities (GitHub, databases, filesystem, etc.) |

Messages use JSON-RPC 2.0.

## 3. What a Server Can Expose

| Primitive | Controlled by | Purpose | Example |
|-----------|---------------|---------|---------|
| Tools | Model | Actions Claude can call (see [Tool Use](tool-use.md)) | `create_issue` |
| Resources | Application | Read-only data/context | A file, a DB schema |
| Prompts | User | Reusable prompt templates | `/review-pr` |

## 4. Transports

| Transport | Use case |
|-----------|----------|
| stdio | Local server launched as a subprocess on your machine |
| Streamable HTTP | Remote server over the network (supports OAuth) |
| SSE | Older remote transport, being deprecated in favour of HTTP |

## 5. Using MCP in Claude Code

```bash
# Remote HTTP server
claude mcp add --transport http github https://example.com/mcp

# Local stdio server
claude mcp add my-db -- npx -y my-db-mcp-server

claude mcp list
claude mcp remove my-db
```

| Scope | Stored in | Shared with |
|-------|-----------|-------------|
| local (default) | User's project-specific config | Only you, this project |
| project | `.mcp.json` in repo root | Whole team via version control |
| user | User config | You, all projects |

Inside a session, `/mcp` shows server status and handles authentication.

## 6. Security Considerations

- Only install servers you trust: they can run code or return content that tries to manipulate the model (prompt injection).
- Review tool permissions; Claude Code asks for approval before using MCP tools unless allowed in settings.
- Never commit secrets to `.mcp.json`; use environment variable expansion (`${VAR}`).

## Key Takeaways

- MCP standardises how AI apps connect to tools and data (host, client, server).
- Servers expose tools, resources and prompts.
- Use stdio for local servers and HTTP for remote ones.
- Project-scoped servers live in `.mcp.json` and are shared via git.
- Treat third-party servers as untrusted input and limit permissions.

## Practice Questions

1. **What three primitives can an MCP server expose?**
   Tools, resources and prompts.

2. **Which transport suits a local server launched as a subprocess?**
   stdio.

3. **How do you share an MCP server configuration with your whole team in Claude Code?**
   Add it with project scope, which writes to `.mcp.json` in the repo and is committed to version control.

4. **What problem does MCP solve?**
   It replaces many custom point-to-point integrations with one standard protocol, so any compatible client can use any compatible server.

5. **Why should you be cautious with third-party MCP servers?**
   They can execute code or return content containing prompt injection, so only trusted servers should be used and permissions reviewed.
