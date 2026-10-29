# Model Context Protocol (MCP)

MCP is an open protocol that lets AI applications such as Claude connect to external tools, data and prompts through one standard interface.

## 1. Why It Matters

Without a standard, every app needs a custom integration for every tool (an N x M problem). With MCP, a tool is wrapped once as an **MCP server**, and any **MCP client** can use it. It builds directly on [tool use](tool-use.md): MCP tools are exposed to Claude as ordinary tool definitions.

## 2. Architecture

| Component | Role | Example |
|-----------|------|---------|
| Host | The AI application the user interacts with | Claude Code, Claude Desktop |
| Client | Connector inside the host; keeps one connection per server | Created by the host |
| Server | Program exposing capabilities over MCP | GitHub, filesystem, database server |

Messages use JSON-RPC 2.0. The client and server negotiate capabilities when the connection starts.

## 3. What a Server Can Expose

| Primitive | Controlled by | Purpose | Example |
|-----------|--------------|---------|---------|
| Tools | The model | Actions Claude can decide to call | `create_issue`, `run_query` |
| Resources | The application | Read-only data and context | A file, a database schema |
| Prompts | The user | Reusable prompt templates | A `/review-code` command |

Tools are the most commonly used primitive. Clients may also offer features back to servers, such as sampling (asking the host's model for a completion) and roots (the directories a server may work in).

## 4. Transports

| Transport | Use for |
|-----------|---------|
| stdio | Local servers started as a subprocess on your machine |
| Streamable HTTP | Remote servers reached over the network (supports auth such as OAuth) |

Older SSE-based remote transport is deprecated in favor of Streamable HTTP.

## 5. Using MCP in Claude Code

```bash
# Add a local stdio server
claude mcp add my-server -- npx -y @example/mcp-server

# Add a remote HTTP server
claude mcp add --transport http my-remote https://example.com/mcp

claude mcp list      # show configured servers
claude mcp remove my-server
```

| Topic | Detail |
|-------|--------|
| Scopes | `local` (default, this project and you), `project` (shared via `.mcp.json` in version control), `user` (all your projects) |
| Tool naming | MCP tools appear as `mcp__<server>__<tool>` |
| Permissions | Allow or deny MCP tools by that name in settings or `--allowed-tools` |
| Project config | `.mcp.json` at the repo root; Claude Code asks for approval before using project servers |
| In session | `/mcp` shows server status and handles authentication |

Example `.mcp.json`:

```json
{
  "mcpServers": {
    "db": {
      "command": "npx",
      "args": ["-y", "@example/db-mcp"],
      "env": {"DB_URL": "${DB_URL}"}
    }
  }
}
```

Use environment variable expansion for secrets instead of hard-coding them.

## 6. Security Considerations

- Only install servers you trust: a server runs code and its tool output enters Claude's context.
- Tool results and descriptions can contain **prompt injection**; treat them as untrusted data.
- Apply least privilege: scope tokens narrowly and allow only the tools needed.
- Keep credentials in environment variables or secret stores, never in committed config.
- Require confirmation for destructive actions.
- Many tool definitions consume context tokens, so enable only the servers you need.

## Key Takeaways

- MCP standardizes how hosts connect to external tools and data: host, client, server.
- Servers expose tools (model-controlled), resources (app-controlled) and prompts (user-controlled).
- Use stdio for local servers and Streamable HTTP for remote ones.
- In Claude Code, add servers with `claude mcp add`; share team servers through `.mcp.json` (project scope).
- MCP tools are named `mcp__<server>__<tool>` and are governed by the normal permission system.
- Treat third-party servers and their outputs as untrusted.

## Practice Questions

**1. What problem does MCP solve?**
It replaces many custom point-to-point integrations with one open standard, so any MCP client can use any MCP server.

**2. Name the three server primitives and who controls each.**
Tools (model-controlled), resources (application-controlled) and prompts (user-controlled).

**3. When would you choose stdio over Streamable HTTP?**
For a local server run as a subprocess on the same machine. Streamable HTTP is for remote servers over the network.

**4. How do you share an MCP server with your whole team in Claude Code?**
Add it with project scope so it is stored in `.mcp.json` and committed to version control. Members approve it before use.

**5. What is the main security risk of connecting a third-party MCP server?**
Untrusted code and content: the server can run with your privileges and its tool output can carry prompt injection. Mitigate with trusted sources, least-privilege credentials and restricted allowed tools.
