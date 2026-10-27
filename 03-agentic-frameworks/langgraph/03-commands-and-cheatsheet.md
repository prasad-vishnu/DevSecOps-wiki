# LangGraph: Commands and Cheatsheet

| | |
|---|---|
| **Cluster** | 03 Agentic Frameworks |
| **Role** | Anchor |
| **Prerequisites** | [02-core-concepts.md](02-core-concepts.md) |
| **Target version** | LangGraph 1.x; CLI `langgraph-cli` 0.3+ |
| **Capstone phase** | Phase 6 (Months 21-24) |

> A task-organised reference for the Python API, the CLI, the Server HTTP API and the SDK. Every entry has a one-line explanation. Keep this open while doing the labs.

---

## 1. Imports

```python
# Graph construction
from langgraph.graph import StateGraph, START, END, MessagesState      # builder, sentinels, prebuilt state
from langgraph.graph.message import add_messages                         # reducer for message lists
# Control flow
from langgraph.types import Command, Send, interrupt, RetryPolicy, CachePolicy, StateSnapshot
# Prebuilt
from langgraph.prebuilt import create_react_agent, ToolNode, tools_condition, InjectedState, InjectedStore
# Persistence
from langgraph.checkpoint.memory import MemorySaver                      # alias InMemorySaver in newer releases
from langgraph.checkpoint.postgres import PostgresSaver
from langgraph.checkpoint.postgres.aio import AsyncPostgresSaver
from langgraph.checkpoint.redis import RedisSaver
from langgraph.store.memory import InMemoryStore
from langgraph.store.postgres import PostgresStore
# Runtime helpers inside nodes
from langgraph.config import get_stream_writer, get_store
# Functional API
from langgraph.func import entrypoint, task
# Errors
from langgraph.errors import GraphRecursionError, InvalidUpdateError, NodeInterrupt
# LangChain pieces used with graphs
from langchain.chat_models import init_chat_model
from langchain_core.messages import HumanMessage, AIMessage, SystemMessage, ToolMessage, RemoveMessage
from langchain_core.tools import tool, InjectedToolArg, InjectedToolCallId
```

---

## 2. Building a Graph

| Call | Meaning |
|---|---|
| `StateGraph(State)` | Builder for a graph whose nodes read/write `State` |
| `StateGraph(State, input_schema=In, output_schema=Out)` | Narrow what callers pass in and get back (older kwargs: `input=`, `output=`) |
| `StateGraph(State, context_schema=Ctx)` | Typed run-scoped context (0.6+) |
| `builder.add_node("name", fn)` | Register a node; `fn(state)` or `fn(state, config)` |
| `builder.add_node(fn)` | Node named after the function |
| `builder.add_node("name", fn, retry_policy=RetryPolicy(max_attempts=3))` | Retry this node on exception (older: `retry=`) |
| `builder.add_node("name", fn, cache_policy=CachePolicy(ttl=300))` | Cache node output by input hash |
| `builder.add_node("name", compiled_subgraph)` | Subgraph as a node (shared state keys) |
| `builder.add_edge(START, "a")` | Entry edge |
| `builder.set_entry_point("a")` | Same as above |
| `builder.add_edge("a", "b")` | Fixed edge |
| `builder.add_edge(["a", "b"], "c")` | Fan-in: `c` runs after both finish |
| `builder.add_conditional_edges("a", router)` | Router returns next node name(s) or `END` |
| `builder.add_conditional_edges("a", router, {"yes": "b", "no": END})` | Router returns a key mapped to a node |
| `builder.add_conditional_edges("a", fan_out, ["worker"])` | Router returns `list[Send]`; list declares possible targets for rendering |
| `builder.add_edge("z", END)` | Terminal edge |
| `builder.compile(checkpointer=..., store=..., interrupt_before=[...], interrupt_after=[...], debug=False)` | Validate and produce a runnable graph |
| `graph.get_graph().draw_mermaid()` | Mermaid source of the topology |
| `graph.get_graph(xray=True).draw_mermaid_png()` | PNG including subgraphs (needs network or `pyppeteer`) |

---

## 3. Running a Graph

