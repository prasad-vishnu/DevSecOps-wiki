# MCP: Troubleshooting

| | |
|---|---|
| **Cluster** | 10 AI Infrastructure, RAG, MCP & Evals |
| **Role** | Anchor |
| **Prerequisites** | [02-core-concepts.md](02-core-concepts.md), [04-hands-on-labs.md](04-hands-on-labs.md) |
| **Target version** | MCP spec `2025-06-18`; `mcp` 1.12+ |
| **Capstone phase** | Phase 6 (Months 21-24) |

> Symptom-to-cause-to-fix tables for the failures you will actually hit: handshake failures, stdout corruption in stdio servers, schema validation errors, session and transport problems with Streamable HTTP, authentication errors, timeouts, host-specific configuration mistakes, and agent-level misbehaviour. Followed by a diagnostic workflow and the exact commands to run.

---

## 1. Diagnostic Workflow

```mermaid
flowchart TD
    A[Tool call fails or server "not connected"] --> B{Does the server start alone?}
    B -- no --> C[Run the command from the config by hand<br/>uv run server.py / docker run ...<br/>read stderr]
    B -- yes --> D{Does Inspector CLI tools/list work?}
    D -- no --> E[Protocol problem: stdout pollution, version mismatch,<br/>wrong transport, auth]
    D -- yes --> F{Does the host show the server connected?}
    F -- no --> G[Host config: path, env, scope, timeout,<br/>JSON syntax, header]
    F -- yes --> H{Does the model call the tool?}
    H -- no --> I[Description / naming / too many tools /<br/>permission denied in host]
    H -- yes --> J{isError true?}
    J -- yes --> K[Downstream error: RBAC, URL, PromQL,<br/>read the content text]
    J -- no --> L[Output shape / size / timeout - inspect result]
```

First three commands, always:

```bash
# 1. run the server exactly as the host would, see stderr
uv --directory ./ai/mcp-servers/prom-mcp run server.py 2>&1 | head
# 2. protocol-level smoke test without any host
npx @modelcontextprotocol/inspector --cli uv --directory ./ai/mcp-servers/prom-mcp run server.py --method tools/list
# 3. host view
claude mcp list && claude mcp get prom
```

---

## 2. Handshake and Lifecycle

| Symptom | Likely cause | Diagnose | Fix |
|---|---|---|---|
| Host shows `✘ Failed to connect` immediately | Command not found or wrong working directory (`uv` not on host PATH, relative path) | Run the exact `command` + `args` from the config in a fresh shell | Use absolute paths or `uv --directory`; on macOS GUI apps (Claude Desktop) do not inherit shell PATH, so use `/Users/<USER>/.local/bin/uv` |
| `Unexpected token 'b', "booting..." is not valid JSON` or `SyntaxError: Unexpected token` in host logs | Server wrote non-JSON to stdout (a `print`, a library banner, a warning) | `uv run server.py < /dev/null \| head -c 200` shows the offending text | Remove `print`; `logging.basicConfig(stream=sys.stderr)`; silence libraries (`PYTHONWARNINGS=ignore`, `TQDM_DISABLE=1`, set `transformers`/`fastembed` logging to stderr) |
| `Received request before initialization was complete` (-32602) | Client sent a request before `notifications/initialized`; usually a hand-written client or test | Check message order | Call `await session.initialize()` first; in raw tests send the notification |
| Connection closes right after `initialize` | Protocol version mismatch (old SDK server vs new client) or server crashed on first request | stderr traceback; compare `protocolVersion` | Upgrade `mcp`; check `SUPPORTED_PROTOCOL_VERSIONS` |
| `Method not found` (-32601) on `resources/subscribe` or `elicitation/create` | Capability not advertised by the other side | Inspect `initialize` result capabilities | Implement/advertise the capability; guard calls with capability checks |
| Server works in Inspector, hangs in host | Host timeout during startup (model download, slow import) | Measure startup time; `MCP_TIMEOUT` | Lazy-load heavy objects in `lifespan`; raise `MCP_TIMEOUT=60000` for Claude Code |
| `ping` fails periodically, host reconnects | Event loop blocked by a synchronous call (sync `kubernetes` client in async tool) | Add timing logs | Use `await anyio.to_thread.run_sync(...)` for blocking calls or make tools sync `def` so FastMCP runs them in a thread |

---

## 3. Tools and Schemas

