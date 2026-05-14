---
tags: [enumeration, enumeration/mcp, protocol]
module: attacking_ai_application_and_systems
last_updated: 2026-05-13
source_count: 1
---

# MCP Server Enumeration

Discovering and mapping MCP (Model Context Protocol) servers on a network: port and path discovery, protocol fingerprinting, capability listing, and attack surface documentation.

## Overview

MCP servers expose capabilities over either stdio or HTTP. In network testing only the HTTP transports are reachable remotely:
- **Streamable HTTP** (modern): single endpoint, client POSTs JSON-RPC messages, server can push events via Server-Sent Events.
- **SSE** (legacy): GET request opens an event stream; POST sends messages to a separate endpoint.

There is **no reserved port** for MCP. Servers run on arbitrary ports, typically co-located with web applications or AI service backends. Enumeration has three stages: discovery (find the port/path), fingerprinting (confirm it is MCP and extract server info), and capability mapping (list all tools, resources, and prompts to build an attack surface map).

---

## Discovery

### Port scanning

MCP servers are HTTP services. Scan for open HTTP/HTTPS ports first.

```bash
# Fast scan for common web ports
nmap -sV -p 80,443,3000,4000,5000,7000,8000,8080,8443,8888,9000 <target>

# Broader sweep if time permits
nmap -sV -p 1000-10000 --open <target>
```

### Path discovery

Common MCP server paths (vary by framework):

| Framework / Library | Default path |
|---------------------|-------------|
| fastmcp | `/mcp/` |
| MCP Python SDK | `/mcp/`, `/sse` |
| LangChain / LangGraph | `/mcp`, `/api/mcp` |
| Custom | `/api`, `/tools`, `/rpc` |

Probe candidate paths by sending a `GET` request — an MCP server on the SSE transport responds with `Content-Type: text/event-stream`. A Streamable HTTP server typically returns 405 on GET to the main endpoint (expects POST).

```bash
# Check for SSE transport
curl -v http://target:8000/mcp/ 2>&1 | grep -E "event-stream|Content-Type"

# Check for Streamable HTTP transport (expect 400/405, not 404)
curl -s -o /dev/null -w "%{http_code}" -X POST http://target:8000/mcp/ \
  -H "Content-Type: application/json" -d '{}'
```

A 4xx response (not 404/connection refused) on POST is a strong indicator of an MCP endpoint.

---

## Protocol Fingerprinting

### Initialize handshake

Send a minimal `initialize` JSON-RPC request. A valid MCP server returns its name, version, and capability set.

```bash
curl -s -X POST http://target:8000/mcp/ \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -d '{
    "jsonrpc": "2.0",
    "id": 1,
    "method": "initialize",
    "params": {
      "protocolVersion": "2024-11-05",
      "capabilities": {},
      "clientInfo": {"name": "enum-client", "version": "1.0"}
    }
  }'
```

**What the response reveals:**
- `result.serverInfo.name` — server application name (e.g., `"Inventory MCP Server"`)
- `result.serverInfo.version` — application version (useful for CVE matching)
- `result.capabilities` — what primitive types are available (`tools`, `resources`, `prompts`) and whether any support real-time notifications

Example response indicating all primitives are available:
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "protocolVersion": "2024-11-05",
    "serverInfo": {"name": "warehouse-mcp", "version": "2.1.0"},
    "capabilities": {
      "tools": {"listChanged": false},
      "resources": {"subscribe": false, "listChanged": false},
      "prompts": {"listChanged": false}
    }
  }
}
```

Complete the handshake before calling any other methods:

```bash
curl -s -X POST http://target:8000/mcp/ \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc": "2.0", "method": "notifications/initialized"}'
```

---

## Capability Enumeration

### Python client (recommended)

Use `fastmcp` for structured enumeration. The library handles the handshake, SSE vs HTTP detection, and response parsing automatically.

```python
import asyncio
from fastmcp import Client

TARGET = "http://target:8000/mcp/"

async def enumerate():
    async with Client(TARGET) as client:

        # Server info is available after connect
        print(f"[*] Connected")

        # Tools
        tools = await client.list_tools()
        print(f"\n[+] Tools ({len(tools)}):")
        for t in tools:
            params = list(t.inputSchema.get('properties', {}).keys())
            required = t.inputSchema.get('required', [])
            print(f"  {t.name}({', '.join(params)}) — {t.description}")
            if required:
                print(f"    required: {required}")

        # Resources
        resources = await client.list_resources()
        print(f"\n[+] Resources ({len(resources)}):")
        for r in resources:
            print(f"  {r.uri} — {r.description}")

        # Resource templates (dynamic URIs)
        templates = await client.list_resource_templates()
        print(f"\n[+] Resource templates ({len(templates)}):")
        for rt in templates:
            print(f"  {rt.uriTemplate} — {rt.description}")

        # Prompts
        prompts = await client.list_prompts()
        print(f"\n[+] Prompts ({len(prompts)}):")
        for p in prompts:
            print(f"  {p.name} — {p.description}")

