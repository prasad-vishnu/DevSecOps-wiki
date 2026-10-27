# MCP: Hands-On Labs

| | |
|---|---|
| **Cluster** | 10 AI Infrastructure, RAG, MCP & Evals |
| **Role** | Anchor |
| **Prerequisites** | [01-installation-and-setup.md](01-installation-and-setup.md), [02-core-concepts.md](02-core-concepts.md), Docker, `kind`, `kubectl`, a Bedrock or other chat model for Labs 5-6 |
| **Target version** | MCP spec `2025-06-18`; `mcp` 1.12+; Kubernetes 1.31+; Prometheus 2.5x/3.x; Qdrant 1.12+ |
| **Capstone phase** | Phase 6 (Months 21-24) |

> Seven progressive labs that end with the three capstone MCP servers (Prometheus, Kubernetes, Runbook RAG) running on a kind cluster over Streamable HTTP, consumed by a LangGraph agent with a human-approval gate and by a Strands agent, registered in Claude Code, and covered by pytest contract tests. All code is complete and runnable; placeholders are marked `<LIKE_THIS>`.

Repository layout the labs build (mirrors the capstone):

```text
capstone/
└── ai/mcp-servers/
    ├── prom-mcp/      server.py  pyproject.toml  Dockerfile  tests/
    ├── k8s-mcp/       server.py  pyproject.toml  Dockerfile  rbac/  tests/
    ├── runbook-mcp/   server.py  ingest.py  pyproject.toml  Dockerfile
    └── agent/         langgraph_agent.py  strands_agent.py
```

---

## Lab 1: Protocol by Hand

**Objective.** See every lifecycle message of MCP without an SDK client, then with the Inspector.

**Prerequisites.** `hello-mcp` from [01-installation-and-setup.md](01-installation-and-setup.md).

**Time.** 30 minutes.

### Steps

