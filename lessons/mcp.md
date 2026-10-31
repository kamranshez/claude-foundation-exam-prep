# Model Context Protocol (MCP)

MCP is an open standard that lets AI applications connect to external tools, data and prompts through a common client-server protocol, so one integration works across many apps.

## 1. Why It Matters

Without a standard, every app needs a custom integration for every data source (an N x M problem). With MCP, a service builds one MCP server and any MCP-capable client (Claude Code, Claude Desktop, IDEs, your own agent) can use it.

## 2. Architecture

| Role | Description |
|------|-------------|
| Host | The AI application the user interacts with (e.g. Claude Code, Claude Desktop) |
| Client | A connector inside the host; each client keeps a 1:1 connection to one server |
| Server | A program that exposes capabilities (tools, resources, prompts) |

Messages use JSON-RPC 2.0.

## 3. Server Primitives

| Primitive | Controlled by | Purpose | Example |
|-----------|---------------|---------|---------|
| Tools | Model | Actions Claude can call | `create_issue`, `run_query` |
| Resources | Application | Read-only data/context identified by URI | A file, a database schema |
| Prompts | User | Reusable prompt templates | A `/review-pr` command |

Clients can also offer features to servers, such as sampling (the server asks the host's model for a completion) and roots (which directories the server may access).

## 4. Transports

| Transport | Use case |
|-----------|----------|
| stdio | Local server launched as a subprocess; communicates over stdin/stdout |
| Streamable HTTP | Remote servers over HTTP; supports auth such as OAuth |
| SSE (legacy) | Older remote transport, being replaced by Streamable HTTP |

## 5. Using MCP in Claude Code

```bash
# Add a local stdio server
claude mcp add my-server -- npx -y some-mcp-server

# Add a remote HTTP server
claude mcp add --transport http github https://example.com/mcp

# List / remove
claude mcp list
claude mcp remove my-server
```

| Scope | Stored in | Shared with |
|-------|-----------|-------------|
| local (default) | User's project-specific config | Only you, this project |
| project | `.mcp.json` in the repo | Whole team (committed) |
| user | User config | You, all projects |

MCP tools appear to Claude named `mcp__<server>__<tool>`, and can be allowed or denied in permission settings by that name.

## 6. MCP vs Plain Tool Use

| | Tool use (API) | MCP |
|---|----------------|-----|
| Definition | You write tool schemas in each request | Server advertises tools dynamically |
| Execution | Your app code | The MCP server |
| Reuse | Per application | Across any MCP client |

MCP tools are ultimately passed to Claude as ordinary tools; MCP standardizes how they are discovered and executed.

## 7. Security

- Only install servers you trust; tool descriptions and results can contain prompt injection.
- Grant least privilege and restrict permissions per tool.
- Keep secrets in environment variables, not committed `.mcp.json` values.
- Require human approval for destructive actions.

## Key Takeaways

- MCP standardizes how hosts connect to tools, resources and prompts via servers.
- Host contains clients; each client talks to one server over JSON-RPC.
- Tools are model-controlled, resources application-controlled, prompts user-controlled.
- stdio is for local servers; Streamable HTTP is for remote ones.
- In Claude Code, scopes are local, project (`.mcp.json`) and user.
- Treat MCP servers as untrusted input sources; apply least privilege.

## Practice Questions

1. **What problem does MCP solve?**
   It removes the need for custom integrations between every AI app and every data source by providing one standard protocol.

2. **Name the three server primitives and who controls each.**
   Tools (model-controlled), resources (application-controlled), prompts (user-controlled).

3. **Which transport suits a locally launched server, and which suits a remote one?**
   stdio for local subprocess servers; Streamable HTTP for remote servers.

4. **How do you share an MCP server configuration with your whole team in Claude Code?**
   Use project scope, which stores the config in a committed `.mcp.json` file.

5. **Why is installing a third-party MCP server a security concern?**
   Its tool descriptions and outputs enter Claude's context and may contain prompt injection, and its tools can take real actions, so use only trusted servers with least privilege.
