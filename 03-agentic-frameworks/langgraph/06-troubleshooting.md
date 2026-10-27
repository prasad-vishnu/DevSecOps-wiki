# LangGraph: Troubleshooting

| | |
|---|---|
| **Cluster** | 03 Agentic Frameworks |
| **Role** | Anchor |
| **Prerequisites** | [02-core-concepts.md](02-core-concepts.md) |
| **Target version** | LangGraph 1.x |
| **Capstone phase** | Phase 6 (Months 21-24) |

> Symptom-to-fix tables for graph construction, execution, persistence, human-in-the-loop, streaming, providers and deployment, followed by the diagnostic commands and a debugging workflow. Error strings are quoted as they appear so you can search this page.

---

## 1. Graph Construction

| Symptom | Likely cause | Diagnose | Fix |
|---|---|---|---|
| `ValueError: Node 'x' is not reachable` at `compile()` | Missing edge from `START` or a typo in a node name | Print `builder.nodes.keys()` and `builder.edges` | Add `add_edge(START, ...)` or fix the name |
| `ValueError: Found edge ending at unknown node 'tools'` | Conditional edge mapping targets a node that was never added | Check `add_conditional_edges` mapping values | Add the node or correct the mapping |
| `ValueError: Already found path for node 'x'` | Two unconditional edges leave the same node | Review `add_edge` calls | Use `add_conditional_edges` or `Command` for branching |
| `TypeError: State ... has no attribute` / `KeyError: 'plan'` inside a node | Key missing from input; schema has no defaults | Print `state.keys()` | Pass all keys in the initial input or use `state.get()`; Pydantic state gives defaults |
| `InvalidUpdateError: Must write to at least one channel` | Node returned a key not in the schema or returned the wrong type | Compare returned dict with `State` | Return only declared keys |
| `InvalidUpdateError: Can receive only one value per step. Use an Annotated key to handle multiple values` | Two parallel nodes wrote the same key without a reducer | Which nodes ran in the same super-step (`stream_mode="debug"`) | Add a reducer (`operator.add`) or serialise the nodes |
| Graph picture shows `agent -.-> tools -.-> agent` but no `__end__` | Router never returns `END` | Inspect the router function | Return `END` (not the string `"end"`) |
| `TypeError: add_node() got an unexpected keyword argument 'retry'` | Version mismatch (`retry` renamed `retry_policy`) | `pip show langgraph` | Use `retry_policy=` on 0.6+/1.x |

---

## 2. Execution

| Symptom | Likely cause | Diagnose | Fix |
|---|---|---|---|
| `GraphRecursionError: Recursion limit of 25 reached without hitting a stop condition` | Tool loop never ends; model keeps calling tools or a router always returns the same node | Stream with `updates` and watch the node sequence | Raise `recursion_limit` only if legitimate; add a step counter in state and a router exit; improve the prompt |
| Node ran twice | It called `interrupt()` and was resumed, or a `RetryPolicy` retried it | `get_state_history` shows two checkpoints for the node | Expected; make pre-interrupt code idempotent |
| `ToolMessage` says `Error: ... Please fix your mistakes.` | Tool raised and `handle_tool_errors=True` turned it into a message | Read the tool's exception in the message | Fine if the model recovers; otherwise validate args in the tool |
| `ValueError: ... tool_calls ... not found in tools` / `KeyError` in `ToolNode` | Model called a tool name not in the `ToolNode` list | Compare `bind_tools` list with `ToolNode` list | Use the same list object for both |
| Parallel nodes see stale data | They ran in the same super-step | Understand Pregel: updates apply after the step | Chain them with an edge if one depends on the other |
| `Send` payload missing parent keys | `Send` nodes receive only the payload | Print the node input | Include needed fields in the `Send` dict |
| `asyncio` `RuntimeError: There is no current event loop` / blocking call warning in `langgraph dev` | Sync I/O inside async graph | Error points at the call | Use async clients or `langgraph dev --allow-blocking` for dev only |
| Output missing keys you expected | `output_schema` restricts returned keys | Check `StateGraph(..., output_schema=)` | Remove or extend the schema |

---

## 3. Persistence and Threads