| Symptom | Likely cause | Diagnose | Fix |
|---|---|---|---|
| `Input validation error: 'sixty' is not of type 'integer'` with `isError: true` | Model passed wrong type; schema allows it ambiguously | Look at `inputSchema` in `tools/list` | Tighten types, add `description` and examples in `Field(...)`; avoid `Any` and unions |
| Tool never called by the model | Description does not say *when*; too many similar tools; name collision across servers | Ask the model "which tools do you have and when would you use each?" | Rewrite descriptions ("Use when ..."), reduce tool count, give unique prefixes (`prom_query_instant`) |
| `structuredContent` missing | Return type not annotated or `structured_output=False` | Check `outputSchema` presence | Annotate return type with a Pydantic model/TypedDict |
| `Output validation error` | Returned object does not match declared output schema (e.g. `None` for a required field) | Reproduce in unit test | Make fields optional or fix the handler |
| Tool result huge, model truncates or host errors `MAX_MCP_OUTPUT_TOKENS` | No output cap | Measure `len(text)` | Cap, summarise, paginate, return `resource_link` for full content |
| Enum argument rejected | Used `Literal` in a nested model unsupported by the host's schema subset | Check the generated schema for `anyOf`/`$ref` | Flatten to top-level `Literal`; avoid recursive models |
| `Error executing tool x: ...` but you expected a JSON-RPC error | SDK converts handler exceptions to tool errors by design | - | This is correct; raise `McpError` only for protocol-level problems |
| Resource template URI never matches (`Unknown resource: k8s://api/pods/x/logs`) | Template variables do not match function parameter names, or function has extra required params | Compare `uriTemplate` with signature | Function params must equal template variables exactly, nothing else (except `ctx`) |

---

## 4. Streamable HTTP and SSE

| Symptom | Likely cause | Diagnose | Fix |
|---|---|---|---|
| `406 Not Acceptable` | Missing `Accept: application/json, text/event-stream` | `curl -i` with and without header | Add both media types to `Accept` |
| `400 Bad Request: Missing session ID` | Stateful server; request lacks `Mcp-Session-Id` | Check `initialize` response header | Echo the header; or run `stateless_http=True` |
| `404 Session not found` after a while or intermittently | Server restarted, session expired, or round-robin across replicas | `kubectl get pods` restarts; replica count | Stateless mode, single replica, or sticky sessions; client must re-initialize on 404 |
| Responses arrive all at once at the end, progress missing | Proxy buffers SSE (NGINX `proxy_buffering on`, some gateways) | Compare direct pod access vs via ingress | `proxy_buffering off; proxy_read_timeout 3600;` or set `json_response=True` |
| Connection drops after 60 s | Idle timeout on LB/ingress for long tool calls or GET stream | Timing | Raise idle timeouts; send progress notifications as keep-alives; shorten tool work |
| `Invalid Host header` / `421` | SDK transport security rejecting host (DNS rebinding protection) | Server logs | Set allowed hosts in `TransportSecuritySettings` or bind correctly behind the gateway |
| Works with `curl`, fails in browser-based Inspector | CORS / `Origin` validation | Browser console | Allow Inspector origin in dev only; use CLI mode |
| Legacy client cannot connect | Client speaks 2024-11-05 HTTP+SSE, server only Streamable HTTP | Client logs `GET /sse 404` | Run both (`mcp.run(transport="sse")` on a second path) or upgrade the client |

---

## 5. Authentication and Authorization

| Symptom | Likely cause | Diagnose | Fix |
|---|---|---|---|
| `401 Unauthorized` with `WWW-Authenticate: Bearer resource_metadata=...` | No token sent, or token rejected (`iss`/`aud`/`exp`/signature) | Decode token (`jwt.io`-style locally, never paste real tokens online), compare `aud` with `MCP_RESOURCE_URL` | Request token with `resource=` indicator; align audience; check clock skew; check JWKS URL reachable from the pod |
| `403 Forbidden insufficient_scope` | Token lacks `required_scopes` | Inspect `scope` claim | Add scope to client registration/consent; map roles to scopes in the IdP |
| Host never opens the browser for OAuth | Host does not support OAuth for that transport or metadata document unreachable | `curl https://mcp.example.com/.well-known/oauth-protected-resource` | Serve metadata at the root of the resource host (and at path-suffixed location for path-based resources); use `claude mcp add --transport http` and `/mcp` to authenticate |
| Dynamic client registration fails | AS has DCR disabled | AS logs | Pre-register the client; configure static `client_id` in the host |
| `Kubernetes API 403: Forbidden` inside a tool | Server identity lacks RBAC (expected for writer tools) | `kubectl auth can-i --as=system:serviceaccount:mcp:k8s-mcp patch deployments -n api` | Bind the writer role only where intended |
| Prometheus `401` | Server reads a wrong/expired bearer token for Prometheus | Check env | Rotate; use a ServiceAccount token projected volume with short TTL |

---

## 6. Host Configuration

