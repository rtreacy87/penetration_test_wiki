---
tags: [protocol, definition, attack/ai]
module: attacking_ai_application_and_systems
last_updated: 2026-05-13
source_count: 2
---

# MCP — Model Context Protocol

Protocol reference for MCP (Model Context Protocol): architecture, primitives, message format, transport mechanisms, and lifecycle. For attack and enumeration techniques see the linked pages.

## Overview

MCP, introduced by Anthropic in 2024, standardizes how LLM applications connect to external tools, data sources, and APIs. Analogous to USB — before USB, peripherals required custom drivers and ports; before MCP, each LLM integration required a custom API consumer. MCP provides a single protocol that the LLM host speaks to all integrations.

MCP is young technology. As of 2024–2025 it has no built-in authentication, no mandatory encryption, and minimal input validation requirements. This makes it a high-priority attack surface on any engagement involving AI-powered applications.

---

## Architecture

Three components:

| Component | Role |
|-----------|------|
| **Host** | Container and coordinator. Manages one or more MCP client instances. Handles LLM integration. One host can manage multiple clients. |
| **Client** | Created by the host. Connects to exactly one MCP server. Handles all MCP protocol communication with that server. Lives inside the host process. |
| **Server** | Exposes capabilities (prompts, resources, tools) to connecting clients. Operates independently of the LLM. Can run locally (stdio) or remotely (HTTP). |

```
Host Process
├── LLM Integration
├── MCP Client A ──────► MCP Server 1 (files, git)
├── MCP Client B ──────► MCP Server 2 (database)
└── MCP Client C ──────► MCP Server 3 (external API)
```

A user can connect to multiple MCP servers simultaneously through the same host — each gets its own client instance.

---

## Server Primitives (Capabilities)

MCP servers expose capabilities via three primitive types:

| Primitive | Controlled by | Nature | Description |
|-----------|--------------|--------|-------------|
| **Prompts** | User | Read-only | Pre-defined templates that guide LLM interactions. User selects them explicitly. Analogous to slash commands. |
| **Resources** | Application | Read-only | Structured data identified by URI. Application enriches the LLM prompt with resource content. Supports static URIs and URI templates (parameterized). |
| **Tools** | Model | Executable | Functions the LLM invokes autonomously based on context. Not necessarily read-only — tools frequently have side effects (file writes, API calls, shell commands). |

Clients can also offer capabilities to servers (`roots` for filesystem path sharing, `sampling` for LLM generation requests), but server-side capabilities are the primary attack surface.

---

## Transport Mechanisms

### stdio

- Client and server communicate over standard input/output.
- Requires client and server to run on the **same local machine**.
- Used for local tool integrations (e.g., Claude Desktop launching a local MCP process).
- Not reachable from the network — relevant only in post-exploitation or insider-threat scenarios.

### Streamable HTTP

- MCP server exposes an **HTTP endpoint** (commonly `/mcp/`).
- Client sends JSON-RPC messages via **HTTP POST**.
- Server can push messages to the client via **Server-Sent Events (SSE)** on the same or a separate endpoint.
- Reachable over the network — the primary target for remote MCP exploitation.

The spec requires Streamable HTTP servers to validate the `Origin` header to prevent DNS rebinding attacks. Many implementations omit this check.

### Legacy SSE transport (deprecated)

An earlier transport where the client established a persistent SSE stream via GET, and the server sent a `endpoint` event containing a URL for the client to POST messages to. Still found in older deployments. Identifiable by the server returning `Content-Type: text/event-stream` on a GET to the MCP path.

---

## Message Format (JSON-RPC 2.0)

All MCP messages follow JSON-RPC 2.0. Three message types:

### Request

Initiates an operation. Expects a response.

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/list",
  "params": {}
}
```

- `id` — unique request identifier (integer or string); echoed in the response
- `method` — the operation to perform
- `params` — optional; operation-specific parameters

### Response

Result of a previous request. Carries either `result` or `error`, never both.

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "tools": [
      {
        "name": "store_file",
        "description": "Store a file.",
        "inputSchema": {
          "type": "object",
          "properties": {
            "file_content": {"title": "File Content", "type": "string"},
            "file_name": {"title": "File Name", "type": "string"}
          },
          "required": ["file_content", "file_name"]
        }
      }
    ]
  }
}
```

