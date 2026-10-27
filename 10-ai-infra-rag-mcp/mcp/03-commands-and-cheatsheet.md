# MCP: Commands and Cheatsheet

| | |
|---|---|
| **Cluster** | 10 AI Infrastructure, RAG, MCP & Evals |
| **Role** | Anchor |
| **Prerequisites** | [02-core-concepts.md](02-core-concepts.md) |
| **Target version** | MCP spec `2025-06-18`; `mcp` Python SDK 1.12+; `@modelcontextprotocol/sdk` 1.x; `langchain-mcp-adapters` 0.1+; `strands-agents` 1.x |
| **Capstone phase** | Phase 6 (Months 21-24) |

> A task-organised reference: every protocol method with its params and result shape, the Python SDK server and client API, the TypeScript SDK equivalents, host configuration files, Inspector and Claude Code CLI commands, framework adapters (LangChain, LangGraph, Strands), and hand-crafted JSON-RPC snippets for debugging with `curl`.

---

## 1. Protocol Method Reference

Direction: C->S = client calls server; S->C = server calls client.

| Method | Dir | Params | Result | Needs capability |
|---|---|---|---|---|
| `initialize` | C->S | `protocolVersion`, `capabilities`, `clientInfo{name,version}` | `protocolVersion`, `capabilities`, `serverInfo`, `instructions?` | - |
| `notifications/initialized` | C->S | - | (notification) | - |
| `ping` | both | - | `{}` | - |
| `tools/list` | C->S | `cursor?` | `tools[]`, `nextCursor?` | `tools` |
| `tools/call` | C->S | `name`, `arguments` | `content[]`, `structuredContent?`, `isError?` | `tools` |
| `notifications/tools/list_changed` | S->C | - | - | `tools.listChanged` |
| `resources/list` | C->S | `cursor?` | `resources[]`, `nextCursor?` | `resources` |
| `resources/templates/list` | C->S | `cursor?` | `resourceTemplates[]` | `resources` |
| `resources/read` | C->S | `uri` | `contents[]{uri,mimeType,text\|blob}` | `resources` |
| `resources/subscribe` / `unsubscribe` | C->S | `uri` | `{}` | `resources.subscribe` |
| `notifications/resources/updated` | S->C | `uri` | - | `resources.subscribe` |
| `notifications/resources/list_changed` | S->C | - | - | `resources.listChanged` |
| `prompts/list` | C->S | `cursor?` | `prompts[]` | `prompts` |
| `prompts/get` | C->S | `name`, `arguments?` | `description?`, `messages[]` | `prompts` |
| `notifications/prompts/list_changed` | S->C | - | - | `prompts.listChanged` |
| `completion/complete` | C->S | `ref{type,name\|uri}`, `argument{name,value}`, `context?` | `completion{values[],total?,hasMore?}` | `completions` |
| `logging/setLevel` | C->S | `level` | `{}` | `logging` |
| `notifications/message` | S->C | `level`, `logger?`, `data` | - | `logging` |
| `sampling/createMessage` | S->C | `messages[]`, `modelPreferences?`, `systemPrompt?`, `maxTokens`, `temperature?`, `stopSequences?` | `role`, `content`, `model`, `stopReason?` | client `sampling` |
| `roots/list` | S->C | - | `roots[]{uri,name?}` | client `roots` |
| `notifications/roots/list_changed` | C->S | - | - | client `roots.listChanged` |
| `elicitation/create` | S->C | `message`, `requestedSchema` | `action`, `content?` | client `elicitation` |
| `notifications/progress` | both | `progressToken`, `progress`, `total?`, `message?` | - | - |
| `notifications/cancelled` | both | `requestId`, `reason?` | - | - |

### Capability objects

```json
// client
{"roots":{"listChanged":true},"sampling":{},"elicitation":{},"experimental":{}}
// server
{"tools":{"listChanged":true},"resources":{"subscribe":true,"listChanged":true},"prompts":{"listChanged":true},"logging":{},"completions":{},"experimental":{}}
```

### Content block types

| `type` | Fields | Used in |
|---|---|---|
| `text` | `text` | tool results, prompts, sampling |
| `image` | `data` (base64), `mimeType` | tool results (Grafana panel PNG) |
| `audio` | `data`, `mimeType` | rarely |
| `resource_link` | `uri`, `name`, `description?`, `mimeType?` | tool result pointing at a resource to read later |
| `resource` | `resource{uri,mimeType,text\|blob}` | embedded resource in tool result or prompt |

Every content block may carry `annotations{audience:["user"|"assistant"],priority:0..1,lastModified}`.

