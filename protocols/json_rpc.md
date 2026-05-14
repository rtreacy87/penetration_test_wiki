---
tags: [protocol, definition, reference]
module:
last_updated: 2026-05-13
source_count: 1
---

# JSON-RPC

Protocol reference for JSON-RPC: a stateless, transport-agnostic remote procedure call protocol encoded in JSON. Widely used in blockchain nodes, internal APIs, AI tool frameworks (MCP), and developer tooling.

## Overview

JSON-RPC is a lightweight RPC protocol that encodes procedure calls and responses as JSON objects. It is:

- **Stateless** — each request is independent; the server holds no per-request session state.
- **Transport-agnostic** — runs over HTTP, HTTPS, WebSocket, stdio, Unix sockets, TCP, or any stream.
- **Bidirectional** (over WebSocket/stdio) — server can push notifications to the client without a prior request.

Two major versions are deployed in practice:

| Feature | JSON-RPC 1.0 | JSON-RPC 2.0 |
|---------|-------------|-------------|
| `jsonrpc` field | Absent | `"2.0"` (required) |
| Notifications | `id: null` | No `id` field |
| Named parameters | Not supported | Supported (`params` as object) |
| Batch requests | Not supported | Supported (array of requests) |
| Error objects | Basic | Structured (`code`, `message`, `data`) |

JSON-RPC 2.0 is the current standard. Deployments of 1.0 are legacy (older Ethereum clients, some Bitcoin implementations).

---

## Message Types

### Request

Initiates a remote procedure call. The server must return a Response with the same `id`.

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "eth_blockNumber",
  "params": []
}
```

- `jsonrpc` — always `"2.0"` in version 2
- `id` — client-chosen unique identifier; integer, string, or `null` (though `null` is discouraged in 2.0 — it conflicts with notification semantics). Echoed verbatim in the response.
- `method` — string naming the procedure to invoke
- `params` — optional; either an **array** (positional) or an **object** (named)

Named params:
```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "method": "tools/call",
  "params": {
    "name": "execute_command",
    "arguments": {"command": "id"}
  }
}
```

### Response — Success

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": "0x10d4f"
}
```

- `id` — matches the originating request
- `result` — procedure's return value; any JSON type (object, array, string, number, boolean, null)

### Response — Error

```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "error": {
    "code": -32602,
    "message": "Invalid params",
    "data": "expected 'command' parameter of type string"
  }
}
```

- `error.code` — integer (see error code table below)
- `error.message` — short human-readable description
- `error.data` — optional; arbitrary additional detail. **Frequently leaks internal state** (stack traces, file paths, credentials in connection strings).

A response contains `result` or `error`, never both.

### Notification

A request that expects no response. Identified by the **absence of an `id` field**.

```json
{
  "jsonrpc": "2.0",
  "method": "notifications/initialized"
}
```