Error response:
```json
{
  "jsonrpc": "2.0",
  "id": 8,
  "error": {
    "code": 0,
    "message": "Error creating resource: [Errno 2] No such file or directory: '/tmp/invalid.mcpfile'"
  }
}
```

### Notification

One-way message. No `id`, no response.

```json
{
  "jsonrpc": "2.0",
  "method": "notifications/initialized"
}
```

---

## Protocol Lifecycle

### Phase 1 — Initialization

Three messages, always in this order:

**1. Client → Server: `initialize` request**

```json
{
  "jsonrpc": "2.0",
  "id": 0,
  "method": "initialize",
  "params": {
    "protocolVersion": "2024-11-05",
    "capabilities": {
      "sampling": {},
      "roots": {"listChanged": true}
    },
    "clientInfo": {"name": "mcp", "version": "0.1.0"}
  }
}
```

**2. Server → Client: `initialize` response**

```json
{
  "jsonrpc": "2.0",
  "id": 0,
  "result": {
    "protocolVersion": "2024-11-05",
    "capabilities": {
      "prompts":   {"listChanged": false},
      "resources": {"subscribe": false, "listChanged": false},
      "tools":     {"listChanged": false}
    },
    "serverInfo": {"name": "warehouse-mcp", "version": "2.1.0"}
  }
}
```

The server's `serverInfo` leaks application name and version. Capabilities tell you which primitives are available.

**3. Client → Server: `notifications/initialized`**

```json
{
  "jsonrpc": "2.0",
  "method": "notifications/initialized"
}
```

This notification must be sent before the operation phase begins. Some servers will not respond to capability listing requests without it.

### Phase 2 — Operation

Normal request/response cycles using these methods:

| Method | Direction | Description |
|--------|-----------|-------------|
| `prompts/list` | C→S | List available prompt templates |
| `prompts/get` | C→S | Retrieve a specific prompt with arguments |
| `resources/list` | C→S | List static resources |
| `resources/templates/list` | C→S | List URI templates (parameterized resources) |
| `resources/read` | C→S | Read a resource by URI |
| `tools/list` | C→S | List available tools with input schemas |
| `tools/call` | C→S | Invoke a tool with arguments |

Example `tools/call` request:
```json
{
  "jsonrpc": "2.0",
  "id": 5,
  "method": "tools/call",
  "params": {
    "name": "execute_server_command",
    "arguments": {"command": "date;id"}
  }
}
```

Example `resources/read` request:
```json
{
  "jsonrpc": "2.0",
  "id": 3,
  "method": "resources/read",
  "params": {"uri": "price://banana'--"}
}
```

### Phase 3 — Shutdown

No dedicated shutdown message in the MCP spec. Session ends when the transport closes:
- stdio: stream is closed
- Streamable HTTP: HTTP connection is closed

---

## Security Properties (Protocol-Level)

| Property | Status |
|----------|--------|
| Authentication | **Not specified.** Must be implemented on top (API key, OAuth). Most deployments omit it. |
| Encryption | **Not mandated.** HTTP transport is plaintext by default. TLS requires explicit server configuration. |
| Input validation | **Not mandated.** Tool and resource parameters are passed through; validation is left to the implementation. |
| Origin validation | **Required by spec** for Streamable HTTP (DNS rebinding prevention). Widely omitted in practice. |
| Tool description integrity | **None.** Tool descriptions are controlled entirely by the server — a malicious server injects prompt payloads via this field. |

---

## Pentester's Interest

| Finding | What it enables |
|---------|----------------|
| Unauthenticated MCP endpoint | Direct tool/resource invocation without LLM involvement |
| Verbose error messages | API key and credential extraction from error responses |
| Tools accepting `command`, `query`, `url`, `path` params | Command injection, SQLi, SSRF |
| Resource URIs with user-controlled segments | IDOR, path traversal |
| No Origin header validation | DNS rebinding attack to reach localhost-bound servers |
| Multiple clients connected | Tool shadowing across servers |
| Exposed log resources | Credential and PII extraction from `resource://logs` |

---

## Related pages
- [[protocols/json_rpc]]
- [[enumeration/mcp_servers]]
- [[attack/ai/mcp_security]]
- [[attack/ai/prompt_injection]]
- [[attack/ai/attacking_ai_systems]]
- [[definitions/owasp_llm_top10]]

## Sources
- raw/attacking_ai-application_and_systems/mcp_introduction_to_mcp.md
- raw/attacking_ai-application_and_systems/mcp_practical_introduction_to_mcp.md