---

## 2. Python SDK: Server API

### Construction

```python
from mcp.server.fastmcp import FastMCP, Context
from mcp.types import ToolAnnotations

mcp = FastMCP(
    name="k8s-mcp",
    instructions="Kubernetes read access. rollout_restart needs approval.",
    host="127.0.0.1",            # 0.0.0.0 in containers
    port=8000,
    streamable_http_path="/mcp",
    stateless_http=False,
    json_response=False,
    lifespan=lifespan,           # async context manager yielding shared state
    # auth: token_verifier=..., auth=AuthSettings(...)   see 05-production-patterns.md
)
```

Settings may also come from env vars prefixed `FASTMCP_` (for example `FASTMCP_PORT=9000`, `FASTMCP_LOG_LEVEL=DEBUG`).

### Decorators

```python
@mcp.tool()                                   # name = function name, description = docstring
@mcp.tool(name="list_pods", title="List pods", description="...",
          annotations=ToolAnnotations(readOnlyHint=True, destructiveHint=False,
                                      idempotentHint=True, openWorldHint=False),
          structured_output=True)              # force/disable outputSchema generation
@mcp.resource("prom://targets", name="targets", description="...", mime_type="application/json")
@mcp.resource("k8s://{namespace}/pods/{pod}/logs")   # template; params become function params
@mcp.prompt(name="triage_alert", title="Triage an alert", description="...")
@mcp.completion()                              # (ref, argument, context) -> Completion | None
```

Return types and how they are serialised:

| Return annotation | `content` | `structuredContent` |
|---|---|---|
| `str` | text as-is | none (unless `structured_output=True`, then `{"result": ...}`) |
| `int`, `float`, `bool` | `str(value)` | `{"result": value}` |
| `dict[str, Any]` / `TypedDict` | JSON text | the dict |
| Pydantic `BaseModel` / dataclass | JSON text | `model_dump()` and `outputSchema` from the model |
| `list[...]` | JSON text | `{"result": [...]}` |
| `mcp.server.fastmcp.Image(path=..)` / `Image(data=.., format="png")` | `image` block | none |
| `list[types.TextContent \| types.ImageContent \| ...]` | passed through | none |
| `None` | empty | none |
| raise `Exception` | error text, `isError: true` | none |

### `Context` methods

```python
async def tool(arg: str, ctx: Context) -> str:
    await ctx.debug("msg"); await ctx.info("msg"); await ctx.warning("msg"); await ctx.error("msg")
    await ctx.log(level="info", message="msg", logger_name="k8s-mcp")
    await ctx.report_progress(progress=1, total=3, message="step 1")
    contents = await ctx.read_resource("prom://targets")
    result = await ctx.elicit(message="Confirm?", schema=ConfirmModel)   # result.action, result.data
    msg = await ctx.session.create_message(messages=[...], max_tokens=200)  # sampling
    ctx.request_id; ctx.client_id
    ctx.request_context.lifespan_context      # object yielded by lifespan
    ctx.request_context.meta                  # _meta from the request (progressToken etc.)
```

### Running

```python
mcp.run()                                   # stdio by default
mcp.run(transport="stdio")
mcp.run(transport="streamable-http")        # uvicorn on host:port
mcp.run(transport="sse")                    # legacy

# Mount into your own ASGI app (FastAPI/Starlette) for health probes, metrics, auth middleware:
from starlette.applications import Starlette
from starlette.routing import Mount
app = Starlette(routes=[Mount("/", app=mcp.streamable_http_app())], lifespan=lambda a: mcp.session_manager.run())
# uvicorn server:app --host 0.0.0.0 --port 8000
```

### Low-level server (when you need full control)

```python
import mcp.types as types
from mcp.server.lowlevel import Server
from mcp.server.stdio import stdio_server

server = Server("lowlevel-example")

@server.list_tools()
async def list_tools() -> list[types.Tool]:
    return [types.Tool(name="echo", description="Echo", inputSchema={"type": "object", "properties": {"s": {"type": "string"}}, "required": ["s"]})]

@server.call_tool()
async def call_tool(name: str, arguments: dict) -> list[types.TextContent]:
    if name != "echo":
        raise ValueError(f"unknown tool {name}")
    return [types.TextContent(type="text", text=arguments["s"])]

async def main() -> None:
    async with stdio_server() as (read, write):
        await server.run(read, write, server.create_initialization_options())
```

### Dynamic tool list changes