| Symptom | Likely cause | Diagnose | Fix |
|---|---|---|---|
| `ValueError: Checkpointer requires one or more of the following 'configurable' keys: ['thread_id', 'checkpoint_ns', 'checkpoint_id']` | Compiled with a checkpointer but no `thread_id` passed | Look at `config` | `config={"configurable": {"thread_id": "..."}}` |
| `psycopg.errors.UndefinedTable: relation "checkpoints" does not exist` | `setup()` never ran | `\dt` in psql | Call `saver.setup()` once (migration-safe) |
| `psycopg.OperationalError: connection failed ... SSL` | DSN or network | `psql "$LANGGRAPH_PG_URI"` | Fix `sslmode`, security group, IAM auth token |
| Resumed thread starts from scratch | Different `thread_id` or different `checkpoint_ns`; `MemorySaver` in a restarted process | Compare configs; check saver type | Reuse the exact `thread_id`; use Postgres/Redis |
| `graph.invoke(None, config)` returns immediately | `next` is empty: thread already finished | `graph.get_state(config).next` | Start a new thread or `update_state` then invoke |
| History order confusing | `get_state_history` is newest-first | Print `metadata["step"]` | Sort by step if you need chronological order |
| Checkpoint table growing fast | Large tool outputs stored each step | `select pg_size_pretty(pg_total_relation_size('checkpoint_blobs'))` | Truncate tool output; trim messages; retention job |
| `RedisSaver` fails with `unknown command 'FT.CREATE'` | Redis without RediSearch | `redis-cli MODULE LIST` | Use `redis/redis-stack-server` or Redis 8 |
| Prepared statement errors behind pgbouncer | Transaction pooling + prepared statements | Logs mention `prepared statement ... already exists` | `kwargs={"prepare_threshold": 0, "autocommit": True}` on the pool |

---

## 4. Human-in-the-Loop

| Symptom | Likely cause | Diagnose | Fix |
|---|---|---|---|
| `interrupt()` raises `GraphInterrupt` to my code / no `__interrupt__` in result | Graph compiled **without** a checkpointer | Check `compile()` | Pass a checkpointer; interrupts need persistence |
| Resume value is `None` | Resumed with `graph.invoke(None, cfg)` instead of `Command(resume=...)` | Look at the resume call | `graph.invoke(Command(resume=value), cfg)` |
| Resume value went to the wrong `interrupt()` | Multiple interrupts in one node matched by order; non-deterministic code before them | Review node | Make the node deterministic or use interrupt ids |
| Side effect executed twice | Side effect placed before `interrupt()` in the same node | History shows node ran twice | Move the side effect to the next node |
| `409` / "thread is not waiting" in your API | Decision posted after completion or before interrupt | `get_state(cfg).tasks[0].interrupts` | Check for pending interrupts before resuming |
| Static `interrupt_before` not firing in `langgraph dev` | Server injects its own checkpointer; graph compiled with one | Server logs | Compile without a checkpointer for the server; pass `interrupt_before` in the run request instead |

---

## 5. Streaming

| Symptom | Likely cause | Diagnose | Fix |
|---|---|---|---|
| No tokens in `stream_mode="messages"` | Model does not stream, or node calls `.invoke` on a non-streaming wrapper | Try `model.stream()` directly | Use a streaming-capable model; keep the chat model call inside the node (not inside a plain function with `disable_streaming`) |
| Tokens from the wrong node | Filtering missing | `meta["langgraph_node"]` | Filter by node name |
| `custom` events missing | `get_stream_writer()` called outside a node or in a thread | Where is the call | Call inside the node coroutine; in Python < 3.11 async contexts pass `config` explicitly |
| SSE stops after ~60 s behind ingress | Proxy idle timeout | Ingress logs | Send a `: keepalive` comment every 15 s; raise `proxy-read-timeout` |
| `TypeError: cannot unpack` when iterating | Mixed single/multi `stream_mode` shapes | Check whether `stream_mode` is a list | List -> `(mode, chunk)` tuples; string -> chunk only |

---

## 6. Providers and Tools

| Symptom | Likely cause | Diagnose | Fix |
|---|---|---|---|
| `AccessDeniedException: You don't have access to the model with the specified model ID` | Model access not enabled in region, or IAM denies | `aws bedrock list-foundation-models` in that region; check policy | Enable model access; add `bedrock:InvokeModel*` on the ARN |
| `ValidationException: Invocation of model ID ... with on-demand throughput isn't supported` | Model requires an inference profile | Model id lacks `us.`/`eu.` prefix | Use the cross-region inference profile id |
| `ThrottlingException: Too many requests` | Account TPS/TPM quota | CloudWatch Bedrock metrics | `RetryPolicy` on the node, `InMemoryRateLimiter`, request quota increase, model routing |
| `botocore.exceptions.NoCredentialsError` | No IRSA / profile | `aws sts get-caller-identity` | Fix ServiceAccount annotation or `AWS_PROFILE` |
| `NotImplementedError` from `bind_tools` | Model wrapper lacks tool calling | Provider docs | Use `ChatBedrockConverse` (not legacy `ChatBedrock` with non-tool models) |
| Model ignores tools | Poor docstrings; too many tools; temperature high | Inspect `tool_calls` | Improve descriptions, reduce tool count, `tool_choice="any"` when a call is mandatory |
| `pydantic_core.ValidationError` from `with_structured_output` | Model output did not fit the schema | `include_raw=True` to see raw | Simplify schema, add field descriptions, retry node |
| `httpx.ConnectError` from `query_prometheus` | Wrong `PROM_URL` / NetworkPolicy | `kubectl exec ... curl $PROM_URL/-/ready` | Fix service DNS or allow egress |
| `kubernetes.client.exceptions.ApiException: (403) Forbidden` | RBAC | `kubectl auth can-i list pods -n payments --as=system:serviceaccount:platform-ai:incident-agent` | Grant the verb in a Role/ClusterRole |

