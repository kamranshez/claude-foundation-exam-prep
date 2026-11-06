# Model Context Protocol (MCP)

MCP is an open standard that lets AI applications connect to external tools, data and prompts through one common protocol, instead of custom integrations for each service.

## 1. Why It Matters

Without MCP, every app needs its own connector for every tool (an N x M problem). With MCP, a service exposes one MCP server and any MCP-capable client (Claude Code, Claude Desktop, your own agent) can use it.

## 2. Architecture

| Role | Description |
|------|-------------|
| Host | The AI application the user interacts with (e.g. Claude Code) |
| Client | Connector inside the host; keeps a 1:1 connection to one server |
| Server | Program that exposes capabilities from a data source or service |

Messages use JSON-RPC 2.0.

## 3. What Servers Expose (Primitives)

| Primitive | Controlled by | Purpose | Example |
|-----------|---------------|---------|---------|
| Tools | Model | Actions Claude can call | `create_issue`, `run_query` |
| Resources | Application | Read-only data/context | File contents, DB schema |
| Prompts | User | Reusable prompt templates | `/summarize-pr` |

Clients can also offer features to servers, such as sampling (server asks the client's model for a completion) and roots (which directories the server may access).

## 4. Transports

| Transport | Use |
|-----------|-----|
| stdio | Server runs as a local subprocess; best for local tools |
| Streamable HTTP | Remote servers over HTTP; supports auth such as OAuth |

An older HTTP+SSE transport exists but is superseded by Streamable HTTP.

## 5. Using MCP in Claude Code

```bash
# Add a remote HTTP server
claude mcp add --transport http github https://example.com/mcp

# Add a local stdio server
claude mcp add my-server -- npx -y some-mcp-server

claude mcp list
claude mcp remove my-server
```

- Scopes: `local` (just you, this project), `project` (shared via `.mcp.json`), `user` (all your projects).
- MCP tools appear namespaced, e.g. `mcp__server__tool`, and are subject to the permission system.
- `/mcp` shows server status and handles authentication.

## 6. MCP vs Plain Tool Use

| | Tool use (API) | MCP |
|---|----------------|-----|
| Definition | Tools written in your own request | Tools discovered from a server |
| Reuse | Per application | Shared across any MCP client |
| Execution | Your code | The MCP server |

MCP tools are ultimately presented to Claude as ordinary tools.

## 7. Security

- Only install servers you trust; tool descriptions and results can contain prompt injection.
- Grant least privilege and review permissions before approving tool calls.
- Keep secrets in environment variables, not in committed config.
- Require confirmation for destructive actions.

## Key Takeaways

- MCP standardizes how AI apps connect to tools and data (host, client, server).
- Servers expose tools, resources and prompts; tools are model-controlled.
- Transports: stdio for local, Streamable HTTP for remote.
- Claude Code manages servers with `claude mcp add/list/remove`; scopes are local, project, user.
- Treat third-party servers as untrusted input sources.

## Practice Questions

1. **Which MCP primitive is model-controlled, and what is it for?**
   Tools: actions the model can invoke, such as querying a database.

2. **When would you choose stdio over Streamable HTTP?**
   For local servers running as a subprocess on the user's machine; HTTP is for remote servers.

3. **How do you share an MCP server configuration with your whole team in Claude Code?**
   Use project scope, which stores the config in `.mcp.json` in the repo.

4. **What is the main security risk of connecting an untrusted MCP server?**
   Prompt injection through tool descriptions or results, plus over-broad access to data and actions.

5. **How does MCP differ from defining tools directly in an API request?**
   MCP tools are discovered from a reusable server shared across clients; API tools are defined per request and executed by your own code.
