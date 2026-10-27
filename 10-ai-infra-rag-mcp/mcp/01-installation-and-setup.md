# MCP: Installation and Setup

| | |
|---|---|
| **Cluster** | 10 AI Infrastructure, RAG, MCP & Evals |
| **Role** | Anchor |
| **Prerequisites** | Python 3.12+, `uv` (or `pip`), Node.js 20+ (for the Inspector and TypeScript SDK), Docker |
| **Target version** | MCP spec `2025-06-18`; `mcp` Python SDK 1.12+; `@modelcontextprotocol/sdk` 1.x; Inspector 0.16+ |
| **Capstone phase** | Phase 6 (Months 21-24) |

> After this file you will have the Python SDK installed with `uv`, a minimal FastMCP server running over stdio and over Streamable HTTP, the MCP Inspector connected to it, and the server registered in Claude Code, Claude Desktop, Cursor and VS Code. You will also know how to check which protocol version your SDK speaks.

---

## 1. Toolchain

| Tool | Why | Install |
|---|---|---|
| `uv` | Fast Python package/project manager; `uv run` and `uvx` are the idiomatic way to launch Python MCP servers | https://docs.astral.sh/uv/ |
| Python 3.12+ | SDK requires 3.10+, we standardise on 3.12 | `uv python install 3.12` |
| Node.js 20+ / 22 LTS | Inspector (`npx @modelcontextprotocol/inspector`), TypeScript SDK | https://nodejs.org or `nvm install 22` |
| Docker | Containerise servers for Kubernetes | https://docs.docker.com/engine/install/ |
| `kubectl`, `kind` | Lab 5 deploys servers to a local cluster | see [Kubernetes folder](../../04-containers-and-idp/kubernetes/) |

### Linux / macOS

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
uv --version
uv python install 3.12
node --version   # v22.x
```

### Windows (PowerShell)

```powershell
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"
uv --version
winget install OpenJS.NodeJS.LTS
```

> On Windows, stdio servers work in Claude Desktop and Claude Code, but path quoting in JSON configs needs double backslashes (`C:\\Users\\...`). Prefer `uv --directory <path> run server.py` so the config contains no relative paths.

---

## 2. Create a Server Project

```bash
mkdir -p ~/capstone/ai/mcp-servers/hello-mcp && cd ~/capstone/ai/mcp-servers/hello-mcp
uv init --python 3.12
uv add "mcp[cli]" httpx
```

`mcp[cli]` pulls in the `mcp` command (which provides `mcp dev`, `mcp run`, `mcp install`) plus `typer`. The resulting `pyproject.toml`:

```toml
[project]
name = "hello-mcp"
version = "0.1.0"
requires-python = ">=3.12"
dependencies = [
    "httpx>=0.27",
    "mcp[cli]>=1.12",
]
```

### Check installed versions

```bash
uv run python -c "import mcp, importlib.metadata as m; print(m.version('mcp'))"
uv run python -c "from mcp.types import LATEST_PROTOCOL_VERSION; print(LATEST_PROTOCOL_VERSION)"
uv run mcp version
```

Expected output (your numbers will be newer):

```text
1.12.4
2025-06-18
MCP version 1.12.4
```

> The SDK supports **several** protocol versions and negotiates one during `initialize`. `LATEST_PROTOCOL_VERSION` is what the SDK offers first; `SUPPORTED_PROTOCOL_VERSIONS` lists the fallbacks it accepts.

---

## 3. First Server (stdio)

`server.py`:

```python
"""Minimal FastMCP server exposing one tool, one resource and one prompt."""
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("hello-mcp")


@mcp.tool()
def add(a: int, b: int) -> int:
    """Add two integers and return the sum."""
    return a + b


@mcp.resource("greeting://{name}")
def greeting(name: str) -> str:
    """Return a personalised greeting."""
    return f"Hello, {name}!"


@mcp.prompt()
def review_code(code: str) -> str:
    """Prompt template asking the model to review a code snippet."""
    return f"Please review this code for bugs and security issues:\n\n{code}"


if __name__ == "__main__":
    mcp.run(transport="stdio")