| Call | Meaning |
|---|---|
| `graph.invoke(inputs, config)` | Run to completion (or interrupt); returns final state |
| `await graph.ainvoke(inputs, config)` | Async variant |
| `graph.invoke(None, config)` | Resume from the thread's latest checkpoint |
| `graph.invoke(Command(resume=value), config)` | Resume a pending `interrupt()` with a value |
| `graph.invoke(Command(update={...}, goto="node"), config)` | Resume with a state update and routing |
| `graph.stream(inputs, config, stream_mode="updates")` | Iterate events; see table below |
| `graph.astream(inputs, config, stream_mode=["updates", "messages"])` | Multiple modes; yields `(mode, chunk)` |
| `graph.astream(..., subgraphs=True)` | Include nested graph events as `(namespace, chunk)` |
| `graph.batch([in1, in2], [cfg1, cfg2])` | Run many threads concurrently |
| `graph.invoke(inputs, {"configurable": {"thread_id": "t1"}, "recursion_limit": 60})` | Raise the super-step limit |
| `graph.invoke(inputs, config, durability="sync")` | Write checkpoints synchronously (0.6+) |
| `graph.invoke(inputs, config, context={"user_id": "u1"})` | Typed context (0.6+) |

### Stream modes

| Mode | Chunk shape | One-liner |
|---|---|---|
| `values` | full state dict | State after each step |
| `updates` | `{node: update}` | What each node returned |
| `messages` | `(AIMessageChunk, metadata)` | LLM tokens; `metadata["langgraph_node"]` tells which node |
| `custom` | anything | From `get_stream_writer()(obj)` in a node |
| `debug` | dicts with `type` in `task`, `task_result`, `checkpoint` | Executor internals |

---

## 4. State Inspection and Time Travel

| Call | Meaning |
|---|---|
| `graph.get_state(config)` | `StateSnapshot` of latest checkpoint: `.values`, `.next`, `.tasks`, `.config`, `.metadata`, `.created_at`, `.parent_config` |
| `graph.get_state(config).tasks[0].interrupts` | Pending interrupts with `.value` payload |
| `graph.get_state(config, subgraphs=True)` | Include subgraph snapshots in `.tasks[i].state` |
| `graph.get_state_history(config)` | Iterator of snapshots, newest first |
| `graph.get_state_history(config, limit=10, before=cfg)` | Page through history |
| `graph.update_state(config, values)` | Apply `values` via reducers as a new checkpoint; returns new config |
| `graph.update_state(config, values, as_node="triage")` | Attribute the update to a node so routing continues from its edges |
| `graph.invoke(None, snapshot.config)` | Replay from that checkpoint (fork if it is not the latest) |
| `checkpointer.list(config)` | Raw checkpoint tuples |
| `checkpointer.delete_thread(thread_id)` | Remove a thread's checkpoints (1.x; older: delete via SQL) |

---

## 5. Checkpointers and Stores

```python
# In-memory (tests)
saver = MemorySaver()

# Postgres (sync)
with PostgresSaver.from_conn_string(PG_URI) as saver:
    saver.setup()
    graph = builder.compile(checkpointer=saver)

# Postgres (async, with pool)
from psycopg_pool import AsyncConnectionPool
async with AsyncConnectionPool(PG_URI, max_size=20, kwargs={"autocommit": True, "prepare_threshold": 0}) as pool:
    saver = AsyncPostgresSaver(pool)
    await saver.setup()
    graph = builder.compile(checkpointer=saver)

# Redis
with RedisSaver.from_conn_string(REDIS_URI) as saver:
    saver.setup()

# Store (long-term memory)
store = InMemoryStore(index={"embed": embeddings, "dims": 1024})
with PostgresStore.from_conn_string(PG_URI) as store:
    store.setup()
```

| Store call | Meaning |
|---|---|
| `store.put(("ns", "sub"), "key", {"...": ...})` | Upsert a JSON document |
| `store.get(("ns", "sub"), "key")` | Fetch an `Item` or `None` |
| `store.search(("ns",), query="text", filter={"k": "v"}, limit=5)` | Semantic + filter search |
| `store.list_namespaces(prefix=("ns",))` | Enumerate namespaces |
| `store.delete(ns, key)` | Remove |
| `get_store()` inside a node | Access the compiled store without plumbing |