```python
# FastMCP: add/remove at runtime then notify
mcp.add_tool(fn, name="new_tool")
mcp.remove_tool("old_tool")      # SDK >= 1.10
# inside a request: await ctx.session.send_tool_list_changed()
```

---

## 3. Python SDK: Client API

```python
from mcp import ClientSession, StdioServerParameters, types
from mcp.client.stdio import stdio_client
from mcp.client.streamable_http import streamablehttp_client
from mcp.client.sse import sse_client                      # legacy servers

# stdio
params = StdioServerParameters(command="uv", args=["run", "server.py"], env={...}, cwd="/path")
async with stdio_client(params) as (r, w):
    async with ClientSession(r, w, sampling_callback=my_sampler, elicitation_callback=my_elicit,
                             list_roots_callback=my_roots, logging_callback=my_logger,
                             read_timeout_seconds=timedelta(seconds=30)) as s:
        await s.initialize()

# streamable http
async with streamablehttp_client("https://mcp.example.com/mcp", headers={"Authorization": f"Bearer {token}"}) as (r, w, get_sid):
    async with ClientSession(r, w) as s:
        await s.initialize()
```

| Session method | Protocol method |
|---|---|
| `await s.initialize()` | `initialize` + `notifications/initialized` |
| `await s.send_ping()` | `ping` |
| `await s.list_tools()` / `.tools` | `tools/list` |
| `await s.call_tool(name, arguments, read_timeout_seconds=, progress_callback=)` | `tools/call` |
| `await s.list_resources()` / `list_resource_templates()` | `resources/list`, `resources/templates/list` |
| `await s.read_resource(AnyUrl("prom://targets"))` | `resources/read` |
| `await s.subscribe_resource(uri)` / `unsubscribe_resource(uri)` | `resources/subscribe` |
| `await s.list_prompts()` / `get_prompt(name, arguments)` | `prompts/list`, `prompts/get` |
| `await s.set_logging_level("debug")` | `logging/setLevel` |
| `await s.complete(ref, argument)` | `completion/complete` |
| `await s.send_roots_list_changed()` | `notifications/roots/list_changed` |

Sampling callback signature:

```python
async def my_sampler(context, params: types.CreateMessageRequestParams) -> types.CreateMessageResult:
    text = await call_your_llm(params.messages, params.maxTokens)
    return types.CreateMessageResult(role="assistant", model="claude-sonnet-4", stopReason="endTurn",
                                     content=types.TextContent(type="text", text=text))
```

### OAuth client helper

```python
from mcp.client.auth import OAuthClientProvider, TokenStorage
from mcp.shared.auth import OAuthClientMetadata

auth = OAuthClientProvider(
    server_url="https://mcp.example.com/mcp",
    client_metadata=OAuthClientMetadata(client_name="incident-agent", redirect_uris=["http://localhost:3030/callback"],
                                        grant_types=["authorization_code", "refresh_token"], response_types=["code"]),
    storage=MyTokenStorage(),                    # implement get/set tokens + client info
    redirect_handler=open_browser, callback_handler=wait_for_code)
async with streamablehttp_client("https://mcp.example.com/mcp", auth=auth) as (r, w, _):
    ...
```

### Multi-server, in-memory testing

```python
from mcp.shared.memory import create_connected_server_and_client_session
async with create_connected_server_and_client_session(mcp._mcp_server) as session:
    await session.initialize()
    res = await session.call_tool("add", {"a": 1, "b": 2})
```

(`_mcp_server` is the low-level server inside a `FastMCP` instance; the helper lives in the SDK's shared module and is what the SDK's own tests use.)

---

## 4. Framework Adapters

### LangChain / LangGraph (`langchain-mcp-adapters`)

```bash
uv add langchain-mcp-adapters langgraph "langchain[aws]"
```

```python
from langchain_mcp_adapters.client import MultiServerMCPClient
from langgraph.prebuilt import create_react_agent
from langchain_aws import ChatBedrockConverse

client = MultiServerMCPClient({
    "prom":    {"transport": "stdio", "command": "uv", "args": ["--directory", "ai/mcp-servers/prom-mcp", "run", "server.py"],
                "env": {"PROMETHEUS_URL": "http://localhost:9090"}},
    "k8s":     {"transport": "streamable_http", "url": "http://localhost:8001/mcp",
                "headers": {"Authorization": "Bearer <TOKEN>"}},
    "runbook": {"transport": "streamable_http", "url": "http://localhost:8002/mcp"},
})
tools = await client.get_tools()                      # list[BaseTool], names prefixed? no - keep unique names
model = ChatBedrockConverse(model="anthropic.claude-sonnet-4-20250514-v1:0", region_name="<AWS_REGION>")
agent = create_react_agent(model, tools)
out = await agent.ainvoke({"messages": [("user", "Why is api burning error budget?")]})
```

