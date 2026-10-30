# Model Context Protocol (MCP)

MCP is an open standard that lets AI applications connect to external tools, data and prompts through a common client-server interface.

## 1. Why It Matters

Without a standard, every app needs a custom integration for every data source (an N x M problem). With MCP, a server is written once and works with any MCP-compatible client, such as Claude Code or Claude Desktop. It builds on [tool use](tool-use.md): MCP servers expose tools that Claude calls through the usual tool loop.

## 2. Architecture

| Component | Role |
|-----------|------|
| Host | The AI application the user interacts with (e.g. Claude Code, Claude Desktop) |
| Client | Lives inside the host; keeps a 1:1 connection to one server |
| Server | A program that exposes capabilities (tools, resources, prompts) |

Messages use JSON-RPC 2.0.

## 3. Server Primitives

| Primitive | Controlled by | Description | Example |
|-----------|---------------|-------------|---------|
| Tools | Model | Functions Claude can decide to call | `create_issue`, `run_query` |
| Resources | Application | Read-only data identified by URI | A file, a DB schema |
| Prompts | User | Reusable prompt templates, often surfaced as slash commands | `/review-pr` |

Clients can also offer features to servers, such as sampling (the server asks the host's model for a completion) and roots (which directories the server may access).

## 4. Transports

| Transport | Use |
|-----------|-----|
| stdio | Server runs as a local subprocess; communication over stdin/stdout |
| Streamable HTTP | Remote servers over HTTP; supports OAuth-based authorization |
| SSE (older) | Legacy HTTP transport, being replaced by Streamable HTTP |

For stdio servers, never write logs to stdout, because it carries the protocol messages. Use stderr.

## 5. Using MCP in Claude Code

```bash
# Remote HTTP server
claude mcp add --transport http github https://example.com/mcp

# Local stdio server
claude mcp add my-server -- npx -y some-mcp-server

claude mcp list
claude mcp remove my-server
```

- Scopes: `local` (just you, this project), `project` (shared via `.mcp.json` in the repo), `user` (all your projects).
- Tools appear as `mcp__<server>__<tool>` and are subject to the normal permission rules.
- Project-scoped servers need user approval before first use.
- Inside a session, `/mcp` shows server status and handles authentication.

## 6. Security Considerations

- Only install servers you trust; a server can run code or return malicious content.
- Tool results are untrusted input and can contain prompt injection.
- Use least privilege: limited tokens, scoped permissions, read-only where possible.
- Keep secrets out of `.mcp.json`; use environment variable expansion.
- Require confirmation for destructive actions.

## Key Takeaways

- MCP standardizes how hosts connect to tools, resources and prompts.
- Host contains clients; each client talks to one server over JSON-RPC.
- Tools are model-controlled, resources application-controlled, prompts user-controlled.
- stdio is for local servers, Streamable HTTP for remote ones.
- In Claude Code, add servers with `claude mcp add`; share them with the team via project scope (`.mcp.json`).
- Treat server output as untrusted and apply least privilege.

## Practice Questions

**1. What problem does MCP solve?**
It replaces many custom, one-off integrations with one standard protocol, so any compatible client can use any compatible server.

**2. Name the three server primitives and who controls each.**
Tools (model-controlled), resources (application-controlled), prompts (user-controlled).

**3. When would you choose stdio over HTTP transport?**
For local servers run as a subprocess on the same machine; HTTP is for remote servers.

**4. How do you share an MCP server configuration with your whole team in Claude Code?**
Add it with project scope, which stores it in `.mcp.json` in the repository (users approve it before first use).

**5. Why should a stdio MCP server not print logs to stdout?**
Stdout carries the JSON-RPC protocol messages, so extra output corrupts the stream. Log to stderr.