---

## 6. Human-in-the-Loop

| Call | Meaning |
|---|---|
| `value = interrupt(payload)` | Pause here; on resume returns the resume value |
| `graph.invoke(Command(resume=value), config)` | Continue; node re-runs from the top |
| `compile(interrupt_before=["node"])` | Static breakpoint before `node` |
| `compile(interrupt_after=["node"])` | Static breakpoint after `node` |
| `graph.invoke(None, config)` | Continue after a static breakpoint |
| `result["__interrupt__"]` | Interrupt payloads returned by `invoke` (1.x) |
| `graph.update_state(config, {...})` then `invoke(None, config)` | Edit then continue (human edits the proposed action) |
| `create_react_agent(..., interrupt_before=["tools"])` | Approve every tool call in the prebuilt agent |

---

## 7. Tools in Graphs

| Call | Meaning |
|---|---|
| `@tool def f(x: int) -> str: """doc"""` | Define a tool; docstring + type hints become the schema |
| `@tool(args_schema=Model)` | Explicit Pydantic schema |
| `model.bind_tools(tools)` | Give the model the schemas; returns a new runnable |
| `model.bind_tools(tools, tool_choice="any")` | Force a tool call |
| `ToolNode(tools, handle_tool_errors=True)` | Execute tool calls from the last `AIMessage` |
| `ToolNode(tools, handle_tool_errors="Custom message")` | Return a fixed error text to the model |
| `Annotated[dict, InjectedState]` tool argument | Tool receives graph state (not exposed to the model) |
| `Annotated[BaseStore, InjectedStore]` | Tool receives the store |
| `Annotated[str, InjectedToolCallId]` | Tool receives its own call id (for `ToolMessage`s and handoffs) |
| `Annotated[str, InjectedToolArg]` | Argument supplied by code, hidden from the model |
| `tools_condition(state)` | Prebuilt router: `"tools"` or `END` |
| `return Command(update={...}, goto=...)` from a tool | Tool updates state / routes (used for handoffs) |

MCP tools:

```python
from langchain_mcp_adapters.client import MultiServerMCPClient
client = MultiServerMCPClient({
    "prometheus": {"transport": "streamable_http", "url": "http://mcp-prometheus.monitoring.svc:8000/mcp"},
    "kubernetes": {"command": "python", "args": ["-m", "k8s_mcp_server"], "transport": "stdio"},
})
tools = await client.get_tools()          # list of LangChain tools, ready for ToolNode
```

---

## 8. Multi-Agent Helpers

```bash
pip install langgraph-supervisor langgraph-swarm
```

| Call | Meaning |
|---|---|
| `create_supervisor(agents=[a, b], model=m, prompt="...")` | Supervisor graph; `.compile(checkpointer=...)` |
| `create_handoff_tool(agent_name="b", description="...")` | Tool that transfers control (swarm) |
| `create_swarm(agents=[a, b], default_active_agent="a")` | Swarm graph remembering the active agent per thread |
| `Command(goto="b", graph=Command.PARENT)` | Manual handoff from a subgraph |
| `Send("worker", payload)` | Spawn parallel worker invocations |

---

## 9. Functional API

| Call | Meaning |
|---|---|
| `@entrypoint(checkpointer=saver)` | Durable workflow function; invoked like a graph with a `thread_id` |
| `@task` | Checkpointed step; returns a future, call `.result()` |
| `entrypoint.final(value=..., save=...)` | Return one value to caller, persist another |
| `interrupt()` inside an entrypoint | Same semantics as in graphs |

---

## 10. LangGraph CLI

| Command | Meaning |
|---|---|
| `pip install -U "langgraph-cli[inmem]"` | CLI with the in-memory dev server |
| `langgraph new ./my-agent --template new-langgraph-project-python` | Scaffold a project |
| `langgraph dev --port 2024 --no-browser` | Hot-reloading local server from `langgraph.json` |
| `langgraph dev --allow-blocking` | Permit sync blocking calls in nodes (dev only) |
| `langgraph build -t ghcr.io/example/incident-agent:1.0.0` | Build the server container image |
| `langgraph up --port 8123 --postgres-uri "$PG_URI"` | Run the built image locally with Postgres |
| `langgraph dockerfile Dockerfile.langgraph` | Emit the Dockerfile the build would use (customise and build yourself) |
| `langgraph --version` | CLI version |