Other helpers:

```python
from langchain_mcp_adapters.tools import load_mcp_tools          # from an existing ClientSession
from langchain_mcp_adapters.prompts import load_mcp_prompt        # prompts/get -> list[BaseMessage]
from langchain_mcp_adapters.resources import load_mcp_resources   # resources/read -> list[Blob]
async with client.session("k8s") as session:                      # keep one session open for stateful servers
    tools = await load_mcp_tools(session)
```

### Strands Agents

```bash
uv add strands-agents strands-agents-tools
```

```python
from mcp import StdioServerParameters, stdio_client
from mcp.client.streamable_http import streamablehttp_client
from strands import Agent
from strands.tools.mcp import MCPClient

prom = MCPClient(lambda: stdio_client(StdioServerParameters(command="uv", args=["run", "ai/mcp-servers/prom-mcp/server.py"])))
k8s = MCPClient(lambda: streamablehttp_client("http://localhost:8001/mcp"))

with prom, k8s:                                   # context manager keeps sessions alive for the agent's lifetime
    tools = prom.list_tools_sync() + k8s.list_tools_sync()
    agent = Agent(tools=tools, system_prompt="You are an SRE incident agent...")
    agent("Why is api burning error budget?")
```

Strands also exposes `mcp_client.call_tool_sync(tool_use_id, name, arguments)` for direct invocation and supports pagination of large tool lists via `list_tools_sync(pagination_token=...)`.

---

## 5. Host CLI and Config Cheatsheet

### Claude Code

| Command | Purpose |
|---|---|
| `claude mcp add <name> -- <cmd> [args...]` | Add stdio server (local scope) |
| `claude mcp add --scope project <name> -- ...` | Write to `.mcp.json` (shared) |
| `claude mcp add --scope user <name> -- ...` | All projects for this user |
| `claude mcp add --transport http <name> <url>` | Remote Streamable HTTP |
| `claude mcp add --transport sse <name> <url>` | Legacy SSE |
| `claude mcp add -e KEY=value <name> -- ...` | Set env var for server |
| `claude mcp add-json <name> '{"command":...}'` | Add from JSON |
| `claude mcp add-from-claude-desktop` | Import Desktop config |
| `claude mcp list` / `get <name>` / `remove <name>` | Manage |
| `claude mcp serve` | Expose Claude Code itself as an MCP server |
| `/mcp` (in session) | Status, OAuth login, enable/disable |
| `MCP_TIMEOUT=30000 claude` | Server startup timeout (ms) |
| `MAX_MCP_OUTPUT_TOKENS=50000 claude` | Cap on tool output size |

### Config files

| Host | File | Root key |
|---|---|---|
| Claude Code (project) | `.mcp.json` | `mcpServers` |
| Claude Code (user) | `~/.claude.json` | `mcpServers` |
| Claude Desktop | `claude_desktop_config.json` | `mcpServers` |
| Cursor | `.cursor/mcp.json` | `mcpServers` |
| VS Code | `.vscode/mcp.json` | `servers`, `inputs` |
| Windsurf | `~/.codeium/windsurf/mcp_config.json` | `mcpServers` |

### Inspector

```bash
npx @modelcontextprotocol/inspector <cmd> [args]                       # UI
npx @modelcontextprotocol/inspector --cli <cmd> --method tools/list    # CLI
npx @modelcontextprotocol/inspector --cli <cmd> --method tools/call --tool-name list_pods --tool-arg namespace=api
npx @modelcontextprotocol/inspector --cli <cmd> --method resources/read --uri prom://targets
npx @modelcontextprotocol/inspector --cli http://127.0.0.1:8000/mcp --transport http --method tools/list
npx @modelcontextprotocol/inspector --config mcp.json --server prom     # load from a config file
```

Environment: `CLIENT_PORT=6274 SERVER_PORT=6277 MCP_PROXY_FULL_ADDRESS=... DANGEROUSLY_OMIT_AUTH=true` (never in shared environments).

### `mcp` Python CLI

```bash
uv run mcp dev server.py                 # Inspector + server
uv run mcp dev server.py --with pandas   # extra deps
uv run mcp run server.py                 # run a server file (looks for `mcp`/`server`/`app` object)
uv run mcp install server.py --name prom # add to Claude Desktop config
uv run mcp install server.py -v PROMETHEUS_URL=http://localhost:9090 -f .env
uv run mcp version
```