```

Important rules for stdio servers:

- **Never `print()` to stdout.** stdout is the JSON-RPC channel. Use `logging` configured to stderr, or the `Context` logging methods. A stray `print("starting")` produces `Unexpected token 's', "starting" is not valid JSON` in the client.
- The server should not read stdin itself; the SDK owns it.

Run it directly (it will wait silently for JSON-RPC on stdin, which is correct):

```bash
uv run server.py
```

Send a handshake by hand to prove it works. Each message is one line of JSON terminated by `\n`:

```bash
printf '%s\n%s\n%s\n' \
'{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-06-18","capabilities":{},"clientInfo":{"name":"curl-test","version":"0.0.1"}}}' \
'{"jsonrpc":"2.0","method":"notifications/initialized"}' \
'{"jsonrpc":"2.0","id":2,"method":"tools/list"}' \
| uv run server.py
```

Expected output (pretty-printed here, actually one JSON object per line):

```json
{"jsonrpc":"2.0","id":1,"result":{"protocolVersion":"2025-06-18","capabilities":{"experimental":{},"prompts":{"listChanged":false},"resources":{"subscribe":false,"listChanged":false},"tools":{"listChanged":false}},"serverInfo":{"name":"hello-mcp","version":"1.12.4"}}}
{"jsonrpc":"2.0","id":2,"result":{"tools":[{"name":"add","description":"Add two integers and return the sum.","inputSchema":{"properties":{"a":{"title":"A","type":"integer"},"b":{"title":"B","type":"integer"}},"required":["a","b"],"title":"addArguments","type":"object"},"outputSchema":{"properties":{"result":{"title":"Result","type":"integer"}},"required":["result"],"title":"addOutput","type":"object"}}]}}
```

Note that FastMCP generated the JSON Schema from the Python type hints and the description from the docstring, and (since SDK 1.10) an `outputSchema` from the return annotation.

---

## 4. MCP Inspector

The Inspector is a browser UI plus a local proxy that speaks every transport. It is the fastest way to exercise a server without an LLM.

```bash
npx @modelcontextprotocol/inspector uv run server.py
```

Expected output:

```text
Starting MCP inspector...
Proxy server listening on localhost:6277
Session token: <RANDOM_TOKEN>
Open inspector with token pre-filled:
   http://localhost:6274/?MCP_PROXY_AUTH_TOKEN=<RANDOM_TOKEN>
```

In the UI: **Connect**, then **Tools > List Tools > add**, enter `a=2`, `b=3`, **Run Tool**. The result panel shows `5` plus the raw JSON-RPC exchange in the right-hand history pane.

`mcp dev` is a shortcut that does the same with the Python SDK's bundled Inspector launcher:

```bash
uv run mcp dev server.py
```

CLI mode (scriptable, no browser):

```bash
npx @modelcontextprotocol/inspector --cli uv run server.py --method tools/list
npx @modelcontextprotocol/inspector --cli uv run server.py --method tools/call --tool-name add --tool-arg a=2 --tool-arg b=3
```

> Keep the Inspector bound to localhost. It proxies arbitrary commands; the session token exists because of CVE-2025-49596, where an unauthenticated proxy allowed RCE from a malicious web page. Always run the latest version.

---

## 5. Streamable HTTP

Change only the last line:

```python
if __name__ == "__main__":
    mcp.run(transport="streamable-http")
```

Defaults: bind `127.0.0.1:8000`, endpoint path `/mcp`. Override via constructor settings or environment:

```python
mcp = FastMCP("hello-mcp", host="0.0.0.0", port=8000, streamable_http_path="/mcp",
              stateless_http=False, json_response=False)
```

| Setting | Meaning |
|---|---|
| `stateless_http=True` | No `Mcp-Session-Id`; every request independent. Required for horizontally scaled Deployments without sticky sessions |
| `json_response=True` | Reply with plain `application/json` instead of an SSE stream for request/response calls. Simpler for gateways, loses streaming progress |
| `host` | Bind to `0.0.0.0` only in containers; locally stay on `127.0.0.1` to avoid DNS-rebinding exposure |

Run and probe with curl (note the `Accept` header must include both media types and the protocol version header is required after initialize):

```bash
uv run server.py &
curl -s -i http://127.0.0.1:8000/mcp \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-06-18","capabilities":{},"clientInfo":{"name":"curl","version":"0"}}}'
```

Expected output:

```text
HTTP/1.1 200 OK
content-type: text/event-stream
mcp-session-id: 3f1d2c9ab6e04f0c9a2b7e1c5d8f4a21
...

event: message
data: {"jsonrpc":"2.0","id":1,"result":{"protocolVersion":"2025-06-18",...}}
```

Copy the session id and continue:

```bash
SID=3f1d2c9ab6e04f0c9a2b7e1c5d8f4a21
curl -s http://127.0.0.1:8000/mcp -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' -H "Mcp-Session-Id: $SID" \
  -H 'MCP-Protocol-Version: 2025-06-18' \
  -d '{"jsonrpc":"2.0","method":"notifications/initialized"}'