| Symptom | Likely cause | Diagnose | Fix |
|---|---|---|---|
| Server missing in Claude Code | Wrong scope (added `local` on a different machine) or `.mcp.json` not approved | `claude mcp list`, run `/mcp` | Use `--scope project`, approve the project servers when prompted |
| `.mcp.json` ignored | Invalid JSON (trailing comma), wrong root key | `jq . .mcp.json` | Fix JSON; root key `mcpServers` (VS Code uses `servers`) |
| Env var not visible to server | Env block missing; GUI host has minimal environment | Print `os.environ` to stderr | Put values in `env` block or `-e KEY=value`; reference `${VAR}` expansions where the host supports them |
| Windows: `'uv' is not recognized` | PATH differs for GUI apps | - | Full path to `uv.exe`; double backslashes in JSON |
| Claude Desktop shows no tools after restart | Config file edited but app not fully quit (tray) | Logs at `~/Library/Logs/Claude/mcp*.log` or `%APPDATA%\Claude\logs` | Quit fully; validate JSON |
| Cursor/VS Code shows "0 tools" | Server advertised no `tools` capability (no tools registered) or startup error | Output panel > MCP | Register at least one tool; read the output channel |
| Permission prompt on every call | Host permission rules | - | Allow-list read-only tools in host settings (Claude Code: `permissions.allow: ["mcp__prom__*"]`) |

---

## 7. Agent-Level Behaviour

| Symptom | Likely cause | Diagnose | Fix |
|---|---|---|---|
| Agent calls `rollout_restart` without investigating | System prompt lacks process; destructive tool too attractive | Trace spans in Tempo/LangSmith | Prompt "investigate with read-only tools first"; approval gate; DeepEval `ToolCorrectnessMetric` with `expected_tools` ordering |
| Agent loops on the same tool | Error text not informative; model retries | Look at `isError` content | Return actionable error messages ("namespace not allowed; allowed: [api]") |
| Agent ignores runbooks | `search_runbooks` description weak; retrieval scores low | Evaluate with RAGAS `ContextRecall` | Improve description; tune hybrid search; return `uri` so the agent can read the full runbook |
| Agent follows instructions found in logs | Prompt injection via tool output | Red-team test with injected log line | Delimit tool output, instruct model to treat as data, approval gate, Promptfoo red-team in CI |
| Context window exceeded after a few calls | Large outputs | Count tokens per tool result | Caps, summaries via Sampling, pagination |

---

## 8. Useful Diagnostic Commands

```bash
# Raw stdout check for pollution (should print nothing within 2s)
timeout 2 uv run server.py < /dev/null; echo "exit=$?"

# Full JSON-RPC trace of an Inspector CLI call
npx @modelcontextprotocol/inspector --cli uv run server.py --method tools/call --tool-name list_alerts 2>&1 | tee trace.log

# Streamable HTTP probe
curl -s -i http://127.0.0.1:8000/mcp -H 'Content-Type: application/json' -H 'Accept: application/json, text/event-stream' \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-06-18","capabilities":{},"clientInfo":{"name":"dbg","version":"0"}}}' | head -20

# Protected resource metadata
curl -s https://mcp.example.com/.well-known/oauth-protected-resource | jq .

# Kubernetes identity checks
kubectl auth can-i list pods --as=system:serviceaccount:mcp:k8s-mcp -n api
kubectl -n mcp logs deploy/k8s-mcp --since=10m | grep -E "AUDIT|ERROR"

# Python SDK debug logging
FASTMCP_LOG_LEVEL=DEBUG uv run server.py

# Claude Code debug
claude --mcp-debug
```

### Reading an SDK traceback

```text
  File ".../mcp/server/fastmcp/tools/base.py", line 85, in run
    result = await self.fn_metadata.call_fn_with_arg_validation(
pydantic_core._pydantic_core.ValidationError: 1 validation error for query_rangeArguments
minutes
  Input should be a valid integer, unable to parse string as an integer [type=int_parsing, input_value='sixty', input_type=str]
```

`ValidationError` in `call_fn_with_arg_validation` is always the model's arguments failing your schema; the fix is in the schema or the description, not in the handler.

---

## 9. Known Version Pitfalls

| Version | Pitfall |
|---|---|
| `mcp` < 1.8 | No Streamable HTTP server support; only SSE |
| `mcp` < 1.10 | No structured output / `outputSchema`, no elicitation, no `TokenVerifier` API |
| Spec 2025-03-26 clients | Send JSON-RPC batches; 2025-06-18 servers reject them |
| Clients omitting `MCP-Protocol-Version` header | Server assumes 2025-03-26 behaviour |
| `langchain-mcp-adapters` < 0.1 | `MultiServerMCPClient` was an async context manager; newer versions use `await client.get_tools()` without `async with` |
| Inspector < 0.14.1 | Unauthenticated proxy (CVE-2025-49596); upgrade |

Next: [07-interview-questions.md](07-interview-questions.md).