---

## 6. Raw JSON-RPC Snippets (debugging)

Save as `handshake.jsonl` and pipe into any stdio server:

```json
{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-06-18","capabilities":{},"clientInfo":{"name":"dbg","version":"0"}}}
{"jsonrpc":"2.0","method":"notifications/initialized"}
{"jsonrpc":"2.0","id":2,"method":"tools/list"}
{"jsonrpc":"2.0","id":3,"method":"resources/list"}
{"jsonrpc":"2.0","id":4,"method":"prompts/list"}
{"jsonrpc":"2.0","id":5,"method":"tools/call","params":{"name":"list_alerts","arguments":{"state":"firing"}}}
{"jsonrpc":"2.0","id":6,"method":"resources/read","params":{"uri":"prom://targets"}}
{"jsonrpc":"2.0","id":7,"method":"prompts/get","params":{"name":"triage_alert","arguments":{"alertname":"HighErrorBudgetBurn"}}}
{"jsonrpc":"2.0","id":8,"method":"logging/setLevel","params":{"level":"debug"}}
{"jsonrpc":"2.0","id":9,"method":"ping"}
```

```bash
cat handshake.jsonl | uv run server.py 2>server.err | jq -c '.id, (.result // .error) | if type=="object" then keys else . end'
```

Streamable HTTP helper function for a shell:

```bash
mcp_post() { # usage: mcp_post URL SESSION_ID JSON
  curl -s "$1" -H 'Content-Type: application/json' -H 'Accept: application/json, text/event-stream' \
       -H "Mcp-Session-Id: $2" -H 'MCP-Protocol-Version: 2025-06-18' -d "$3" | sed -n 's/^data: //p'
}
mcp_post http://127.0.0.1:8000/mcp "$SID" '{"jsonrpc":"2.0","id":2,"method":"tools/list"}' | jq '.result.tools[].name'
```

---

## 7. TypeScript SDK Equivalents

| Python | TypeScript |
|---|---|
| `FastMCP("name")` | `new McpServer({ name, version })` |
| `@mcp.tool()` | `server.registerTool(name, { description, inputSchema: zodShape, outputSchema?, annotations? }, handler)` |
| `@mcp.resource("uri")` | `server.registerResource(name, uri, { mimeType }, handler)` |
| `@mcp.resource("x://{id}")` | `server.registerResource(name, new ResourceTemplate("x://{id}", { list: undefined }), ..., handler)` |
| `@mcp.prompt()` | `server.registerPrompt(name, { argsSchema }, handler)` |
| `mcp.run(transport="stdio")` | `await server.connect(new StdioServerTransport())` |
| `mcp.run(transport="streamable-http")` | Express app + `new StreamableHTTPServerTransport({ sessionIdGenerator })` |
| `stdio_client` | `new StdioClientTransport({ command, args })` + `new Client({name, version})` |
| `streamablehttp_client` | `new StreamableHTTPClientTransport(new URL(url))` |
| `ctx.info()` | `server.server.sendLoggingMessage({ level, data })` |

---

## 8. Kubernetes and Prometheus API calls the capstone servers wrap

| Tool | Underlying call |
|---|---|
| `query_instant` | `GET /api/v1/query?query=&time=` |
| `query_range` | `GET /api/v1/query_range?query=&start=&end=&step=` |
| `list_alerts` | `GET /api/v1/alerts` (filter `state`) |
| `get_slo_burn_rate` | two `query_instant` calls with `rate(...[1h])` and `[6h]` windows divided by `(1 - SLO)` |
| `prom://targets` | `GET /api/v1/targets?state=active` |
| `list_pods` | `CoreV1Api.list_namespaced_pod(namespace, label_selector=)` |
| `describe_pod` | `CoreV1Api.read_namespaced_pod` + `list_namespaced_event(field_selector=involvedObject.name=)` |
| `get_logs` | `CoreV1Api.read_namespaced_pod_log(name, namespace, container=, tail_lines=, previous=)` |
| `get_events` | `CoreV1Api.list_namespaced_event(namespace)` sorted by `last_timestamp` |
| `rollout_restart` | `AppsV1Api.patch_namespaced_deployment(name, namespace, {"spec":{"template":{"metadata":{"annotations":{"kubectl.kubernetes.io/restartedAt": now}}}}})` |
| `search_runbooks` | `QdrantClient.query_points(collection, query=embedding, limit=, with_payload=True)` |

Continue with [04-hands-on-labs.md](04-hands-on-labs.md).
