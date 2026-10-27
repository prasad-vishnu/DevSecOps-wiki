# LangGraph: Installation and Setup

| | |
|---|---|
| **Cluster** | 03 Agentic Frameworks |
| **Role** | Anchor |
| **Prerequisites** | Python 3.12+, `uv` or `pip`, Docker (for Postgres/Redis), AWS credentials via SSO or IRSA for Bedrock (optional: labs have an offline mode) |
| **Target version** | LangGraph 1.x, langchain-core 1.x, Python 3.12+ |
| **Capstone phase** | Phase 6 (Months 21-24) |

> After this file you will have a reproducible Python environment with LangGraph, the Bedrock and Anthropic model adapters, Postgres and Redis checkpointers, the LangGraph CLI, and LangSmith tracing wired up; you will have run a two-node graph end to end and know how to check which versions you are documenting against.

---

## 1. Package Map

LangGraph is split into small packages. Install only what you use.

| Package | Purpose | Needed for |
|---|---|---|
| `langgraph` | Graph runtime, `StateGraph`, prebuilt agents, `MemorySaver` | Everything |
| `langgraph-checkpoint-postgres` | `PostgresSaver`, `AsyncPostgresSaver`, `PostgresStore` | Production persistence |
| `langgraph-checkpoint-redis` | `RedisSaver`, `AsyncRedisSaver` | Low-latency persistence |
| `langgraph-cli[inmem]` | `langgraph dev`, `langgraph build`, `langgraph up` | LangGraph Server |
| `langgraph-sdk` | Python client for LangGraph Server | Calling a deployed graph |
| `langgraph-supervisor`, `langgraph-swarm` | Multi-agent reference implementations | Multi-agent labs |
| `langchain` / `langchain-core` | Messages, tools, `init_chat_model` | Everything |
| `langchain-aws` | `ChatBedrockConverse`, `BedrockEmbeddings` | Bedrock |
| `langchain-anthropic`, `langchain-openai`, `langchain-google-vertexai` | Other providers | Provider swapping |
| `langchain-mcp-adapters` | MCP servers as LangChain tools | MCP labs |
| `langsmith` | Tracing client (installed transitively) | Tracing |

---

## 2. Install

### 2.1 Linux / macOS with `uv` (recommended)

```bash
mkdir -p ~/capstone/agent && cd ~/capstone/agent
uv init --python 3.12
uv add "langgraph>=1.0,<2" "langchain>=1.0,<2" "langchain-aws>=1.0" \
       "langchain-anthropic>=1.0" "langgraph-checkpoint-postgres>=2.0" \
       "langgraph-checkpoint-redis>=0.1" "langchain-mcp-adapters>=0.1" \
       "pydantic>=2.7" "httpx>=0.27" "kubernetes>=30" "psycopg[binary,pool]>=3.2"
uv add --dev "pytest>=8" "pytest-asyncio>=0.23" "ruff" "langgraph-cli[inmem]>=0.3"
```

### 2.2 Linux / macOS with `pip` and a pinned requirements file

```bash
python3.12 -m venv .venv && source .venv/bin/activate
cat > requirements.txt <<'EOF'
langgraph>=1.0,<2
langchain>=1.0,<2
langchain-core>=1.0,<2
langchain-aws>=1.0
langchain-anthropic>=1.0
langgraph-checkpoint-postgres>=2.0
langgraph-checkpoint-redis>=0.1
langchain-mcp-adapters>=0.1
pydantic>=2.7
httpx>=0.27
kubernetes>=30
psycopg[binary,pool]>=3.2
EOF
pip install --upgrade pip && pip install -r requirements.txt
```

### 2.3 Windows (PowerShell)

```powershell
py -3.12 -m venv .venv
.\.venv\Scripts\Activate.ps1
pip install --upgrade pip
pip install "langgraph>=1.0,<2" "langchain>=1.0,<2" "langchain-aws>=1.0" "langgraph-checkpoint-postgres>=2.0" "psycopg[binary,pool]>=3.2"
```

Windows note: `psycopg[binary]` ships wheels; `AsyncPostgresSaver` requires an asyncio event loop policy that supports sockets (`asyncio.WindowsSelectorEventLoopPolicy()`), set it before creating the loop if you hit `ProactorEventLoop` errors.

### 2.4 Container (the way the capstone ships it)

```dockerfile
# syntax=docker/dockerfile:1.7
FROM python:3.12-slim AS build
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir --prefix=/install -r requirements.txt

FROM gcr.io/distroless/python3-debian12:nonroot
WORKDIR /app
COPY --from=build /install /usr/local
COPY src/ ./src/
ENV PYTHONPATH=/usr/local/lib/python3.12/site-packages PYTHONUNBUFFERED=1
USER nonroot
ENTRYPOINT ["python", "-m", "uvicorn", "src.app:app", "--host", "0.0.0.0", "--port", "8080"]
```