asyncio.run(enumerate())
```

Install:
```bash
pip3 install fastmcp
```

### Tool schema deep-dive

Tool `inputSchema` is a JSON Schema object. Extract parameter types, constraints, and descriptions — these reveal the underlying implementation and data being handled.

```python
async with Client(TARGET) as client:
    tools = await client.list_tools()
    for t in tools:
        schema = t.inputSchema
        print(f"\n=== {t.name} ===")
        print(f"Description: {t.description}")
        for param, props in schema.get('properties', {}).items():
            ptype = props.get('type', 'any')
            desc = props.get('description', '')
            print(f"  {param} ({ptype}): {desc}")
```

Look for:
- Parameters named `command`, `query`, `url`, `path`, `id` — high-value injection targets
- Descriptions mentioning "system", "database", "execute", "fetch" — indicates interaction with external systems
- Parameters that look like they accept URIs or file paths

### Read exposed resources

Static resources and log endpoints can contain sensitive operational data. Read every resource listed:

```python
async with Client(TARGET) as client:
    resources = await client.list_resources()
    for r in resources:
        try:
            content = await client.read_resource(r.uri)
            print(f"\n=== {r.uri} ===")
            for c in content:
                print(c.text if hasattr(c, 'text') else c)
        except Exception as e:
            print(f"  Error reading {r.uri}: {e}")
```

Common high-value resource URIs: `resource://logs`, `resource://config`, `resource://status`

---

## Error-Based Information Extraction

MCP servers frequently return verbose error messages. Probe every tool and resource with malformed inputs to extract:
- API keys in HTTP headers logged to error messages
- Database connection strings
- Internal service names and URLs
- Stack traces revealing framework versions and file paths

```python
async with Client(TARGET) as client:
    tools = await client.list_tools()
    for t in tools:
        params = list(t.inputSchema.get('properties', {}).keys())
        # Send a deliberately invalid value for the first parameter
        if params:
            try:
                result = await client.call_tool(t.name, {params[0]: "' OR 1=1--"})
                print(f"{t.name}: {result}")
            except Exception as e:
                print(f"{t.name} error: {e}")  # errors often contain secrets
```

---

## Automated Scanning

`mcp-scan` runs a suite of checks against an MCP server endpoint, including tool poisoning indicators and common vulnerability patterns.

```bash
pip install mcp-scan
mcp-scan scan http://target:8000/mcp/
```

Output flags:
- `POISONED` — tool description contains suspicious hidden instructions
- `VULN` — common vulnerability detected (SQLi, SSRF, command injection indicators)
- `INFO` — informational findings (verbose errors, exposed credentials)

---

## Attack Surface Map Template

Fill this in for each server discovered. Feed to [[attack/ai/mcp_security]] for exploitation.

```
Target: <host:port/path>
Server name/version: 
Transport: [Streamable HTTP | SSE]
Auth required: [None | API key | OAuth]

Tools:
  <tool_name>
    Params: <list>
    Interesting: [SQLi? SSRF? cmd injection? file path?]
    Notes:

Resources:
  <uri>
    Contents: [logs | config | user data | other]
    Notes:

Resource templates:
  <uriTemplate>
    User-controlled segment: <which part>
    Notes:

Prompts:
  <name>
    Notes:

Key findings:
- [ ] No authentication on MCP endpoint
- [ ] Verbose error messages reveal API keys / connection strings
- [ ] Tool accepts URL parameter (SSRF candidate)
- [ ] Tool accepts command or query parameter (injection candidate)
- [ ] IDOR: resource URI accepts user-controlled ID without authorization
- [ ] Log resource exposed
```

---

## Gotchas & notes

- **No reserved port**: Always include broad port scanning in discovery. MCP servers are most common in the 3000–9000 range but can run anywhere.
- **Origin header**: The MCP spec requires Streamable HTTP servers to validate the `Origin` header to prevent DNS rebinding. Many implementations omit this check — add `Origin: http://localhost` to your curl requests if you get 403.
- **Session IDs on SSE transport**: Some SSE servers assign a `sessionId` on the event stream and require it on POST requests. Capture the initial SSE event before sending messages.
- **Tools/list before initialized**: Some servers require the `notifications/initialized` message before they will respond to capability listing requests. Always complete the handshake.
- **Capabilities vs reality**: A server returning `"capabilities": {}` may still respond to `tools/list` — the capabilities field is informational, not enforced.
- **mcp-scan false negatives**: Automated scanning misses logic-based IDOR and application-specific injection. Always manually probe each tool.

---

## Related pages
- [[protocols/mcp]]
- [[attack/ai/mcp_security]]
- [[attack/ai/prompt_injection]]
- [[attack/ai/insecure_ai_components]]
- [[attack/ai/llm_reconnaissance]]
- [[enumeration/_overview]]

## Sources
- raw/attacking_ai-application_and_systems/mcp_practical_introduction_to_mcp.md
- raw/attacking_ai-application_and_systems/mcp_vulnerable_mcp_servers.md