Servers must not reply to notifications. Used for fire-and-forget events and protocol lifecycle signals (e.g., MCP's `notifications/initialized`).

### Batch Request

An array of request objects sent in a single transport call. The server processes all items (order not guaranteed) and returns an array of responses — one per non-notification item.

```json
[
  {"jsonrpc": "2.0", "id": 1, "method": "tools/list"},
  {"jsonrpc": "2.0", "id": 2, "method": "resources/list"},
  {"jsonrpc": "2.0", "method": "notifications/ping"}
]
```

Response (no response for the notification):
```json
[
  {"jsonrpc": "2.0", "id": 1, "result": {"tools": [...]}},
  {"jsonrpc": "2.0", "id": 2, "result": {"resources": [...]}}
]
```

Batch support is optional — servers that don't implement it return a single error response.

---

## Standard Error Codes

| Code | Name | Meaning |
|------|------|---------|
| `-32700` | Parse error | Invalid JSON received |
| `-32600` | Invalid Request | JSON is valid but not a valid Request object |
| `-32601` | Method not found | Method does not exist on the server |
| `-32602` | Invalid params | Invalid method parameters |
| `-32603` | Internal error | Internal server error |
| `-32000` to `-32099` | Server error | Implementation-defined server errors |

Codes outside this range are application-defined. The `data` field on any error is application-defined and unvalidated — prime location for information leakage.

---

## Transport Mechanisms

### HTTP / HTTPS (most common for remote services)

- Client sends requests as **HTTP POST** with `Content-Type: application/json`.
- Response returned in the HTTP body.
- Stateless — each POST is an independent RPC session.
- No persistent connection; notifications from server to client are not possible without long-polling or SSE.

```bash
curl -s -X POST http://target:8545 \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"eth_accounts","params":[]}'
```

### WebSocket

- Persistent bidirectional connection.
- Server can push notifications without a prior client request.
- Common in Ethereum nodes (`ws://target:8546`), language servers (LSP), and some internal APIs.

```bash
# Probe with wscat
wscat -c ws://target:8546
> {"jsonrpc":"2.0","id":1,"method":"eth_blockNumber","params":[]}
```

### stdio

- Client and server communicate over standard input/output.
- Requires both processes to run on the same host.
- Used by MCP servers (local transport), language servers (LSP), and development tooling.
- Not reachable from the network; relevant post-exploitation.

### Unix socket / TCP socket

- Raw socket connection, less common.
- Bitcoin Core uses a Unix socket by default (`~/.bitcoin/.cookie` for auth).

---

## Where JSON-RPC Appears

| System | Default port / path | Notes |
|--------|-------------------|-------|
| **MCP servers** | `:8000/mcp/` (varies) | All MCP communication is JSON-RPC 2.0 |
| **Ethereum nodes** (Geth, Besu) | HTTP `:8545`, WS `:8546` | Often unauthenticated on internal networks |
| **Bitcoin Core** | HTTP `:8332` (mainnet) | Requires HTTP Basic Auth (rpcuser/rpcpassword) |
| **Language Server Protocol** (LSP) | stdio | Used by IDEs; reachable via process injection |
| **Kubernetes API** (some extensions) | HTTPS | Mixed REST and JSON-RPC |
| **Internal microservices** | Varies | Developers often expose JSON-RPC without auth on internal networks |
| **Jupyter / Jupyter Lab** | HTTP | Kernel communication protocol |

---

## Pentester Relevance

### Method enumeration

There is no standard for discovering available methods. Common approaches:
- Try well-known methods for the suspected service (`eth_accounts`, `tools/list`, `rpc.modules`)
- Send an invalid method and check if the `-32601` error leaks a method list in `data`
- Some implementations expose a `system.listMethods` or `rpc.discover` method

```bash
# Probe for method list disclosure
curl -s -X POST http://target:8545 \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"rpc.modules","params":[]}'
```

### Error-based information extraction

The `data` field in error responses is unspecified and frequently contains stack traces, file paths, connection strings, and API keys. Deliberately provoke errors with malformed or out-of-range parameters on every method discovered.

### Missing authentication

JSON-RPC has no built-in authentication mechanism. Services relying solely on network-level controls (firewall, localhost binding) are vulnerable when those controls fail:
- SSRF from another service that can reach the RPC port
- DNS rebinding (if the server doesn't validate `Origin` or `Host`)
- Direct access on misconfigured internal networks

Ethereum nodes exposed without auth (`--http.addr 0.0.0.0 --http.corsdomain "*"`) allow full account enumeration and transaction signing.

### CSRF over HTTP

JSON-RPC over HTTP POST is not protected against CSRF by default. If the server:
- Accepts requests with `Content-Type: text/plain` or no content type, and
- Has no CSRF token or `Origin` check

then a malicious web page can issue cross-origin JSON-RPC calls against a locally bound server (e.g., `http://127.0.0.1:8545`) from a victim's browser.

### Batch request abuse

Batch requests allow sending many RPC calls in a single HTTP request. Useful for:
- Parallel enumeration of methods without triggering per-request rate limits
- Racing concurrent requests (e.g., multiple `eth_sendTransaction` calls)

### No method-level authorization

Many JSON-RPC implementations apply authentication at the transport level (a single API key grants full access). If you obtain any valid credential, you can call all available methods — including privileged ones like `personal_unlockAccount` on Ethereum or `tools/call` with arbitrary parameters on MCP.

---

## Raw Request Crafting

```bash
# Generic JSON-RPC probe
curl -s -X POST http://target:PORT/PATH \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"METHOD","params":PARAMS}'

# WebSocket (requires wscat: npm install -g wscat)
wscat -c ws://target:PORT/PATH
> {"jsonrpc":"2.0","id":1,"method":"METHOD","params":[]}

# With Basic Auth (Bitcoin-style)
curl -s --user rpcuser:rpcpass -X POST http://target:8332 \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"1.0","id":"test","method":"getblockchaininfo","params":[]}'
```

---

## Related pages
- [[protocols/mcp]]
- [[enumeration/mcp_servers]]
- [[attack/ai/mcp_security]]
- [[attack/ai/insecure_ai_components]]

## Sources
- raw/attacking_ai-application_and_systems/mcp_introduction_to_mcp.md