```bash
docker build -t ghcr.io/example/incident-agent:dev .
trivy image --severity HIGH,CRITICAL ghcr.io/example/incident-agent:dev
```

---

## 3. Check the Installed Versions

Document the version you are running; the API has moved (for example `retry=` became `retry_policy=` on `add_node`, `MemorySaver` gained the alias `InMemorySaver`).

```bash
pip show langgraph langchain-core langchain-aws | grep -E '^(Name|Version)'
python -c "import langgraph, langchain_core; print('langgraph', langgraph.__version__); print('langchain_core', langchain_core.__version__)"
```

Expected output (your patch numbers will differ):

```text
Name: langgraph
Version: 1.0.x
Name: langchain-core
Version: 1.0.x
Name: langchain-aws
Version: 1.0.x
langgraph 1.0.x
langchain_core 1.0.x
```

If `langgraph.__version__` is missing on your release, use `importlib.metadata.version("langgraph")`.

---

## 4. Authentication for Model Providers

Never put API keys in code or in the image. Pick one of these per environment.

### 4.1 Amazon Bedrock (default for the capstone)

Local development uses AWS SSO; the cluster uses IRSA or EKS Pod Identity.

```bash
aws sso login --profile capstone-dev
export AWS_PROFILE=capstone-dev
export AWS_REGION=<REGION>
export BEDROCK_MODEL_ID=us.anthropic.claude-sonnet-4-20250514-v1:0
aws bedrock list-foundation-models --region "$AWS_REGION" --query 'modelSummaries[?contains(modelId, `claude`)].modelId' --output table
```

Minimum IAM policy for the agent's role (scope the resource to the inference profile and model ARNs you actually use):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["bedrock:InvokeModel", "bedrock:InvokeModelWithResponseStream"],
      "Resource": [
        "arn:aws:bedrock:<REGION>:<ACCOUNT_ID>:inference-profile/us.anthropic.claude-sonnet-4-20250514-v1:0",
        "arn:aws:bedrock:*::foundation-model/anthropic.claude-sonnet-4-20250514-v1:0"
      ]
    }
  ]
}
```

### 4.2 Anthropic API directly

```bash
export ANTHROPIC_API_KEY=<ANTHROPIC_API_KEY>   # from a secrets manager, never committed
```

### 4.3 Offline (no credentials)

Every lab in this folder has an `OFFLINE=1` mode using a fake tool-calling model defined in the labs. Use it on planes, in CI, and when you only want to debug graph structure.

---

## 5. LangSmith Tracing (optional but recommended)

```bash
export LANGSMITH_TRACING=true
export LANGSMITH_API_KEY=<LANGSMITH_API_KEY>
export LANGSMITH_PROJECT=capstone-incident-agent
export LANGSMITH_ENDPOINT=https://api.smith.langchain.com   # or your self-hosted endpoint
```

Older docs use `LANGCHAIN_TRACING_V2` / `LANGCHAIN_API_KEY`; both prefixes still work in 1.x but prefer `LANGSMITH_*`.

---

## 6. Local Persistence Back Ends

```bash
# Postgres for PostgresSaver and PostgresStore
docker run --name lg-postgres --rm -d -p 5432:5432 \
  -e POSTGRES_USER=langgraph -e POSTGRES_PASSWORD=<DEV_ONLY_PASSWORD> -e POSTGRES_DB=langgraph \
  postgres:16-alpine

# Redis for RedisSaver (RediSearch module required: use redis-stack)
docker run --name lg-redis --rm -d -p 6379:6379 redis/redis-stack-server:latest

export LANGGRAPH_PG_URI="postgresql://langgraph:<DEV_ONLY_PASSWORD>@localhost:5432/langgraph?sslmode=disable"
export LANGGRAPH_REDIS_URI="redis://localhost:6379"
```

`sslmode=disable` is a local shortcut. In the cluster use `sslmode=require` and IAM database authentication or a Vault-issued dynamic credential (see Cluster 08, gate 7).

Create the checkpoint tables once:

```bash
python - <<'EOF'
import os
from langgraph.checkpoint.postgres import PostgresSaver
with PostgresSaver.from_conn_string(os.environ["LANGGRAPH_PG_URI"]) as saver:
    saver.setup()
    print("checkpoint tables ready")
EOF
```

Expected output:

```text
checkpoint tables ready
```

```bash
docker exec lg-postgres psql -U langgraph -d langgraph -c '\dt'
```

```text
                 List of relations
 Schema |         Name          | Type  |   Owner