1. Create `handshake.jsonl` with the nine lines from [03-commands-and-cheatsheet.md](03-commands-and-cheatsheet.md#6-raw-json-rpc-snippets-debugging) section 6 (adjust tool names to `add`, resource to `greeting://world`, prompt to `review_code`).

2. Pipe it into the server and separate stdout (protocol) from stderr (logs):

   ```bash
   cat handshake.jsonl | uv run server.py 2>server.err | jq -c '{id, method, ok: (.result != null), err: .error.message}'
   ```

   Expected output:

   ```text
   {"id":1,"method":null,"ok":true,"err":null}
   {"id":2,"method":null,"ok":true,"err":null}
   {"id":3,"method":null,"ok":true,"err":null}
   {"id":4,"method":null,"ok":true,"err":null}
   {"id":5,"method":null,"ok":true,"err":null}
   ...
   ```

3. Break it on purpose. Edit `server.py` and add `print("booting")` at module import, re-run step 2.

   Expected output:

   ```text
   jq: error (at <stdin>:0): Cannot index string with string "id"
   ```

   The first stdout line is `booting`, which is not JSON. This is the single most common MCP bug. Remove the `print`.

4. Send a request **before** `initialize` (reorder the file so `tools/list` is first). Observe the SDK's error response:

   ```json
   {"jsonrpc":"2.0","id":2,"error":{"code":-32602,"message":"Received request before initialization was complete"}}
   ```

5. Open the Inspector (`npx @modelcontextprotocol/inspector uv run server.py`), connect, and watch the **History** pane while you call `add`. Match each entry to a row in the method table.

### Verify

You can name, in order, the three messages that make up a handshake, and explain why `notifications/initialized` has no `id`.

### Cleanup

None.

### What you learned

JSON-RPC framing, the strict ordering of the lifecycle, and why stdout is sacred in stdio servers.

---

## Lab 2: Prometheus MCP Server

**Objective.** Build `prom-mcp` with four read-only tools and one resource, run it against a local Prometheus, and call it from a Python client.

**Prerequisites.** Docker. Lab 1.

**Time.** 90 minutes.

### Steps

1. Start Prometheus with a self-scrape and a demo recording rule:

   `prometheus.yml`

   ```yaml
   global:
     scrape_interval: 15s
   rule_files: ["rules.yml"]
   scrape_configs:
     - job_name: prometheus
       static_configs:
         - targets: ["localhost:9090"]
   ```

   `rules.yml`

   ```yaml
   groups:
     - name: demo
       rules:
         - alert: DemoAlwaysFiring
           expr: vector(1)
           for: 0m
           labels: { severity: warning, service: api }
           annotations: { summary: "Demo alert for MCP lab" }
   ```

   ```bash
   docker run -d --name prom -p 9090:9090 \
     -v "$PWD/prometheus.yml:/etc/prometheus/prometheus.yml:ro" \
     -v "$PWD/rules.yml:/etc/prometheus/rules.yml:ro" prom/prometheus:v3.4.1
   curl -s 'http://localhost:9090/api/v1/query?query=up' | jq .status
   ```

   Expected output: `"success"`.

2. Create the project:

   ```bash
   mkdir -p ~/capstone/ai/mcp-servers/prom-mcp && cd $_
   uv init --python 3.12 && uv add "mcp[cli]>=1.12" httpx pydantic
   ```

3. Write `server.py`:

   ```python
   """prom-mcp: read-only Prometheus access for the incident agent.

   Tools:    query_instant, query_range, list_alerts, get_slo_burn_rate
   Resource: prom://targets
   Prompt:   triage_alert
   """
   from __future__ import annotations

   import logging
   import os
   import sys
   from contextlib import asynccontextmanager
   from collections.abc import AsyncIterator
   from dataclasses import dataclass
   from datetime import datetime, timedelta, timezone
   from typing import Literal

   import httpx
   from pydantic import BaseModel, Field
   from mcp.server.fastmcp import Context, FastMCP
   from mcp.types import ToolAnnotations

   # ---- logging: stderr only, never stdout -------------------------------------
   logging.basicConfig(stream=sys.stderr, level=os.getenv("LOG_LEVEL", "INFO"),
                       format="%(asctime)s %(levelname)s %(name)s %(message)s")
   log = logging.getLogger("prom-mcp")

   PROMETHEUS_URL = os.getenv("PROMETHEUS_URL", "http://localhost:9090")
   MAX_SAMPLES = int(os.getenv("MAX_SAMPLES", "500"))      # output size guard
   QUERY_TIMEOUT = float(os.getenv("QUERY_TIMEOUT", "15"))

   READ_ONLY = ToolAnnotations(readOnlyHint=True, destructiveHint=False,
                               idempotentHint=True, openWorldHint=False)


   @dataclass
   class AppContext:
       http: httpx.AsyncClient


   @asynccontextmanager
   async def lifespan(_: FastMCP) -> AsyncIterator[AppContext]:
       headers = {}
       if token := os.getenv("PROMETHEUS_BEARER_TOKEN"):      # e.g. read-only SA token
           headers["Authorization"] = f"Bearer {token}"
       async with httpx.AsyncClient(base_url=PROMETHEUS_URL, timeout=QUERY_TIMEOUT, headers=headers) as http:
           log.info("connected to %s", PROMETHEUS_URL)
           yield AppContext(http=http)


   mcp = FastMCP(
       "prom-mcp",
       instructions=("Read-only Prometheus. Use query_instant for current values, query_range for trends, "
                     "list_alerts for firing alerts, get_slo_burn_rate before deciding severity."),
       lifespan=lifespan,
       host=os.getenv("HOST", "127.0.0.1"), port=int(os.getenv("PORT", "8000")),
       stateless_http=os.getenv("STATELESS", "true").lower() == "true",
   )


   # ---- models -------------------------------------------------------------------
   class Sample(BaseModel):
       metric: dict[str, str]
       value: float | None = None                      # instant
       values: list[tuple[float, float]] | None = None  # range: (ts, value)


   class QueryResult(BaseModel):
       result_type: str
       samples: list[Sample]
       truncated: bool = False
       executed_at: str


   class Alert(BaseModel):
       name: str
       state: str
       severity: str | None
       labels: dict[str, str]
       summary: str | None
       active_at: str | None


   class BurnRate(BaseModel):
       service: str
       slo: float
       burn_1h: float
       burn_6h: float
       page: bool = Field(description="True when both windows exceed the 14.4x fast-burn threshold")
       ticket: bool = Field(description="True when 6h burn > 6x (slow burn)")


   # ---- helpers --------------------------------------------------------------------
   async def _prom_get(ctx: Context, path: str, **params) -> dict:
       http: httpx.AsyncClient = ctx.request_context.lifespan_context.http
       r = await http.get(path, params=params)
       body = r.json()
       if r.status_code >= 400 or body.get("status") != "success":
           raise ValueError(f"Prometheus {r.status_code}: {body.get('error', 'unknown error')}")
       return body["data"]


   def _to_samples(data: dict) -> tuple[list[Sample], bool]:
       raw = data["result"]
       truncated = len(raw) > MAX_SAMPLES
       out: list[Sample] = []
       for item in raw[:MAX_SAMPLES]:
           if "value" in item:
               out.append(Sample(metric=item["metric"], value=float(item["value"][1])))
           else:
               out.append(Sample(metric=item["metric"],
                                 values=[(float(t), float(v)) for t, v in item["values"]]))
       return out, truncated


   # ---- tools ------------------------------------------------------------------------
   @mcp.tool(annotations=READ_ONLY)
   async def query_instant(query: str, ctx: Context) -> QueryResult:
       """Evaluate a PromQL expression now. Use for 'what is X right now'. Returns at most MAX_SAMPLES series."""
       await ctx.debug(f"instant: {query}")
       data = await _prom_get(ctx, "/api/v1/query", query=query)
       samples, truncated = _to_samples(data)
       return QueryResult(result_type=data["resultType"], samples=samples, truncated=truncated,
                          executed_at=datetime.now(timezone.utc).isoformat())


   @mcp.tool(annotations=READ_ONLY)
   async def query_range(query: str, ctx: Context, minutes: int = 60, step: str = "60s") -> QueryResult:
       """Evaluate PromQL over the last `minutes` minutes at `step` resolution. Use for trends and 'when did it start'."""
       if minutes > 24 * 60:
           raise ValueError("minutes must be <= 1440; use a recording rule for longer windows")
       end = datetime.now(timezone.utc)
       start = end - timedelta(minutes=minutes)
       await ctx.report_progress(0, 2, "querying")
       data = await _prom_get(ctx, "/api/v1/query_range", query=query,
                              start=start.timestamp(), end=end.timestamp(), step=step)
       await ctx.report_progress(2, 2, "done")
       samples, truncated = _to_samples(data)
       return QueryResult(result_type=data["resultType"], samples=samples, truncated=truncated,
                          executed_at=end.isoformat())


   @mcp.tool(annotations=READ_ONLY)
   async def list_alerts(ctx: Context, state: Literal["firing", "pending", "all"] = "firing") -> list[Alert]:
       """List Prometheus alerts, firing by default. Call first when triaging an incident."""
       data = await _prom_get(ctx, "/api/v1/alerts")
       out = []
       for a in data["alerts"]:
           if state != "all" and a["state"] != state:
               continue
           out.append(Alert(name=a["labels"].get("alertname", "?"), state=a["state"],
                            severity=a["labels"].get("severity"), labels=a["labels"],
                            summary=a.get("annotations", {}).get("summary"), active_at=a.get("activeAt")))
       return out


   @mcp.tool(annotations=READ_ONLY)
   async def get_slo_burn_rate(service: str, ctx: Context, slo: float = 0.999) -> BurnRate:
       """Multi-window error-budget burn rate for an HTTP service using http_requests_total{service=...}.
       page=True means a 14.4x fast burn (1h and 6h both over threshold) per the SRE workbook."""
       budget = 1 - slo
       async def burn(window: str) -> float:
           q = (f'sum(rate(http_requests_total{{service="{service}",code=~"5.."}}[{window}])) / '
                f'sum(rate(http_requests_total{{service="{service}"}}[{window}]))')
           data = await _prom_get(ctx, "/api/v1/query", query=q)
           if not data["result"]:
               return 0.0
           return float(data["result"][0]["value"][1]) / budget
       b1, b6 = await burn("1h"), await burn("6h")
       await ctx.info(f"burn {service}: 1h={b1:.2f} 6h={b6:.2f}")
       return BurnRate(service=service, slo=slo, burn_1h=b1, burn_6h=b6,
                       page=(b1 > 14.4 and b6 > 14.4), ticket=(b6 > 6))


   # ---- resources ---------------------------------------------------------------------
   @mcp.resource("prom://targets", mime_type="application/json")
   async def targets() -> str:
       """Active scrape targets and their health."""
       async with httpx.AsyncClient(base_url=PROMETHEUS_URL, timeout=QUERY_TIMEOUT) as http:
           r = await http.get("/api/v1/targets", params={"state": "active"})
           r.raise_for_status()
           import json
           compact = [{"job": t["labels"].get("job"), "instance": t["labels"].get("instance"),
                       "health": t["health"], "lastError": t.get("lastError", "")}
                      for t in r.json()["data"]["activeTargets"]]
           return json.dumps(compact)


   # ---- prompts -----------------------------------------------------------------------
   @mcp.prompt(title="Triage an alert")
   def triage_alert(alertname: str, namespace: str = "default") -> str:
       """Structured triage plan for a firing alert."""
       return (f"Alert `{alertname}` is firing in namespace `{namespace}`.\n"
               "1. Call list_alerts to confirm and gather labels.\n"
               "2. Call get_slo_burn_rate for the affected service; decide page vs ticket.\n"
               "3. Use query_range (60 min) on the alert expression to find the onset time.\n"
               "4. Correlate with Kubernetes events and logs via k8s-mcp.\n"
               "5. Search runbooks via runbook-mcp before proposing any change.\n"
               "Report: impact, onset, probable cause, evidence, proposed action (ask before destructive actions).")


   if __name__ == "__main__":
       mcp.run(transport=os.getenv("MCP_TRANSPORT", "stdio"))
   ```

4. Test with the Inspector CLI:

   ```bash
   npx @modelcontextprotocol/inspector --cli uv run server.py --method tools/call \
     --tool-name list_alerts --tool-arg state=firing | jq '.structuredContent'
   ```

   Expected output:

   ```json
   {"result":[{"name":"DemoAlwaysFiring","state":"firing","severity":"warning","labels":{"alertname":"DemoAlwaysFiring","service":"api","severity":"warning"},"summary":"Demo alert for MCP lab","active_at":"2025-06-18T09:00:12.3Z"}]}
   ```

5. Write a Python client `client_demo.py` that exercises progress and the resource:

   ```python
   import asyncio
   from mcp import ClientSession, StdioServerParameters
   from mcp.client.stdio import stdio_client

   async def main():
       params = StdioServerParameters(command="uv", args=["run", "server.py"],
                                      env={"PROMETHEUS_URL": "http://localhost:9090", "LOG_LEVEL": "DEBUG"})
       async with stdio_client(params) as (r, w):
           async with ClientSession(r, w, logging_callback=lambda p: print("LOG", p.level, p.data)) as s:
               init = await s.initialize()
               print("server:", init.serverInfo.name, "proto:", init.protocolVersion)
               res = await s.call_tool("query_range", {"query": "up", "minutes": 5, "step": "30s"},
                                       progress_callback=lambda p, t, m: print(f"progress {p}/{t} {m}"))
               print("series:", len(res.structuredContent["samples"]))
               tg = await s.read_resource("prom://targets")
               print("targets:", tg.contents[0].text[:80])
               pr = await s.get_prompt("triage_alert", {"alertname": "DemoAlwaysFiring"})
               print("prompt:", pr.messages[0].content.text.splitlines()[0])

   asyncio.run(main())
   ```

   ```bash
   uv run client_demo.py
   ```

   Expected output:

   ```text
   server: prom-mcp proto: 2025-06-18
   progress 0/2 querying
   progress 2/2 done
   series: 1
   targets: [{"job": "prometheus", "instance": "localhost:9090", "health": "up", "lastError": ""}]
   prompt: Alert `DemoAlwaysFiring` is firing in namespace `default`.
   ```

### Verify

`list_alerts` returns the demo alert; `query_range` emits two progress notifications; the resource returns compact JSON.

### Cleanup

`docker rm -f prom` (keep it if you continue to Lab 6).

### What you learned

Lifespan-managed connection pools, read-only annotations, structured output from Pydantic models, output-size guards, progress and logging through `Context`.

---

## Lab 3: Kubernetes MCP Server with Least Privilege and a Destructive Tool

**Objective.** Build `k8s-mcp` that is read-only by default via an RBAC-scoped kubeconfig, and exposes `rollout_restart` as a destructive tool that requires approval (via Elicitation when the host supports it, otherwise refuses unless `ALLOW_WRITE=true`).

**Prerequisites.** `kind`, `kubectl`. Lab 2.

**Time.** 2 hours.

### Steps

1. Create a cluster and a sample workload:

   ```bash
   kind create cluster --name capstone
   kubectl create namespace api
   kubectl -n api create deployment api --image=ghcr.io/nginx/nginx-unprivileged:1.27 --replicas=2
   kubectl -n api rollout status deployment/api
   ```

2. Create the least-privilege identity. `rbac/reader.yaml`:

   ```yaml
   apiVersion: v1
   kind: ServiceAccount
   metadata: { name: k8s-mcp-reader, namespace: api }
   ---
   apiVersion: rbac.authorization.k8s.io/v1
   kind: Role
   metadata: { name: k8s-mcp-reader, namespace: api }
   rules:
     - apiGroups: [""]
       resources: ["pods", "pods/log", "events", "services", "endpoints"]
       verbs: ["get", "list", "watch"]
     - apiGroups: ["apps"]
       resources: ["deployments", "replicasets"]
       verbs: ["get", "list", "watch"]
   ---
   apiVersion: rbac.authorization.k8s.io/v1
   kind: RoleBinding
   metadata: { name: k8s-mcp-reader, namespace: api }
   roleRef: { apiGroup: rbac.authorization.k8s.io, kind: Role, name: k8s-mcp-reader }
   subjects:
     - { kind: ServiceAccount, name: k8s-mcp-reader, namespace: api }
   ```

   And a **separate** writer role used only when `ALLOW_WRITE=true` (`rbac/writer.yaml`):

   ```yaml
   apiVersion: v1
   kind: ServiceAccount
   metadata: { name: k8s-mcp-writer, namespace: api }
   ---
   apiVersion: rbac.authorization.k8s.io/v1
   kind: Role
   metadata: { name: k8s-mcp-writer, namespace: api }
   rules:
     - apiGroups: ["apps"]
       resources: ["deployments"]
       verbs: ["get", "patch"]          # patch only: rollout restart, no delete/create
   ---
   apiVersion: rbac.authorization.k8s.io/v1
   kind: RoleBinding
   metadata: { name: k8s-mcp-writer, namespace: api }
   roleRef: { apiGroup: rbac.authorization.k8s.io, kind: Role, name: k8s-mcp-writer }
   subjects:
     - { kind: ServiceAccount, name: k8s-mcp-writer, namespace: api }
   ```

   ```bash
   kubectl apply -f rbac/reader.yaml -f rbac/writer.yaml
   # Short-lived token (1h) and a scoped kubeconfig for local stdio use
   TOKEN=$(kubectl -n api create token k8s-mcp-reader --duration=1h)
   SERVER=$(kubectl config view --minify -o jsonpath='{.clusters[0].cluster.server}')
   kubectl config view --minify --raw -o jsonpath='{.clusters[0].cluster.certificate-authority-data}' | base64 -d > ca.crt
   kubectl config --kubeconfig=reader.kubeconfig set-cluster kind --server="$SERVER" --certificate-authority=ca.crt --embed-certs
   kubectl config --kubeconfig=reader.kubeconfig set-credentials reader --token="$TOKEN"
   kubectl config --kubeconfig=reader.kubeconfig set-context reader --cluster=kind --user=reader --namespace=api
   kubectl config --kubeconfig=reader.kubeconfig use-context reader
   kubectl --kubeconfig=reader.kubeconfig auth can-i delete pods
   ```

   Expected output: `no`.

3. Project and dependencies:

   ```bash
   mkdir -p ~/capstone/ai/mcp-servers/k8s-mcp && cd $_
   uv init --python 3.12 && uv add "mcp[cli]>=1.12" kubernetes pydantic
   ```

4. `server.py`:

   ```python
   """k8s-mcp: namespace-scoped Kubernetes access. Read-only unless ALLOW_WRITE=true.

   Tools: list_pods, describe_pod, get_logs, get_events, rollout_restart (destructive)
   Resource template: k8s://{namespace}/pods/{pod}/logs
   """
   from __future__ import annotations

   import logging
   import os
   import sys
   from datetime import datetime, timezone
   from typing import Literal

   from kubernetes import client, config
   from kubernetes.client.exceptions import ApiException
   from pydantic import BaseModel, Field
   from mcp.server.fastmcp import Context, FastMCP
   from mcp.types import ToolAnnotations

   logging.basicConfig(stream=sys.stderr, level=os.getenv("LOG_LEVEL", "INFO"))
   log = logging.getLogger("k8s-mcp")

   ALLOWED_NAMESPACES = set(filter(None, os.getenv("ALLOWED_NAMESPACES", "api").split(",")))
   ALLOW_WRITE = os.getenv("ALLOW_WRITE", "false").lower() == "true"
   MAX_LOG_BYTES = int(os.getenv("MAX_LOG_BYTES", "32768"))

   try:
       config.load_incluster_config()
       log.info("in-cluster config")
   except config.ConfigException:
       config.load_kube_config(config_file=os.getenv("KUBECONFIG"))
       log.info("kubeconfig %s", os.getenv("KUBECONFIG", "~/.kube/config"))

   core = client.CoreV1Api()
   apps = client.AppsV1Api()

   READ_ONLY = ToolAnnotations(readOnlyHint=True, destructiveHint=False, idempotentHint=True, openWorldHint=False)
   DESTRUCTIVE = ToolAnnotations(readOnlyHint=False, destructiveHint=True, idempotentHint=False, openWorldHint=False)

   mcp = FastMCP("k8s-mcp",
                 instructions=(f"Kubernetes access limited to namespaces {sorted(ALLOWED_NAMESPACES)}. "
                               "All tools are read-only except rollout_restart, which requires explicit human approval."),
                 host=os.getenv("HOST", "127.0.0.1"), port=int(os.getenv("PORT", "8001")))


   def _check_ns(namespace: str) -> None:
       if namespace not in ALLOWED_NAMESPACES:
           raise ValueError(f"namespace '{namespace}' is not in the allow-list {sorted(ALLOWED_NAMESPACES)}")


   def _api_error(e: ApiException) -> ValueError:
       return ValueError(f"Kubernetes API {e.status}: {e.reason}")


   class PodSummary(BaseModel):
       name: str
       phase: str
       ready: str = Field(description="ready/total containers, e.g. 1/2")
       restarts: int
       node: str | None
       age_minutes: int
       waiting_reason: str | None = Field(default=None, description="e.g. CrashLoopBackOff, ImagePullBackOff")


   class PodDetail(BaseModel):
       name: str
       namespace: str
       labels: dict[str, str]
       node: str | None
       phase: str
       conditions: list[dict]
       containers: list[dict]
       recent_events: list[str]


   class Event(BaseModel):
       last_seen: str | None
       type: str | None
       reason: str | None
       object: str
       message: str | None


   class RestartResult(BaseModel):
       deployment: str
       namespace: str
       restarted_at: str
       approved_by: str


   @mcp.tool(annotations=READ_ONLY)
   def list_pods(namespace: str = "api", label_selector: str | None = None) -> list[PodSummary]:
       """List pods in an allowed namespace with readiness, restarts and waiting reasons. Start here for 'what is unhealthy'."""
       _check_ns(namespace)
       try:
           pods = core.list_namespaced_pod(namespace, label_selector=label_selector).items
       except ApiException as e:
           raise _api_error(e)
       now = datetime.now(timezone.utc)
       out = []
       for p in pods:
           statuses = p.status.container_statuses or []
           ready = sum(1 for s in statuses if s.ready)
           waiting = next((s.state.waiting.reason for s in statuses if s.state.waiting), None)
           out.append(PodSummary(name=p.metadata.name, phase=p.status.phase or "Unknown",
                                 ready=f"{ready}/{len(statuses)}", restarts=sum(s.restart_count for s in statuses),
                                 node=p.spec.node_name,
                                 age_minutes=int((now - p.metadata.creation_timestamp).total_seconds() // 60),
                                 waiting_reason=waiting))
       return out


   @mcp.tool(annotations=READ_ONLY)
   def describe_pod(pod: str, namespace: str = "api") -> PodDetail:
       """Detailed pod state including container statuses and the last 10 events. Equivalent to kubectl describe pod."""
       _check_ns(namespace)
       try:
           p = core.read_namespaced_pod(pod, namespace)
           evs = core.list_namespaced_event(namespace, field_selector=f"involvedObject.name={pod}").items
       except ApiException as e:
           raise _api_error(e)
       evs.sort(key=lambda e: e.last_timestamp or e.event_time or datetime.min.replace(tzinfo=timezone.utc))
       containers = []
       for s in p.status.container_statuses or []:
           state = s.state.to_dict()
           containers.append({"name": s.name, "image": s.image, "ready": s.ready, "restarts": s.restart_count,
                              "state": {k: v for k, v in state.items() if v},
                              "last_state": {k: v for k, v in s.last_state.to_dict().items() if v}})
       return PodDetail(name=p.metadata.name, namespace=namespace, labels=p.metadata.labels or {},
                        node=p.spec.node_name, phase=p.status.phase or "Unknown",
                        conditions=[{"type": c.type, "status": c.status, "reason": c.reason} for c in p.status.conditions or []],
                        containers=containers,
                        recent_events=[f"{e.type} {e.reason}: {e.message}" for e in evs[-10:]])


   @mcp.tool(annotations=READ_ONLY)
   async def get_logs(pod: str, ctx: Context, namespace: str = "api", container: str | None = None,
                      tail_lines: int = 200, previous: bool = False) -> str:
       """Tail container logs. previous=True reads the crashed container's logs (use for CrashLoopBackOff). Output capped at MAX_LOG_BYTES."""
       _check_ns(namespace)
       tail_lines = min(tail_lines, 2000)
       try:
           text = core.read_namespaced_pod_log(pod, namespace, container=container, tail_lines=tail_lines,
                                               previous=previous, timestamps=True)
       except ApiException as e:
           raise _api_error(e)
       if len(text.encode()) > MAX_LOG_BYTES:
           await ctx.warning(f"logs truncated to {MAX_LOG_BYTES} bytes")
           text = text.encode()[-MAX_LOG_BYTES:].decode(errors="ignore")
           text = "[...truncated...]\n" + text
       return text or "(no log output)"


   @mcp.tool(annotations=READ_ONLY)
   def get_events(namespace: str = "api", only_warnings: bool = True, limit: int = 50) -> list[Event]:
       """Recent namespace events (Warning type by default), newest last. Use to find OOMKilled, FailedScheduling, BackOff."""
       _check_ns(namespace)
       try:
           evs = core.list_namespaced_event(namespace).items
       except ApiException as e:
           raise _api_error(e)
       if only_warnings:
           evs = [e for e in evs if e.type == "Warning"]
       evs.sort(key=lambda e: e.last_timestamp or e.event_time or datetime.min.replace(tzinfo=timezone.utc))
       return [Event(last_seen=(e.last_timestamp or e.event_time).isoformat() if (e.last_timestamp or e.event_time) else None,
                     type=e.type, reason=e.reason, object=f"{e.involved_object.kind}/{e.involved_object.name}",
                     message=e.message) for e in evs[-limit:]]


   class Approval(BaseModel):
       change_ticket: str = Field(description="Change ticket id, e.g. CHG-10422")
       confirm: bool = Field(description="Set true to proceed with the rolling restart")


   @mcp.tool(annotations=DESTRUCTIVE)
   async def rollout_restart(deployment: str, ctx: Context, namespace: str = "api",
                             approved_by: str | None = None) -> RestartResult:
       """DESTRUCTIVE: rolling restart of a Deployment (kubectl rollout restart). Causes pod churn.
       Requires ALLOW_WRITE=true on the server and human approval. Prefer describing the proposed action first."""
       _check_ns(namespace)
       if not ALLOW_WRITE:
           raise ValueError("server is read-only (ALLOW_WRITE=false); ask an operator to run the restart")
       # Approval: use elicitation if the client supports it, else require approved_by from the host's approval gate
       caps = ctx.session.client_params.capabilities if ctx.session.client_params else None
       if caps is not None and caps.elicitation is not None:
           ans = await ctx.elicit(message=f"Approve rolling restart of {namespace}/{deployment}?", schema=Approval)
           if ans.action != "accept" or not ans.data or not ans.data.confirm:
               raise ValueError("restart declined by user")
           approved_by = f"elicitation:{ans.data.change_ticket}"
       elif not approved_by:
           raise ValueError("approval required: pass approved_by after a human approves (e.g. 'slack:<USER>:CHG-10422')")
       now = datetime.now(timezone.utc).isoformat()
       body = {"spec": {"template": {"metadata": {"annotations": {"kubectl.kubernetes.io/restartedAt": now}}}}}
       try:
           apps.patch_namespaced_deployment(deployment, namespace, body)
       except ApiException as e:
           raise _api_error(e)
       await ctx.info(f"AUDIT rollout_restart {namespace}/{deployment} approved_by={approved_by}")
       return RestartResult(deployment=deployment, namespace=namespace, restarted_at=now, approved_by=approved_by)


   @mcp.resource("k8s://{namespace}/pods/{pod}/logs", mime_type="text/plain")
   def pod_logs_resource(namespace: str, pod: str) -> str:
       """Last 100 log lines of a pod as an attachable resource."""
       _check_ns(namespace)
       return core.read_namespaced_pod_log(pod, namespace, tail_lines=100)


   if __name__ == "__main__":
       mcp.run(transport=os.getenv("MCP_TRANSPORT", "stdio"))
   ```

5. Run read-only against the scoped kubeconfig:

   ```bash
   KUBECONFIG=$PWD/reader.kubeconfig npx @modelcontextprotocol/inspector --cli uv run server.py \
     --method tools/call --tool-name list_pods --tool-arg namespace=api | jq '.structuredContent.result[] | {name, phase, ready}'
   ```

   Expected output:

   ```json
   {"name":"api-6b7c9d8f5-2xk9q","phase":"Running","ready":"1/1"}
   {"name":"api-6b7c9d8f5-m4rtp","phase":"Running","ready":"1/1"}
   ```

6. Try the destructive tool while read-only:

   ```bash
   KUBECONFIG=$PWD/reader.kubeconfig npx @modelcontextprotocol/inspector --cli uv run server.py \
     --method tools/call --tool-name rollout_restart --tool-arg deployment=api | jq '{isError, text: .content[0].text}'
   ```

   Expected output:

   ```json
   {"isError":true,"text":"Error executing tool rollout_restart: server is read-only (ALLOW_WRITE=false); ask an operator to run the restart"}
   ```

7. Try a namespace outside the allow-list (`--tool-arg namespace=kube-system`) and confirm the allow-list error. Then try with the reader kubeconfig but `ALLOW_WRITE=true` and `approved_by=lab`: you get `Kubernetes API 403: Forbidden` because RBAC, not the server, is the final guard. Create a writer kubeconfig the same way with `k8s-mcp-writer` and repeat to see a successful restart:

   ```bash
   kubectl -n api rollout history deployment/api
   ```

### Verify

Three independent layers deny the restart: tool-level `ALLOW_WRITE`, host approval (elicitation or `approved_by`), and Kubernetes RBAC. Only when all three agree does the patch happen.

### Cleanup

Keep the cluster for Lab 6. Delete `*.kubeconfig` files when done; they contain bearer tokens (short-lived, but still).

### What you learned

Defence in depth for destructive MCP tools, resource templates, output caps, and why least privilege lives outside the server.

---

## Lab 4: Runbook RAG Server over Qdrant

**Objective.** Expose hybrid runbook search as an MCP tool and each runbook as a resource.

**Prerequisites.** Docker. Lab 2.

**Time.** 75 minutes.

### Steps

1. Start Qdrant and prepare runbooks:

   ```bash
   docker run -d --name qdrant -p 6333:6333 qdrant/qdrant:v1.14.1
   mkdir -p ~/capstone/ai/mcp-servers/runbook-mcp/runbooks && cd ~/capstone/ai/mcp-servers/runbook-mcp
   cat > runbooks/RB-042-oomkilled-java.md <<'EOF'
   # RB-042 OOMKilled Java service
   Symptoms: pod restarts with reason OOMKilled, container exit code 137.
   Checks: compare container memory limit to -Xmx; look for heap dump; check recent deploy.
   Mitigation: raise limit to 1.5x observed RSS or lower -Xmx; rollout restart after config change.
   EOF
   cat > runbooks/RB-017-high-error-budget-burn.md <<'EOF'
   # RB-017 High error budget burn
   Symptoms: HighErrorBudgetBurn alert, 5xx ratio above 14.4x budget for 1h and 6h.
   Checks: list_alerts, get_slo_burn_rate, query_range on 5xx by route, recent rollouts, upstream dependencies.
   Mitigation: roll back last deploy if correlated; scale out if saturation; open incident channel.
   EOF
   uv init --python 3.12 && uv add "mcp[cli]>=1.12" qdrant-client fastembed
   ```

2. `ingest.py` (dense + sparse vectors for hybrid search):

   ```python
   import glob, uuid
   from fastembed import TextEmbedding, SparseTextEmbedding
   from qdrant_client import QdrantClient, models

   COLL = "runbooks"
   client = QdrantClient(url="http://localhost:6333")
   dense = TextEmbedding("BAAI/bge-small-en-v1.5")
   sparse = SparseTextEmbedding("Qdrant/bm25")

   if not client.collection_exists(COLL):
       client.create_collection(COLL,
           vectors_config={"dense": models.VectorParams(size=384, distance=models.Distance.COSINE)},
           sparse_vectors_config={"bm25": models.SparseVectorParams(modifier=models.Modifier.IDF)})

   points = []
   for path in glob.glob("runbooks/*.md"):
       text = open(path, encoding="utf-8").read()
       rid = path.split("/")[-1].split("-")[0] + "-" + path.split("/")[-1].split("-")[1]
       d = list(dense.embed([text]))[0]
       s = list(sparse.embed([text]))[0]
       points.append(models.PointStruct(id=str(uuid.uuid5(uuid.NAMESPACE_URL, path)),
           vector={"dense": d.tolist(), "bm25": models.SparseVector(indices=s.indices.tolist(), values=s.values.tolist())},
           payload={"runbook_id": rid, "title": text.splitlines()[0].lstrip("# "), "text": text, "path": path}))
   client.upsert(COLL, points)
   print("indexed", len(points))
   ```

   ```bash
   uv run ingest.py
   ```

   Expected output: `indexed 2`.

3. `server.py`:

   ```python
   """runbook-mcp: hybrid (dense + BM25, RRF fused) runbook search over Qdrant."""
   import logging, os, sys
   from pydantic import BaseModel, Field
   from fastembed import TextEmbedding, SparseTextEmbedding
   from qdrant_client import QdrantClient, models
   from mcp.server.fastmcp import FastMCP
   from mcp.types import ToolAnnotations

   logging.basicConfig(stream=sys.stderr, level=os.getenv("LOG_LEVEL", "INFO"))
   QDRANT_URL = os.getenv("QDRANT_URL", "http://localhost:6333")
   COLL = os.getenv("QDRANT_COLLECTION", "runbooks")

   qdrant = QdrantClient(url=QDRANT_URL, api_key=os.getenv("QDRANT_API_KEY"))
   dense = TextEmbedding("BAAI/bge-small-en-v1.5")
   sparse = SparseTextEmbedding("Qdrant/bm25")

   mcp = FastMCP("runbook-mcp", instructions="Search operational runbooks before proposing remediation.",
                 host=os.getenv("HOST", "127.0.0.1"), port=int(os.getenv("PORT", "8002")), stateless_http=True)


   class Hit(BaseModel):
       runbook_id: str
       title: str
       score: float
       excerpt: str = Field(description="first 600 chars")
       uri: str


   @mcp.tool(annotations=ToolAnnotations(readOnlyHint=True, idempotentHint=True, openWorldHint=False))
   def search_runbooks(query: str, limit: int = 3) -> list[Hit]:
       """Hybrid search (semantic + keyword) over SRE runbooks. Query with symptoms, e.g. 'OOMKilled java heap'."""
       limit = min(limit, 10)
       d = list(dense.embed([query]))[0].tolist()
       s = list(sparse.embed([query]))[0]
       res = qdrant.query_points(
           COLL,
           prefetch=[
               models.Prefetch(query=d, using="dense", limit=20),
               models.Prefetch(query=models.SparseVector(indices=s.indices.tolist(), values=s.values.tolist()),
                               using="bm25", limit=20),
           ],
           query=models.FusionQuery(fusion=models.Fusion.RRF),
           limit=limit, with_payload=True)
       return [Hit(runbook_id=p.payload["runbook_id"], title=p.payload["title"], score=round(p.score, 4),
                   excerpt=p.payload["text"][:600], uri=f"runbook://{p.payload['runbook_id']}") for p in res.points]


   @mcp.resource("runbook://{runbook_id}", mime_type="text/markdown")
   def runbook(runbook_id: str) -> str:
       """Full text of a runbook by id, e.g. runbook://RB-042."""
       res = qdrant.scroll(COLL, scroll_filter=models.Filter(must=[
           models.FieldCondition(key="runbook_id", match=models.MatchValue(value=runbook_id))]), limit=1, with_payload=True)
       points = res[0]
       if not points:
           raise ValueError(f"runbook {runbook_id} not found")
       return points[0].payload["text"]


   if __name__ == "__main__":
       mcp.run(transport=os.getenv("MCP_TRANSPORT", "stdio"))
   ```

4. Query:

   ```bash
   npx @modelcontextprotocol/inspector --cli uv run server.py --method tools/call \
     --tool-name search_runbooks --tool-arg "query=container exit code 137 java" --tool-arg limit=2 \
     | jq '.structuredContent.result[] | {runbook_id, title, score}'
   ```

   Expected output:

   ```json
   {"runbook_id":"RB-042","title":"RB-042 OOMKilled Java service","score":0.5}
   {"runbook_id":"RB-017","title":"RB-017 High error budget burn","score":0.3333}
   ```

   (RRF scores are rank-based, hence the round numbers.)

### Verify

`resources/templates/list` shows `runbook://{runbook_id}`; reading `runbook://RB-042` returns the full Markdown.

### Cleanup

Keep Qdrant running for Lab 5.

### What you learned

Wrapping a RAG retriever as a tool with `resource_link`-style URIs, and keeping the server free of LLM calls (the agent does the generation).

---

## Lab 5: Consume from LangGraph (with approval gate) and Strands

**Objective.** Wire all three servers into an incident agent twice: LangGraph with `interrupt()` before destructive tools, and Strands with `MCPClient`.

**Prerequisites.** Labs 2-4 running locally; Bedrock access in `<AWS_REGION>` (or swap the model class).

**Time.** 2 hours.

### Steps

1. Dependencies:

   ```bash
   mkdir -p ~/capstone/ai/mcp-servers/agent && cd $_
   uv init --python 3.12 && uv add langgraph langchain-mcp-adapters "langchain[aws]" strands-agents
   ```

2. `langgraph_agent.py`:

   ```python
   import asyncio, os
   from langchain_aws import ChatBedrockConverse
   from langchain_core.messages import HumanMessage, ToolMessage
   from langchain_mcp_adapters.client import MultiServerMCPClient
   from langgraph.checkpoint.memory import MemorySaver
   from langgraph.graph import StateGraph, MessagesState, START, END
   from langgraph.prebuilt import ToolNode
   from langgraph.types import interrupt, Command

   BASE = os.path.expanduser("~/capstone/ai/mcp-servers")
   SERVERS = {
       "prom": {"transport": "stdio", "command": "uv", "args": ["--directory", f"{BASE}/prom-mcp", "run", "server.py"],
                "env": {"PROMETHEUS_URL": "http://localhost:9090"}},
       "k8s": {"transport": "stdio", "command": "uv", "args": ["--directory", f"{BASE}/k8s-mcp", "run", "server.py"],
               "env": {"KUBECONFIG": f"{BASE}/k8s-mcp/writer.kubeconfig", "ALLOW_WRITE": "true"}},
       "runbook": {"transport": "stdio", "command": "uv", "args": ["--directory", f"{BASE}/runbook-mcp", "run", "server.py"]},
   }
   DESTRUCTIVE = {"rollout_restart"}      # could also be derived from tool.metadata annotations


   async def main() -> None:
       client = MultiServerMCPClient(SERVERS)
       tools = await client.get_tools()
       model = ChatBedrockConverse(model="anthropic.claude-sonnet-4-20250514-v1:0",
                                   region_name=os.getenv("AWS_REGION", "<AWS_REGION>")).bind_tools(tools)

       async def agent(state: MessagesState):
           return {"messages": [await model.ainvoke(state["messages"])]}

       def approval_gate(state: MessagesState):
           last = state["messages"][-1]
           for call in getattr(last, "tool_calls", []):
               if call["name"] in DESTRUCTIVE:
                   decision = interrupt({"question": f"Approve {call['name']} {call['args']}?"})
                   if decision != "approve":
                       # replace the tool call with a refusal so the model re-plans
                       return {"messages": [ToolMessage(tool_call_id=call["id"], content="Human declined this action.")]}
                   call["args"]["approved_by"] = "slack:<USER>:CHG-10422"
           return {}

       def route(state: MessagesState):
           last = state["messages"][-1]
           if isinstance(last, ToolMessage):
               return "agent"
           return "gate" if getattr(last, "tool_calls", None) else END

       g = StateGraph(MessagesState)
       g.add_node("agent", agent); g.add_node("gate", approval_gate); g.add_node("tools", ToolNode(tools))
       g.add_edge(START, "agent")
       g.add_conditional_edges("agent", route, {"gate": "gate", END: END})
       g.add_conditional_edges("gate", lambda s: "agent" if isinstance(s["messages"][-1], ToolMessage) else "tools",
                               {"agent": "agent", "tools": "tools"})
       g.add_edge("tools", "agent")
       app = g.compile(checkpointer=MemorySaver())

       cfg = {"configurable": {"thread_id": "inc-1"}}
       result = await app.ainvoke({"messages": [HumanMessage(
           "Alert DemoAlwaysFiring is firing for service api. Triage it and fix if a runbook says to restart.")]}, cfg)
       if "__interrupt__" in result:
           print("INTERRUPT:", result["__interrupt__"][0].value)
           result = await app.ainvoke(Command(resume="approve"), cfg)
       print(result["messages"][-1].content)

   asyncio.run(main())
   ```

   ```bash
   uv run langgraph_agent.py
   ```

   Expected output (abridged, model text varies):

   ```text
   INTERRUPT: {'question': "Approve rollout_restart {'deployment': 'api', 'namespace': 'api'}?"}
   Summary: DemoAlwaysFiring is a synthetic alert ... burn rate 0.0 (no traffic) ... pods healthy ... RB-017 applies ...
   I performed a rolling restart of api/api at 2025-06-18T11:02:44Z after approval.
   ```

3. `strands_agent.py`:

   ```python
   import os
   from mcp import StdioServerParameters, stdio_client
   from strands import Agent
   from strands.models import BedrockModel
   from strands.tools.mcp import MCPClient

   BASE = os.path.expanduser("~/capstone/ai/mcp-servers")
   def stdio(dir_: str, **env):
       return MCPClient(lambda: stdio_client(StdioServerParameters(
           command="uv", args=["--directory", f"{BASE}/{dir_}", "run", "server.py"], env={**os.environ, **env})))

   prom = stdio("prom-mcp", PROMETHEUS_URL="http://localhost:9090")
   k8s = stdio("k8s-mcp", KUBECONFIG=f"{BASE}/k8s-mcp/reader.kubeconfig")   # read-only identity
   runbook = stdio("runbook-mcp")

   with prom, k8s, runbook:
       tools = prom.list_tools_sync() + k8s.list_tools_sync() + runbook.list_tools_sync()
       agent = Agent(model=BedrockModel(model_id="anthropic.claude-sonnet-4-20250514-v1:0", region_name="<AWS_REGION>"),
                     tools=tools,
                     system_prompt="You are an SRE incident agent. Investigate with read-only tools, cite runbook ids, "
                                   "and propose (never perform) destructive actions.")
       agent("Alert DemoAlwaysFiring is firing for service api. What is going on and what should we do?")
   ```

   Strands streams the tool calls to the console as it goes; you should see `Tool #1: list_alerts`, `Tool #2: get_slo_burn_rate`, `Tool #3: search_runbooks`.

### Verify

In LangGraph the graph pauses at the gate and resumes on `Command(resume="approve")`. In Strands the agent completes without a restart because the identity is read-only and the prompt says propose only.

### Cleanup

None.

### What you learned

The same servers serve two frameworks unchanged; approval can live in the host (LangGraph interrupt), in the server (elicitation), or in RBAC, and the capstone uses all three.

---

## Lab 6: Deploy to Kubernetes over Streamable HTTP and Register in Claude Code

**Objective.** Containerise `prom-mcp` and `k8s-mcp`, deploy with least-privilege ServiceAccounts, NetworkPolicy and non-root images, and connect from Claude Code.

**Prerequisites.** Labs 2-3; kind cluster.

**Time.** 2 hours.

### Steps

1. `Dockerfile` (same for each server; shown for prom-mcp):

   ```dockerfile
   FROM python:3.12-slim AS base
   COPY --from=ghcr.io/astral-sh/uv:0.7 /uv /uvx /bin/
   WORKDIR /app
   COPY pyproject.toml uv.lock ./
   RUN uv sync --frozen --no-dev --no-install-project
   COPY server.py ./
   RUN useradd --uid 10001 --no-create-home app && chown -R app /app
   USER 10001
   ENV HOST=0.0.0.0 PORT=8000 MCP_TRANSPORT=streamable-http PATH="/app/.venv/bin:$PATH"
   EXPOSE 8000
   ENTRYPOINT ["python", "server.py"]
   ```

   ```bash
   cd ~/capstone/ai/mcp-servers/prom-mcp && docker build -t prom-mcp:0.1.0 . && kind load docker-image prom-mcp:0.1.0 --name capstone
   cd ../k8s-mcp && docker build -t k8s-mcp:0.1.0 . && kind load docker-image k8s-mcp:0.1.0 --name capstone
   ```

2. Install Prometheus in-cluster (kube-prometheus-stack via Helm, see cluster 09) or for the lab a minimal one:

   ```bash
   kubectl create namespace monitoring
   kubectl -n monitoring create configmap prom-config --from-file=prometheus.yml=../prom-mcp/prometheus.yml --from-file=rules.yml=../prom-mcp/rules.yml
   kubectl -n monitoring create deployment prometheus --image=prom/prometheus:v3.4.1 --port=9090
   kubectl -n monitoring set volume deployment/prometheus --add --name=cfg --type=configmap --configmap-name=prom-config --mount-path=/etc/prometheus
   kubectl -n monitoring expose deployment prometheus --port=9090
   ```

3. `deploy/mcp-servers.yaml`:

   ```yaml
   apiVersion: v1
   kind: Namespace
   metadata: { name: mcp }
   ---
   apiVersion: v1
   kind: ServiceAccount
   metadata: { name: prom-mcp, namespace: mcp }
   automountServiceAccountToken: false          # prom-mcp does not talk to the K8s API
   ---
   apiVersion: apps/v1
   kind: Deployment
   metadata: { name: prom-mcp, namespace: mcp, labels: { app: prom-mcp } }
   spec:
     replicas: 2
     selector: { matchLabels: { app: prom-mcp } }
     template:
       metadata: { labels: { app: prom-mcp } }
       spec:
         serviceAccountName: prom-mcp
         securityContext: { runAsNonRoot: true, runAsUser: 10001, seccompProfile: { type: RuntimeDefault } }
         containers:
           - name: server
             image: prom-mcp:0.1.0
             env:
               - { name: PROMETHEUS_URL, value: "http://prometheus.monitoring.svc:9090" }
               - { name: STATELESS, value: "true" }
             ports: [{ containerPort: 8000, name: http }]
             securityContext: { allowPrivilegeEscalation: false, readOnlyRootFilesystem: true, capabilities: { drop: ["ALL"] } }
             resources: { requests: { cpu: 50m, memory: 128Mi }, limits: { memory: 256Mi } }
             readinessProbe: { tcpSocket: { port: http }, initialDelaySeconds: 3 }
             livenessProbe: { tcpSocket: { port: http }, periodSeconds: 20 }
   ---
   apiVersion: v1
   kind: Service
   metadata: { name: prom-mcp, namespace: mcp }
   spec: { selector: { app: prom-mcp }, ports: [{ port: 8000, targetPort: http }] }
   ---
   # k8s-mcp: needs a ServiceAccount with the reader Role from Lab 3 bound in namespace api
   apiVersion: v1
   kind: ServiceAccount
   metadata: { name: k8s-mcp, namespace: mcp }
   ---
   apiVersion: rbac.authorization.k8s.io/v1
   kind: RoleBinding
   metadata: { name: k8s-mcp-reader-from-mcp-ns, namespace: api }
   roleRef: { apiGroup: rbac.authorization.k8s.io, kind: Role, name: k8s-mcp-reader }
   subjects: [{ kind: ServiceAccount, name: k8s-mcp, namespace: mcp }]
   ---
   apiVersion: apps/v1
   kind: Deployment
   metadata: { name: k8s-mcp, namespace: mcp, labels: { app: k8s-mcp } }
   spec:
     replicas: 1                               # stateful sessions; scale with sticky sessions or stateless mode
     selector: { matchLabels: { app: k8s-mcp } }
     template:
       metadata: { labels: { app: k8s-mcp } }
       spec:
         serviceAccountName: k8s-mcp
         securityContext: { runAsNonRoot: true, runAsUser: 10001, seccompProfile: { type: RuntimeDefault } }
         containers:
           - name: server
             image: k8s-mcp:0.1.0
             env:
               - { name: PORT, value: "8000" }
               - { name: ALLOWED_NAMESPACES, value: "api" }
               - { name: ALLOW_WRITE, value: "false" }
             ports: [{ containerPort: 8000, name: http }]
             securityContext: { allowPrivilegeEscalation: false, readOnlyRootFilesystem: true, capabilities: { drop: ["ALL"] } }
             resources: { requests: { cpu: 50m, memory: 128Mi }, limits: { memory: 256Mi } }
   ---
   apiVersion: v1
   kind: Service
   metadata: { name: k8s-mcp, namespace: mcp }
   spec: { selector: { app: k8s-mcp }, ports: [{ port: 8000, targetPort: http }] }
   ---
   # Only the agent namespace may call MCP servers; servers may only reach Prometheus, kube-apiserver and DNS
   apiVersion: networking.k8s.io/v1
   kind: NetworkPolicy
   metadata: { name: mcp-servers, namespace: mcp }
   spec:
     podSelector: {}
     policyTypes: [Ingress, Egress]
     ingress:
       - from:
           - namespaceSelector: { matchLabels: { kubernetes.io/metadata.name: agent } }
         ports: [{ port: 8000, protocol: TCP }]
     egress:
       - to: [{ namespaceSelector: { matchLabels: { kubernetes.io/metadata.name: monitoring } } }]
         ports: [{ port: 9090, protocol: TCP }]
       - to: [{ ipBlock: { cidr: 10.96.0.1/32 } }]       # kube-apiserver ClusterIP; adjust for your cluster
         ports: [{ port: 443, protocol: TCP }]
       - to: [{ namespaceSelector: { matchLabels: { kubernetes.io/metadata.name: kube-system } } }]
         ports: [{ port: 53, protocol: UDP }, { port: 53, protocol: TCP }]
   ```

   ```bash
   kubectl create namespace agent
   kubectl apply -f deploy/mcp-servers.yaml
   kubectl -n mcp get pods
   ```

   Expected output:

   ```text
   NAME                        READY   STATUS    RESTARTS   AGE
   k8s-mcp-5c8d7b9f4-tq2zl     1/1     Running   0          40s
   prom-mcp-7f6b8c9d5-8hk2m    1/1     Running   0          40s
   prom-mcp-7f6b8c9d5-xw3np    1/1     Running   0          40s
   ```

   (kind's default CNI does not enforce NetworkPolicy; install Calico or Cilium to see it take effect, as covered in the Kubernetes folder.)

4. Port-forward and register in Claude Code:

   ```bash
   kubectl -n mcp port-forward svc/prom-mcp 18000:8000 &
   kubectl -n mcp port-forward svc/k8s-mcp 18001:8000 &
   claude mcp add --transport http --scope project prom http://127.0.0.1:18000/mcp
   claude mcp add --transport http --scope project k8s http://127.0.0.1:18001/mcp
   claude mcp list
   ```

   Expected output:

   ```text
   prom: http://127.0.0.1:18000/mcp (HTTP) - ✓ Connected
   k8s: http://127.0.0.1:18001/mcp (HTTP) - ✓ Connected
   ```

5. In Claude Code, type `/mcp` to view tools, then ask: *"Using the prom and k8s tools, which pods in namespace api are unhealthy and is any alert firing?"* Claude Code will prompt for permission on first use of each tool; read-only tools can be allow-listed in `.claude/settings.json` under `permissions.allow` as `mcp__prom__*`.

### Verify

`kubectl -n mcp logs deploy/k8s-mcp` shows `in-cluster config`; a `tools/call list_pods` from Claude Code returns the two api pods; `rollout_restart` fails with `ALLOW_WRITE=false`.

### Cleanup

```bash
kill %1 %2
kubectl delete -f deploy/mcp-servers.yaml
kind delete cluster --name capstone
docker rm -f prom qdrant
```

### What you learned

Running MCP servers as ordinary hardened Kubernetes workloads, in-cluster identity via ServiceAccount instead of kubeconfig files, and registering remote servers in a host.

---

## Lab 7: Contract Tests with pytest

**Objective.** Test servers without subprocesses or networks using an in-memory client session, and add a schema-snapshot contract test that fails CI when a tool's interface changes unintentionally.

**Prerequisites.** Lab 2.

**Time.** 45 minutes.

### Steps

1. In `prom-mcp`: `uv add --dev pytest pytest-asyncio respx` and set `asyncio_mode = "auto"` under `[tool.pytest.ini_options]` in `pyproject.toml`.

2. `tests/test_contract.py`:

   ```python
   import json, pathlib
   import httpx, pytest, respx
   from mcp.shared.memory import create_connected_server_and_client_session
   import server

   SNAPSHOT = pathlib.Path(__file__).parent / "tools_snapshot.json"


   @pytest.fixture
   async def session():
       async with create_connected_server_and_client_session(server.mcp._mcp_server) as s:
           await s.initialize()
           yield s


   async def test_handshake_and_tool_list(session):
       tools = {t.name: t for t in (await session.list_tools()).tools}
       assert set(tools) == {"query_instant", "query_range", "list_alerts", "get_slo_burn_rate"}
       for t in tools.values():
           assert t.annotations and t.annotations.readOnlyHint is True, f"{t.name} must be read-only"


   async def test_tool_schema_snapshot(session):
       tools = sorted((await session.list_tools()).tools, key=lambda t: t.name)
       current = [{"name": t.name, "inputSchema": t.inputSchema, "outputSchema": t.outputSchema} for t in tools]
       if not SNAPSHOT.exists():
           SNAPSHOT.write_text(json.dumps(current, indent=2, sort_keys=True))
           pytest.skip("snapshot created; commit it")
       assert current == json.loads(SNAPSHOT.read_text()), "tool contract changed - update snapshot deliberately"


   @respx.mock
   async def test_list_alerts_filters_state(session):
       respx.get("http://localhost:9090/api/v1/alerts").mock(return_value=httpx.Response(200, json={
           "status": "success", "data": {"alerts": [
               {"labels": {"alertname": "A", "severity": "critical"}, "state": "firing", "annotations": {}, "activeAt": "t"},
               {"labels": {"alertname": "B"}, "state": "pending", "annotations": {}, "activeAt": "t"}]}}))
       res = await session.call_tool("list_alerts", {"state": "firing"})
       assert res.isError is False
       assert [a["name"] for a in res.structuredContent["result"]] == ["A"]


   @respx.mock
   async def test_prometheus_error_is_tool_error_not_protocol_error(session):
       respx.get("http://localhost:9090/api/v1/query").mock(return_value=httpx.Response(400, json={
           "status": "error", "errorType": "bad_data", "error": "parse error at char 3"}))
       res = await session.call_tool("query_instant", {"query": "up{"})
       assert res.isError is True
       assert "parse error" in res.content[0].text


   async def test_invalid_arguments_rejected(session):
       res = await session.call_tool("query_range", {"query": "up", "minutes": "sixty"})
       assert res.isError is True            # pydantic validation -> tool error with message
   ```

   ```bash
   uv run pytest -q
   ```

   Expected output:

   ```text
   ....s                                                                   [100%]
   4 passed, 1 skipped in 1.9s
   ```

   Run again and the snapshot test passes (5 passed). Commit `tests/tools_snapshot.json`.

3. Add to the capstone GitHub Actions workflow (job excerpt):

   ```yaml
   mcp-contract-tests:
     runs-on: ubuntu-latest
     strategy: { matrix: { server: [prom-mcp, k8s-mcp, runbook-mcp] } }
     steps:
       - uses: actions/checkout@v4
       - uses: astral-sh/setup-uv@v5
       - run: uv sync --frozen
         working-directory: ai/mcp-servers/${{ matrix.server }}
       - run: uv run pytest -q
         working-directory: ai/mcp-servers/${{ matrix.server }}
   ```

### Verify

Change `minutes: int = 60` to `window_minutes` in `query_range`; the snapshot test fails with a clear diff. Revert.

### Cleanup

None.

### What you learned

In-memory sessions make MCP servers unit-testable in milliseconds; schema snapshots turn the tool contract into a reviewed artefact, which matters because agents and evals depend on stable tool names and arguments.

---

## Where Next

- Harden and productionise everything above in [05-production-patterns.md](05-production-patterns.md) (OAuth 2.1, threat model, OpenTelemetry, registry).
- Evaluate whether the agent picks the right tools with DeepEval's `ToolCorrectnessMetric` in [../llm-evals/deepeval/](../llm-evals/deepeval/README.md).