---

## 7. Deployment

| Symptom | Likely cause | Diagnose | Fix |
|---|---|---|---|
| `langgraph dev` error `Graph ... compiled with a checkpointer` | Server requires uncompiled-with-checkpointer graphs | Error text | Export the graph compiled without a checkpointer |
| `langgraph build` fails on dependencies | `langgraph.json` `dependencies` path wrong | Check `["."]` and `pyproject` | Fix path; add `dockerfile_lines` for system packages |
| Pod `CrashLoopBackOff` with `ModuleNotFoundError` | Distroless image `PYTHONPATH` | `kubectl logs` | Set `PYTHONPATH` to the copied site-packages |
| `readOnlyRootFilesystem` errors from tiktoken/httpx cache | Libraries write caches | Logs mention `/root/.cache` | Mount an `emptyDir` at `/tmp` and set `TIKTOKEN_CACHE_DIR=/tmp` |
| High memory after hours | Message history unbounded; compiled graphs cached per tenant | Heap profile | Trim messages; LRU on compiled graphs |

---

## 8. Diagnostic Commands

```bash
# versions
pip show langgraph langchain-core langchain-aws | grep -E 'Name|Version'

# topology
python -c "from labs.incident_agent import graph; print(graph.get_graph().draw_mermaid())"

# full trace of one run
python - <<'EOF'
from langchain_core.globals import set_debug
set_debug(True)
from labs.incident_agent import build
from langgraph.checkpoint.memory import MemorySaver
g = build().compile(checkpointer=MemorySaver())
for ev in g.stream({"messages": [], "alert": {"alertname":"HighErrorRate","service":"payments-api","namespace":"payments","severity":"critical"},
                    "plan": None, "approved": None, "remediated": False, "report": ""},
                   {"configurable": {"thread_id": "dbg"}}, stream_mode="debug"):
    print(ev["type"], ev.get("payload", {}).get("name"))
EOF

# thread state
python -c "import os; from langgraph.checkpoint.postgres import PostgresSaver; from labs.incident_agent import build
with PostgresSaver.from_conn_string(os.environ['LANGGRAPH_PG_URI']) as s:
    g = build().compile(checkpointer=s); snap = g.get_state({'configurable': {'thread_id': 'inc-0002'}})
    print(snap.next, snap.tasks and snap.tasks[0].interrupts)"

# database
docker exec lg-postgres psql -U langgraph -d langgraph -c "select thread_id, count(*) from checkpoints group by 1 order by 2 desc limit 10;"

# provider
aws sts get-caller-identity
aws bedrock list-inference-profiles --region "$AWS_REGION" --query 'inferenceProfileSummaries[].inferenceProfileId'

# cluster access from the pod
kubectl exec -n platform-ai deploy/incident-agent -- python -c "from kubernetes import client, config; config.load_incluster_config(); print(len(client.CoreV1Api().list_namespaced_pod('payments').items))"
```

---

## 9. Debugging Workflow

1. **Reproduce offline.** `OFFLINE=1` with the scripted model isolates graph logic from provider behaviour.
2. **Draw it.** `draw_mermaid()` catches most wiring mistakes before running anything.
3. **Stream `updates`.** Read the node sequence; a repeated node or a missing one tells you where the router or edge is wrong.
4. **Inspect checkpoints.** `get_state_history` shows the exact state before the failing step; `update_state` lets you patch it and continue without re-running the model.
5. **Turn on `set_debug(True)` or `stream_mode="debug"`** for inputs/outputs of every runnable.
6. **Look at the trace** (LangSmith or Tempo) for latency and token counts per node.
7. **Only then** touch the prompt. Most "model is dumb" issues are missing tool descriptions, oversized context, or a router bug.

Next: [07-interview-questions.md](07-interview-questions.md).