--------+-----------------------+-------+-----------
 public | checkpoint_blobs      | table | langgraph
 public | checkpoint_migrations | table | langgraph
 public | checkpoint_writes     | table | langgraph
 public | checkpoints           | table | langgraph
```

---

## 7. First Run: A Two-Node Graph

Save as `hello_graph.py`:

```python
"""Smallest useful LangGraph: typed state, two nodes, one conditional edge, checkpointed."""
from typing import Literal, TypedDict

from langgraph.checkpoint.memory import MemorySaver
from langgraph.graph import END, START, StateGraph


class State(TypedDict):
    alert: str
    severity: str
    action: str


def classify(state: State) -> dict:
    sev = "critical" if "HighErrorRate" in state["alert"] else "warning"
    return {"severity": sev}


def page(state: State) -> dict:
    return {"action": f"page on-call for {state['alert']}"}


def ticket(state: State) -> dict:
    return {"action": f"open ticket for {state['alert']}"}


def route(state: State) -> Literal["page", "ticket"]:
    return "page" if state["severity"] == "critical" else "ticket"


builder = StateGraph(State)
builder.add_node("classify", classify)
builder.add_node("page", page)
builder.add_node("ticket", ticket)
builder.add_edge(START, "classify")
builder.add_conditional_edges("classify", route)
builder.add_edge("page", END)
builder.add_edge("ticket", END)

graph = builder.compile(checkpointer=MemorySaver())

if __name__ == "__main__":
    config = {"configurable": {"thread_id": "demo-1"}}
    result = graph.invoke({"alert": "HighErrorRate payments-api", "severity": "", "action": ""}, config)
    print(result)
    print("checkpoints:", len(list(graph.get_state_history(config))))
    print(graph.get_graph().draw_mermaid())
```

```bash
python hello_graph.py
```

Expected output:

```text
{'alert': 'HighErrorRate payments-api', 'severity': 'critical', 'action': 'page on-call for HighErrorRate payments-api'}
checkpoints: 4
%%{init: {'flowchart': {'curve': 'linear'}}}%%
graph TD;
	__start__([<p>__start__</p>]):::first
	classify(classify)
	page(page)
	ticket(ticket)
	__end__([<p>__end__</p>]):::last
	__start__ --> classify;
	classify -.-> page;
	classify -.-> ticket;
	page --> __end__;
	ticket --> __end__;
	...
```

Four checkpoints: the input, after `classify`, after `page`, and the final state. That checkpoint log is the whole point of the framework.

---

## 8. LangGraph CLI and Server (local)

```bash
pip install -U "langgraph-cli[inmem]"
langgraph --version
```

Create `langgraph.json` beside your code:

```json
{
  "dependencies": ["."],
  "graphs": {
    "incident_agent": "./hello_graph.py:graph"
  },
  "env": ".env",
  "python_version": "3.12"
}
```

```bash
langgraph dev --port 2024
```

Expected output (abbreviated):

```text
Ready!
- API: http://127.0.0.1:2024
- Docs: http://127.0.0.1:2024/docs
- LangGraph Studio Web UI: https://smith.langchain.com/studio/?baseUrl=http://127.0.0.1:2024
```

Call it:

```bash
curl -s http://127.0.0.1:2024/assistants/search -X POST -H 'content-type: application/json' -d '{}' | python -m json.tool | head -20
```

Note: a compiled graph given to `langgraph dev` must be compiled **without** a checkpointer; the server injects its own. The labs show the `if __name__` pattern that keeps both paths working.

---

## 9. Verification Checklist

```bash
python -c "from langgraph.graph import StateGraph, START, END; from langgraph.prebuilt import create_react_agent, ToolNode; from langgraph.types import interrupt, Command, Send; print('imports ok')"
python -c "from langgraph.checkpoint.postgres import PostgresSaver; from langgraph.checkpoint.redis import RedisSaver; print('checkpointers ok')"
python -c "from langchain.chat_models import init_chat_model; m = init_chat_model(__import__('os').environ['BEDROCK_MODEL_ID'], model_provider='bedrock_converse'); print(m.invoke('Reply with the single word pong').content)"
```

Expected:

```text
imports ok
checkpointers ok
pong
```

If the third command fails with `AccessDeniedException`, request model access in the Bedrock console for the region, or check the IAM policy above. See [06-troubleshooting.md](06-troubleshooting.md).

---

## 10. Cleanup

```bash
docker rm -f lg-postgres lg-redis
deactivate 2>/dev/null || true
```

Next: [02-core-concepts.md](02-core-concepts.md).
