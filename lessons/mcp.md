# Model Context Protocol (MCP)

MCP is an open standard that lets AI applications connect to external tools, data and prompts through one common protocol, instead of a custom integration for every pair.

## 1. Why It Matters

Without a standard, every app needs a custom connector for every service (N apps x M services). MCP turns this into N + M: a service builds one MCP server, and any MCP-capable client can use it. It builds on [tool use](tool-use.md): MCP servers expose tools that Claude calls through the normal tool use loop.

## 2. Architecture

| Role | Description | Example |
|------|-------------|---------|
| Host | The AI application the user interacts with | Claude Code, Claude Desktop |
| Client | Connector inside the host; keeps one connection per server | Created by the host |
| Server | Program that exposes capabilities | GitHub, filesystem, database server |

Messages use JSON-RPC 2.0. A connection starts with an initialization handshake where client and server negotiate protocol version and capabilities.

## 3. What a Server Can Expose

| Primitive | Controlled by | Purpose | Example |
|-----------|---------------|---------|---------|
| Tools | The model | Actions Claude can call | `create_issue`, `run_query` |
| Resources | The application | Read-only data/context identified by URI | A file, a DB schema |
| Prompts | The user | Reusable templates, often shown as slash commands | `/review-pr` |

Clients can also offer features to servers, such as sampling (the server asks the host's model for a completion) and roots (which directories the server may access).

## 4. Transports

| Transport | Use | Notes |
|-----------|-----|-------|
| stdio | Local server launched as a subprocess | Simple, no network; messages over stdin/stdout |
| Streamable HTTP | Remote server over the network | Supports auth (OAuth) and multiple clients |
| SSE (legacy) | Older remote servers | Deprecated in favor of Streamable HTTP |

## 5. Using MCP in Claude Code

```bash
# Local stdio server
claude mcp add filesystem -- npx -y @modelcontextprotocol/server-filesystem /path/to/dir

# Remote HTTP server
claude mcp add --transport http github https://example.com/mcp

claude mcp list      # show configured servers
claude mcp remove github
```

- Inside a session, `/mcp` shows server status and handles authentication.
- Scopes: `local` (default, just you in this project), `project` (shared via a checked-in `.mcp.json`), `user` (all your projects).
- MCP tools appear to Claude as `mcp__<server>__<tool>` and can be allowed or denied in permission settings.

Example `.mcp.json`:

```json
{
  "mcpServers": {
    "github": {
      "type": "http",
      "url": "https://example.com/mcp"
    }
  }
}
```

## 6. Security

- Only install servers you trust: a server can run code or return content that tries to manipulate Claude (prompt injection).
- Treat tool results as untrusted data.
- Use least privilege: limit directories, scopes and tokens, and pre-approve only needed tools.
- Never hard-code secrets in `.mcp.json`; use environment variable expansion.
- Keep humans in the loop for destructive actions.

## Key Takeaways

- MCP is a standard protocol connecting hosts (clients) to servers that expose tools, resources and prompts.
- Tools are model-controlled, resources are application-controlled, prompts are user-controlled.
- stdio is for local servers; Streamable HTTP is for remote ones.
- In Claude Code, manage servers with `claude mcp`, `/mcp` and a shared `.mcp.json`.
- MCP servers and their outputs are a security boundary: trust, scope and review them.

## Practice Questions

**1. What problem does MCP solve?**
It replaces many custom point-to-point integrations with one standard protocol, so any MCP client can use any MCP server.

**2. Name the three server primitives and who controls each.**
Tools (model), resources (application), prompts (user).

**3. When would you choose stdio over Streamable HTTP?**
For a local server run as a subprocess on the same machine. Use Streamable HTTP for remote, network-accessible servers.

**4. How do you share an MCP server configuration with your whole team in Claude Code?**
Use project scope, which stores it in a `.mcp.json` file committed to the repository.

**5. Why is a third-party MCP server a security concern?**
It can execute code or return malicious content (prompt injection) that influences Claude, so use only trusted servers, least privilege and confirmation for risky actions.