`langgraph.json` keys: `dependencies` (local paths or packages), `graphs` (`name: path:variable`), `env` (file or map), `python_version`, `dockerfile_lines`, `store` (vector index config), `http` (custom routes), `auth`.

---

## 11. LangGraph Server HTTP API (self-hosted or Platform)

| Endpoint | Meaning |
|---|---|
| `POST /assistants/search` `{}` | List graphs exposed by `langgraph.json` |
| `POST /threads` `{}` | Create a thread; returns `thread_id` |
| `POST /threads/{tid}/runs/wait` `{"assistant_id": "incident_agent", "input": {...}}` | Run and wait for final state |
| `POST /threads/{tid}/runs/stream` `{..., "stream_mode": ["updates"]}` | SSE stream |
| `POST /threads/{tid}/runs` `{..., "command": {"resume": {"approved": true}}}` | Resume an interrupt |
| `GET /threads/{tid}/state` | Current `StateSnapshot` |
| `GET /threads/{tid}/history` | Checkpoint history |
| `POST /threads/{tid}/state` `{"values": {...}, "as_node": "triage"}` | `update_state` over HTTP |
| `POST /runs/crons` | Scheduled runs (Platform) |
| `GET /ok` | Health |

Python SDK:

```python
from langgraph_sdk import get_client
client = get_client(url="http://localhost:2024")
thread = await client.threads.create()
async for ev in client.runs.stream(thread["thread_id"], "incident_agent", input={"alert": alert}, stream_mode="updates"):
    print(ev.event, ev.data)
await client.runs.wait(thread["thread_id"], "incident_agent", command={"resume": {"approved": True}})
```

---

## 12. Observability

| Setting / call | Meaning |
|---|---|
| `LANGSMITH_TRACING=true`, `LANGSMITH_API_KEY`, `LANGSMITH_PROJECT` | Trace every run to LangSmith |
| `config={"run_name": "triage", "tags": ["prod"], "metadata": {"incident": "inc-1"}}` | Annotate the trace |
| `from langsmith import traceable` `@traceable` | Trace arbitrary functions |
| `graph.invoke(..., config={"callbacks": [handler]})` | Attach a `BaseCallbackHandler` |
| `from langchain_core.globals import set_debug; set_debug(True)` | Print every runnable input/output |
| `opentelemetry-instrument python -m uvicorn ...` | Zero-code OTel for FastAPI/HTTP; add an OpenInference/OpenLLMetry instrumentor for LangChain spans |
| `stream_mode="debug"` | Executor-level events without external tooling |

---

## 13. Testing

| Pattern | Meaning |
|---|---|
| Call node functions directly with a dict | Unit test node logic |
| `graph.invoke(inputs, config)` with `MemorySaver` | Integration test the topology |
| Fake tool-calling model (`BaseChatModel` subclass returning scripted `AIMessage`s) | Deterministic agent trajectories without a provider |
| `graph.get_state_history(config)` assertions | Verify the path taken |
| `pytest -k approval` with `Command(resume=...)` | Test both branches of the gate |
| `langgraph dev` + SDK | Smoke test the server contract |

---

## 14. Useful Environment Variables

| Variable | Meaning |
|---|---|
| `BEDROCK_MODEL_ID` | Model id passed to `init_chat_model(..., model_provider="bedrock_converse")` |
| `AWS_REGION`, `AWS_PROFILE` | Bedrock region and local credentials |
| `LANGGRAPH_PG_URI` | Postgres DSN for `PostgresSaver`/`PostgresStore` (project convention) |
| `LANGSMITH_*` | Tracing |
| `LANGGRAPH_AES_KEY` | Encrypt checkpoint blobs at rest in LangGraph Server |
| `BG_JOB_ISOLATED_LOOPS=true` | Server: isolate background runs (Platform) |
| `OTEL_EXPORTER_OTLP_ENDPOINT`, `OTEL_SERVICE_NAME` | OpenTelemetry export |

Next: [04-hands-on-labs.md](04-hands-on-labs.md).