curl -s http://127.0.0.1:8000/mcp -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' -H "Mcp-Session-Id: $SID" \
  -H 'MCP-Protocol-Version: 2025-06-18' \
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"add","arguments":{"a":2,"b":3}}}'
```

Expected final data line:

```json
{"jsonrpc":"2.0","id":2,"result":{"content":[{"type":"text","text":"5"}],"structuredContent":{"result":5},"isError":false}}
```

Inspector against HTTP: choose transport **Streamable HTTP**, URL `http://127.0.0.1:8000/mcp`.

---

## 6. Register the Server in Hosts

### Claude Code

```bash
# local scope (this project, this user) - default
claude mcp add hello -- uv --directory /home/<USER>/capstone/ai/mcp-servers/hello-mcp run server.py
# project scope: writes .mcp.json at repo root, committed for the whole team
claude mcp add --scope project hello -- uv --directory ./ai/mcp-servers/hello-mcp run server.py
# remote Streamable HTTP server
claude mcp add --transport http k8s-mcp https://mcp.example.com/k8s/mcp
# with a header (for example from a secret manager, never hard-coded)
claude mcp add --transport http k8s-mcp https://mcp.example.com/k8s/mcp --header "Authorization: Bearer <TOKEN>"
claude mcp list
claude mcp get hello
claude mcp remove hello
```

The generated `.mcp.json` (project scope):

```json
{
  "mcpServers": {
    "hello": {
      "command": "uv",
      "args": ["--directory", "./ai/mcp-servers/hello-mcp", "run", "server.py"],
      "env": {
        "LOG_LEVEL": "INFO"
      }
    },
    "k8s-mcp": {
      "type": "http",
      "url": "https://mcp.example.com/k8s/mcp"
    }
  }
}
```

Inside a Claude Code session type `/mcp` to see connection status and complete OAuth flows for remote servers.

### Claude Desktop

File locations:

| OS | Path |
|---|---|
| macOS | `~/Library/Application Support/Claude/claude_desktop_config.json` |
| Windows | `%APPDATA%\Claude\claude_desktop_config.json` |

```json
{
  "mcpServers": {
    "hello": {
      "command": "uv",
      "args": ["--directory", "/Users/<USER>/capstone/ai/mcp-servers/hello-mcp", "run", "server.py"]
    }
  }
}
```

Or let the SDK write it: `uv run mcp install server.py --name hello`. Restart Claude Desktop; the tools icon appears under the prompt box. Claude Desktop launches stdio servers only; remote servers are added as "Connectors" in Settings.

### Cursor

`.cursor/mcp.json` (project) or `~/.cursor/mcp.json` (global):

```json
{
  "mcpServers": {
    "hello": { "command": "uv", "args": ["--directory", "./ai/mcp-servers/hello-mcp", "run", "server.py"] },
    "k8s-mcp": { "url": "https://mcp.example.com/k8s/mcp" }
  }
}
```

### VS Code (GitHub Copilot agent mode)

`.vscode/mcp.json` - note the key is `servers`, not `mcpServers`, and inputs let you prompt for secrets instead of committing them:

```json
{
  "inputs": [
    { "id": "prom-url", "type": "promptString", "description": "Prometheus base URL" }
  ],
  "servers": {
    "hello": {
      "type": "stdio",
      "command": "uv",
      "args": ["--directory", "${workspaceFolder}/ai/mcp-servers/hello-mcp", "run", "server.py"],
      "env": { "PROMETHEUS_URL": "${input:prom-url}" }
    },
    "k8s-mcp": { "type": "http", "url": "https://mcp.example.com/k8s/mcp" }
  }
}
```

---

## 7. TypeScript SDK (glimpse)

```bash
mkdir hello-ts && cd hello-ts && npm init -y
npm install @modelcontextprotocol/sdk zod
npm install -D typescript tsx @types/node
```

`server.ts`:

```typescript
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import { z } from "zod";

const server = new McpServer({ name: "hello-ts", version: "0.1.0" });

server.registerTool(
  "add",
  {
    title: "Add",
    description: "Add two integers",
    inputSchema: { a: z.number().int(), b: z.number().int() },
  },
  async ({ a, b }) => ({ content: [{ type: "text", text: String(a + b) }] })
);

const transport = new StdioServerTransport();
await server.connect(transport);
```

Run: `npx tsx server.ts`. Inspector: `npx @modelcontextprotocol/inspector npx tsx server.ts`. The protocol on the wire is identical; only the SDK surface differs.

---

## 8. Verification Checklist

```bash
uv run python -c "import mcp; print('sdk ok')"
uv run mcp version
npx @modelcontextprotocol/inspector --cli uv run server.py --method tools/list | head -c 200
claude mcp list
```

Expected:

```text
sdk ok
MCP version 1.12.4
{"tools":[{"name":"add", ...
hello: uv --directory ... run server.py - ✓ Connected
```

You are ready for [02-core-concepts.md](02-core-concepts.md).
