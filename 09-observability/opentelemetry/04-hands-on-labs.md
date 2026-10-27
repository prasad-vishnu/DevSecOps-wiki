# OpenTelemetry: Hands-On Labs

| | |
|---|---|
| **Cluster** | 09 Observability |
| **Role** | Anchor |
| **Prerequisites** | [01-installation-and-setup.md](01-installation-and-setup.md), [02-core-concepts.md](02-core-concepts.md); Docker Compose v2; for Labs 6-8 a kind or EKS cluster (Kubernetes 1.31+) with Helm 3 |
| **Target version** | Collector 0.115 (pinned in manifests), Python SDK 1.28 / distro 0.49b0, Java agent 2.10 |
| **Capstone phase** | Phase 4 (Months 13-16) |

> Eight progressive labs. Labs 1-5 run on the docker-compose lab stack from the [cluster README](../README.md); Labs 6-8 move to Kubernetes with the OTel Operator. By the end you have both capstone services instrumented, a gateway Collector enforcing redaction and tail sampling, span metrics flowing to Prometheus, logs correlated by `trace_id` in Loki, and an LLM call traced with `gen_ai.*` attributes.

Lab layout used throughout:

```text
observability-lab/
├── docker-compose.yml            # from ../README.md section 7
├── otel-collector/config.yaml
├── prometheus/prometheus.yml
├── loki/config.yaml
├── tempo/tempo.yaml
├── grafana/provisioning/datasources/datasources.yaml
└── services/
    ├── orders-api/               # FastAPI
    └── payments-api/             # Spring Boot
```

---

## Lab 1: Collector with the Debug Exporter

**Objective.** Run the Collector, send one span and one metric with `telemetrygen`, and read them in the debug output. Learn to read the Collector's self-metrics.

**Prerequisites.** Docker; `telemetrygen` installed (`go install github.com/open-telemetry/opentelemetry-collector-contrib/cmd/telemetrygen@latest`) or `curl`.

**Time.** 30 minutes.

### Steps

1. Create `otel-collector/config.yaml` with the minimal config from [01-installation-and-setup.md](01-installation-and-setup.md) section 2 (debug exporter, health_check, zpages, self-telemetry on 8888).

2. Start only the Collector:

   ```bash
   docker compose up -d otel-collector
   docker compose logs -f otel-collector
   ```

   Expected: `Everything is ready. Begin running and processing data.`

3. In a second terminal send 10 traces with 2 child spans each:

   ```bash
   telemetrygen traces --otlp-endpoint localhost:4317 --otlp-insecure --traces 10 --child-spans 2 \
     --otlp-attributes 'service.name="telemetrygen"' --span-duration 120ms
   ```

   Expected (Collector log, abbreviated):

   ```text
   info    TracesExporter  {"kind": "exporter", "data_type": "traces", "name": "debug", "resource spans": 1, "spans": 30}
   info    ResourceSpans #0
   Resource attributes:
        -> service.name: Str(telemetrygen)
   ...
   Span #0
       Trace ID       : 3a9c...
       Parent ID      : 7f12...
       ID             : 9b0e...
       Name           : okey-dokey-0
       Kind           : Server
   ```

4. Send metrics and logs:

   ```bash
   telemetrygen metrics --otlp-endpoint localhost:4317 --otlp-insecure --metrics 5
   telemetrygen logs    --otlp-endpoint localhost:4317 --otlp-insecure --logs 5 --body "hello from lab 1"
   ```

5. Read the self-metrics:

   ```bash
   curl -s localhost:8888/metrics | grep -E '^otelcol_(receiver_accepted|exporter_sent)_(spans|metric_points|log_records)'
   ```

   Expected:

   ```text
   otelcol_exporter_sent_log_records{exporter="debug",service_instance_id="...",service_name="otelcol-contrib",service_version="0.115.0"} 5
   otelcol_exporter_sent_metric_points{exporter="debug",...} 5
   otelcol_exporter_sent_spans{exporter="debug",...} 30
   otelcol_receiver_accepted_log_records{receiver="otlp",transport="grpc",...} 5
   otelcol_receiver_accepted_metric_points{receiver="otlp",transport="grpc",...} 5
   otelcol_receiver_accepted_spans{receiver="otlp",transport="grpc",...} 30
   ```

6. Open zPages at http://localhost:55679/debug/pipelinez and confirm three pipelines, each `otlp -> memory_limiter -> batch -> debug`.

### Verify

`otelcol_receiver_accepted_spans` equals `otelcol_exporter_sent_spans` (30). `otelcol_exporter_send_failed_spans` is absent or 0.

### Cleanup

Keep the Collector running for Lab 2.

### What you learned

The Collector is a pipeline; the debug exporter and `:8888` metrics are your first two debugging tools; `telemetrygen` generates realistic synthetic data without an application.

---

## Lab 2: Zero-Code FastAPI Instrumentation

**Objective.** Run `orders-api` under `opentelemetry-instrument`, see auto-generated spans for FastAPI, httpx and SQLAlchemy, and confirm log records carry `trace_id`.

**Prerequisites.** Lab 1; Python 3.12.

**Time.** 45 minutes.

### Steps

1. Create the service.

   `services/orders-api/requirements.txt`:

   ```text
   fastapi==0.115.5
   uvicorn[standard]==0.32.1
   httpx==0.27.2
   sqlalchemy[asyncio]==2.0.36
   aiosqlite==0.20.0
   opentelemetry-distro==0.49b0
   opentelemetry-exporter-otlp==1.28.0
   opentelemetry-instrumentation-fastapi==0.49b0
   opentelemetry-instrumentation-httpx==0.49b0
   opentelemetry-instrumentation-sqlalchemy==0.49b0
   opentelemetry-instrumentation-logging==0.49b0
   ```

   `services/orders-api/app/main.py`:

   ```python
   import logging, os, random
   from fastapi import FastAPI, HTTPException
   import httpx
   from sqlalchemy.ext.asyncio import create_async_engine
   from sqlalchemy import text

   logging.basicConfig(level=logging.INFO, format="%(asctime)s %(levelname)s %(name)s %(message)s")
   log = logging.getLogger("orders")
   PAYMENTS_URL = os.getenv("PAYMENTS_URL", "http://payments-api:8080")
   engine = create_async_engine("sqlite+aiosqlite:///./orders.db")
   app = FastAPI(title="orders-api")

   @app.on_event("startup")
   async def init_db():
       async with engine.begin() as conn:
           await conn.execute(text("CREATE TABLE IF NOT EXISTS orders (id TEXT PRIMARY KEY, amount_cents INTEGER)"))
           await conn.execute(text("INSERT OR IGNORE INTO orders VALUES ('42', 1999), ('43', 4500)"))

   @app.get("/orders/{order_id}")
   async def get_order(order_id: str):
       async with engine.connect() as conn:
           row = (await conn.execute(text("SELECT amount_cents FROM orders WHERE id = :id"), {"id": order_id})).first()
       if row is None:
           log.warning("order not found", extra={"order_id": order_id})
           raise HTTPException(status_code=404, detail="order not found")
       if random.random() < 0.05:
           log.error("simulated upstream failure", extra={"order_id": order_id})
           raise HTTPException(status_code=500, detail="simulated failure")
       async with httpx.AsyncClient(timeout=2.0) as client:
           r = await client.get(f"{PAYMENTS_URL}/payments/{order_id}")
       log.info("order served", extra={"order_id": order_id})
       return {"order_id": order_id, "amount_cents": row[0], "payment": r.json() if r.status_code == 200 else None}

   @app.get("/healthz")
   async def healthz():
       return {"status": "ok"}
   ```

   `services/orders-api/Dockerfile`:

   ```dockerfile
   FROM python:3.12-slim
   RUN useradd --create-home --uid 10001 app
   WORKDIR /app
   COPY requirements.txt .
   RUN pip install --no-cache-dir -r requirements.txt
   COPY app ./app
   USER app
   EXPOSE 8000
   CMD ["opentelemetry-instrument", "uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
   ```

2. Build and start (the compose file sets the `OTEL_*` variables):

   ```bash
   docker compose up -d --build orders-api
   docker compose logs orders-api | head -n 5
   ```

3. Generate traffic:

   ```bash
   for i in 42 43 42 99 42; do curl -s -o /dev/null -w "%{http_code}\n" localhost:8000/orders/$i; done
   ```

   Expected: `200 200 200 404 200` (occasionally a `500`).

4. Read the Collector output. You should see, per request, a `SERVER` span `GET /orders/{order_id}` with:

   ```text
   -> http.request.method: Str(GET)
   -> http.route: Str(/orders/{order_id})
   -> url.path: Str(/orders/42)
   -> http.response.status_code: Int(200)
   ```

   a child `CLIENT` span `SELECT orders` with `db.system: sqlite` and `db.statement: SELECT amount_cents FROM orders WHERE id = ?`, and a child `CLIENT` span `GET` to `payments-api` (which will be `ERROR` with `net.peer.name`/`server.address` until Lab 3 starts that service).

5. Find the log record for `order served` in the debug output and confirm:

   ```text
   LogRecord #2
   Timestamp: ...
   SeverityText: INFO
   Body: Str(order served)
   Trace ID: 8d2f...   <- same as the SERVER span's Trace ID
   Span ID: 1a7c...
   ```

6. Exclude health checks so they stop producing spans. Add to the `orders-api` service environment in `docker-compose.yml`:

   ```yaml
   OTEL_PYTHON_EXCLUDED_URLS: "healthz"
   ```

   Recreate and confirm `curl localhost:8000/healthz` generates no span.

### Verify

`curl -s localhost:8888/metrics | grep otelcol_receiver_accepted_spans` increases by at least 3 per `/orders/42` request (server span + db span + client span).

### Cleanup

Leave running.

### What you learned

`opentelemetry-instrument` wires the SDK and all installed instrumentations from env vars only; `http.route` is the low-cardinality name while `url.path` is the concrete value; the logging instrumentation stamps `trace_id` so Loki can later join logs to traces.

---

## Lab 3: Java Agent on Spring Boot and Cross-Service Propagation

**Objective.** Attach the javaagent to `payments-api`, make `orders-api` call it, and observe one trace spanning both services.

**Prerequisites.** Lab 2; Java 21, Maven.

**Time.** 45 minutes.

### Steps

1. Scaffold `services/payments-api` with Spring Initializr (web, actuator, micrometer-registry-prometheus) and add the controller:

   ```java
   // src/main/java/com/example/payments/PaymentController.java
   package com.example.payments;

   import org.slf4j.Logger;
   import org.slf4j.LoggerFactory;
   import org.springframework.web.bind.annotation.*;
   import java.util.Map;
   import java.util.concurrent.ThreadLocalRandom;

   @RestController
   public class PaymentController {
       private static final Logger log = LoggerFactory.getLogger(PaymentController.class);

       @GetMapping("/payments/{id}")
       public Map<String, Object> get(@PathVariable String id) throws InterruptedException {
           long delay = ThreadLocalRandom.current().nextLong(20, 180);
           Thread.sleep(delay);
           log.info("payment looked up id={} delayMs={}", id, delay);
           return Map.of("payment_id", id, "status", "SETTLED", "latency_ms", delay);
       }
   }
   ```

   `src/main/resources/application.properties`:

   ```properties
   server.port=8080
   management.endpoints.web.exposure.include=health,prometheus
   management.metrics.distribution.percentiles-histogram.http.server.requests=true
   logging.pattern.level=%5p [trace_id=%X{trace_id} span_id=%X{span_id}]
   ```

   The `logging.pattern.level` line prints the MDC values the agent injects, so you can see correlation in plain-text logs even before switching to JSON.

2. Add the agent to the Dockerfile (from [01-installation-and-setup.md](01-installation-and-setup.md) section 5) and build:

   ```bash
   (cd services/payments-api && ./mvnw -q -DskipTests package)
   docker compose up -d --build payments-api
   docker compose logs payments-api | grep -m1 otel.javaagent
   ```

   Expected: `... opentelemetry-javaagent - version: 2.10.0`.

3. Call the chain:

   ```bash
   curl -s localhost:8000/orders/42 | jq .
   ```

   Expected:

   ```json
   { "order_id": "42", "amount_cents": 1999, "payment": { "payment_id": "42", "status": "SETTLED", "latency_ms": 97 } }
   ```

4. In the Collector output find two `ResourceSpans` blocks sharing one `Trace ID`: `orders-api` (SERVER `GET /orders/{order_id}`, CLIENT `GET`) and `payments-api` (SERVER `GET /payments/{id}` whose `Parent ID` equals the orders-api CLIENT span's `ID`).

5. Look at the `payments-api` log line:

   ```bash
   docker compose logs payments-api | grep "payment looked up"
   ```

   Expected:

   ```text
   INFO [trace_id=8d2f1e... span_id=5c3a...] 1 --- [nio-8080-exec-1] c.e.payments.PaymentController : payment looked up id=42 delayMs=97
   ```

6. Break propagation deliberately to see what it looks like: set `OTEL_PROPAGATORS=b3` on `payments-api` only, recreate, call again. Now `payments-api` starts a *new* trace (no `Parent ID`), because it cannot read `traceparent`. Restore `tracecontext,baggage`.

### Verify

A single `trace_id` appears in the orders-api server span, the orders-api client span, the payments-api server span, and both services' log lines.

### Cleanup

Leave running.

### What you learned

The javaagent needs no code; `JAVA_TOOL_OPTIONS` attaches it; W3C `traceparent` is what ties services together, and mismatched propagators are the first thing to check when a trace splits.

---

## Lab 4: A Real Pipeline to Tempo, Prometheus and Loki

**Objective.** Replace the debug exporter with real backends, add `spanmetrics`, and see metrics, logs and traces in Grafana from one request.

**Prerequisites.** Lab 3; the rest of the compose stack (Prometheus, Loki, Tempo, Grafana) configured per their folders' `01-installation-and-setup.md`.

**Time.** 60 minutes.

### Steps

1. Replace `otel-collector/config.yaml`:

   ```yaml
   receivers:
     otlp:
       protocols:
         grpc: { endpoint: 0.0.0.0:4317 }
         http: { endpoint: 0.0.0.0:4318 }

   processors:
     memory_limiter: { check_interval: 1s, limit_percentage: 75, spike_limit_percentage: 15 }
     resource/env:
       attributes:
         - { key: deployment.environment, value: lab, action: upsert }
     filter/health:
       error_mode: ignore
       traces:
         span: ['attributes["http.route"] == "/healthz"', 'attributes["url.path"] == "/actuator/health"']
     transform/logs:
       error_mode: ignore
       log_statements:
         - context: log
           statements:
             - set(attributes["level"], severity_text) where severity_text != nil
     batch: { timeout: 2s }

   connectors:
     spanmetrics:
       namespace: traces.spanmetrics
       histogram:
         explicit: { buckets: [5ms, 10ms, 25ms, 50ms, 100ms, 250ms, 300ms, 500ms, 1s, 2500ms, 5s] }
       dimensions:
         - { name: http.request.method }
         - { name: http.route }
         - { name: http.response.status_code }
       exemplars: { enabled: true }
     servicegraph:
       latency_histogram_buckets: [10ms, 50ms, 100ms, 250ms, 500ms, 1s]

   exporters:
     otlp/tempo:
       endpoint: tempo:4317
       tls: { insecure: true }          # lab only
     prometheusremotewrite:
       endpoint: http://prometheus:9090/api/v1/write
       resource_to_telemetry_conversion: { enabled: true }
       target_info: { enabled: true }
     prometheus:
       endpoint: 0.0.0.0:8889
       enable_open_metrics: true
     otlphttp/loki:
       endpoint: http://loki:3100/otlp
       tls: { insecure: true }
     debug:
       verbosity: basic

   extensions:
     health_check: { endpoint: 0.0.0.0:13133 }

   service:
     extensions: [health_check]
     telemetry:
       metrics:
         readers: [{ pull: { exporter: { prometheus: { host: 0.0.0.0, port: 8888 } } } }]
     pipelines:
       traces:
         receivers: [otlp]
         processors: [memory_limiter, resource/env, filter/health, batch]
         exporters: [otlp/tempo, spanmetrics, servicegraph]
       metrics:
         receivers: [otlp]
         processors: [memory_limiter, resource/env, batch]
         exporters: [prometheusremotewrite]
       metrics/derived:
         receivers: [spanmetrics, servicegraph]
         processors: [batch]
         exporters: [prometheus, prometheusremotewrite]
       logs:
         receivers: [otlp]
         processors: [memory_limiter, resource/env, transform/logs, batch]
         exporters: [otlphttp/loki]
   ```

2. Ensure Prometheus scrapes the Collector's exporter and self-metrics. In `prometheus/prometheus.yml`:

   ```yaml
   global: { scrape_interval: 15s }
   scrape_configs:
     - job_name: otel-collector
       static_configs: [{ targets: ["otel-collector:8888", "otel-collector:8889"] }]
     - job_name: payments-api
       metrics_path: /actuator/prometheus
       static_configs: [{ targets: ["payments-api:8080"] }]
   ```

3. Restart and generate load:

   ```bash
   docker compose up -d --force-recreate otel-collector prometheus
   for i in $(seq 1 300); do curl -s -o /dev/null localhost:8000/orders/$(( (i % 3) + 42 )); sleep 0.1; done
   ```

4. Query Prometheus for the derived RED metrics:

   ```promql
   sum by (service_name, http_route) (rate(traces_spanmetrics_calls_total[1m]))
   ```

   ```promql
   histogram_quantile(0.95, sum by (le, service_name) (rate(traces_spanmetrics_duration_milliseconds_bucket[5m])))
   ```

   Expected: series for `orders-api` / `/orders/{order_id}` and `payments-api` / `/payments/{id}`; p95 around 150-200 ms.

5. In Grafana Explore (Tempo data source) run:

   ```traceql
   { resource.service.name = "orders-api" && span.http.response.status_code = 500 }
   ```

   Open a trace. Click the `payments-api` span, then "Logs for this span". Loki opens with `{service_name="payments-api"} | trace_id="..."` and shows the `payment looked up` line.

6. In Grafana Explore (Loki):

   ```logql
   {service_name="orders-api"} | json | level="ERROR"
   ```

   Click a `trace_id` value; the derived field opens the trace in Tempo.

7. Check exemplars: in a Prometheus panel for the p95 query above, enable "Exemplars" in the query options. Dots appear on the graph; hovering shows `trace_id` and a "Query with Tempo" button.

### Verify

- `curl -s localhost:8889/metrics | grep -c traces_spanmetrics_calls_total` > 0.
- `curl -s 'localhost:3100/loki/api/v1/labels' | jq .` includes `service_name`.
- `curl -s 'localhost:3200/api/search?tags=service.name%3Dorders-api&limit=1' | jq '.traces | length'` returns 1.

### Cleanup

Leave running.

### What you learned

The three backends each receive the signal they are built for; the `spanmetrics` connector gives RED metrics with no application metric code; the correlation loop (metric -> exemplar -> trace -> logs) works because `trace_id` and `service.name` are consistent everywhere.

---

## Lab 5: PII Redaction, Tail Sampling and Cost Control

**Objective.** Add `redaction`, `attributes` hashing and `tail_sampling` to the gateway; prove that card-like numbers never reach Tempo and that 100% of errors but only 10% of successes are stored.

**Prerequisites.** Lab 4.

**Time.** 45 minutes.

### Steps

1. Make `orders-api` emit a sensitive-looking attribute (intentionally, to test the filter). In `get_order` add:

   ```python
   from opentelemetry import trace
   trace.get_current_span().set_attribute("customer.note", "card 4111111111111111 on file")  # test fixture only
   trace.get_current_span().set_attribute("user.id", f"user-{order_id}")
   ```

2. Add processors to the Collector:

   ```yaml
   processors:
     redaction:
       allow_all_keys: true
       blocked_values:
         - "4[0-9]{12}(?:[0-9]{3})?"
         - "[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\\.[A-Za-z]{2,}"
       summary: info
     attributes/hash:
       actions:
         - { key: user.id, action: hash }
     tail_sampling:
       decision_wait: 5s
       num_traces: 50000
       policies:
         - { name: errors, type: status_code, status_code: { status_codes: [ERROR] } }
         - { name: slow, type: latency, latency: { threshold_ms: 400 } }
         - { name: baseline, type: probabilistic, probabilistic: { sampling_percentage: 10 } }
   ```

   and reorder the traces pipeline so span metrics are computed **before** sampling:

   ```yaml
   pipelines:
     traces:
       receivers: [otlp]
       processors: [memory_limiter, resource/env, filter/health, redaction, attributes/hash, batch]
       exporters: [spanmetrics, servicegraph, otlp/sampled]       # fan out to a second pipeline via a forward connector
     traces/sampled:
       receivers: [forward]
       processors: [tail_sampling, batch]
       exporters: [otlp/tempo]
   connectors:
     forward: {}
   ```

   Replace `otlp/sampled` with `forward` in the first pipeline's exporters (the `forward` connector is both exporter and receiver).

3. Restart the Collector and generate 500 requests. Then query Tempo for a successful trace and inspect `customer.note`:

   Expected attribute value: `card **** on file` (the redaction processor replaces matches with `****`) and `user.id` is a 64-character hex string.

4. Compare counts:

   ```promql
   sum(rate(traces_spanmetrics_calls_total{service_name="orders-api"}[5m]))                 # all requests (pre-sampling)
   ```

   ```promql
   sum(rate(otelcol_processor_tail_sampling_count_traces_sampled{sampled="true"}[5m]))     # stored
   sum by (policy) (rate(otelcol_processor_tail_sampling_count_traces_sampled{sampled="true"}[5m]))
   ```

   Expected: stored traces are roughly 10-15% of all requests; the `errors` policy accounts for every 500.

5. Confirm memory is bounded:

   ```bash
   curl -s localhost:8888/metrics | grep -E 'otelcol_process_memory_rss|otelcol_processor_tail_sampling_sampling_trace_dropped_too_early'
   ```

   `trace_dropped_too_early` should be 0; if not, raise `num_traces`.

### Verify

A TraceQL search `{ span.customer.note =~ ".*4111.*" }` returns nothing. `{ status = error }` returns every failed request for the last 5 minutes.

### Cleanup

Remove the test-fixture attributes from the application.

### What you learned

Redaction and hashing belong in the gateway so no application can leak PII to storage; tail sampling keeps all errors and slow traces while cutting storage 85-90%; computing span metrics before sampling keeps RED metrics exact.

---

## Lab 6: Kubernetes with the OTel Operator and Auto-Injection

**Objective.** Deploy the Operator, a gateway Collector and an `Instrumentation` CR; instrument both services with annotations only.

**Prerequisites.** kind or EKS cluster (Kubernetes 1.31+), Helm 3, `kubectl`; the LGTM stack installed in `observability` per the sibling folders (or Lab 4's compose backends reachable via port-forward is acceptable for kind).

**Time.** 75 minutes.

### Steps

1. Install cert-manager and the Operator (commands in [01-installation-and-setup.md](01-installation-and-setup.md) section 6).

2. Apply RBAC (ServiceAccount `otel-collector` in `observability`, ClusterRole from the cheatsheet, ClusterRoleBinding).

3. Apply the gateway Collector CR, using the Lab 4/5 config with in-cluster endpoints and `k8sattributes`:

   ```yaml
   apiVersion: opentelemetry.io/v1beta1
   kind: OpenTelemetryCollector
   metadata: { name: otel-collector, namespace: observability }
   spec:
     mode: deployment
     replicas: 2
     serviceAccount: otel-collector
     image: otel/opentelemetry-collector-contrib:0.115.0
     resources:
       requests: { cpu: 250m, memory: 512Mi }
       limits:   { memory: 1Gi }
     securityContext:
       runAsNonRoot: true
       runAsUser: 10001
       readOnlyRootFilesystem: true
       allowPrivilegeEscalation: false
       capabilities: { drop: [ALL] }
     observability:
       metrics: { enableMetrics: true }        # creates a ServiceMonitor for :8888
     config:
       receivers:
         otlp:
           protocols:
             grpc: { endpoint: 0.0.0.0:4317 }
             http: { endpoint: 0.0.0.0:4318 }
       processors:
         memory_limiter: { check_interval: 1s, limit_percentage: 75, spike_limit_percentage: 15 }
         k8sattributes:
           auth_type: serviceAccount
           extract:
             metadata: [k8s.namespace.name, k8s.pod.name, k8s.deployment.name, k8s.node.name]
             labels: [{ tag_name: app, key: app, from: pod }]
           pod_association:
             - sources: [{ from: resource_attribute, name: k8s.pod.ip }]
             - sources: [{ from: connection }]
         redaction:
           allow_all_keys: true
           blocked_values: ["4[0-9]{12}(?:[0-9]{3})?", "[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\\.[A-Za-z]{2,}"]
         filter/health:
           traces: { span: ['attributes["http.route"] == "/healthz"', 'attributes["url.path"] == "/actuator/health"'] }
         batch: {}
       connectors:
         spanmetrics:
           histogram: { explicit: { buckets: [5ms, 25ms, 50ms, 100ms, 250ms, 300ms, 500ms, 1s, 2500ms, 5s] } }
           dimensions: [{ name: http.request.method }, { name: http.route }, { name: http.response.status_code }]
           exemplars: { enabled: true }
         servicegraph: {}
       exporters:
         otlp/tempo:
           endpoint: tempo.observability.svc.cluster.local:4317
           tls: { insecure: true }
         prometheusremotewrite:
           endpoint: http://kps-kube-prometheus-stack-prometheus.observability.svc:9090/api/v1/write
           resource_to_telemetry_conversion: { enabled: true }
         otlphttp/loki:
           endpoint: http://loki-gateway.observability.svc/otlp
       service:
         pipelines:
           traces:  { receivers: [otlp], processors: [memory_limiter, k8sattributes, redaction, filter/health, batch], exporters: [otlp/tempo, spanmetrics, servicegraph] }
           metrics: { receivers: [otlp], processors: [memory_limiter, k8sattributes, batch], exporters: [prometheusremotewrite] }
           metrics/derived: { receivers: [spanmetrics, servicegraph], processors: [batch], exporters: [prometheusremotewrite] }
           logs:    { receivers: [otlp], processors: [memory_limiter, k8sattributes, redaction, batch], exporters: [otlphttp/loki] }
   ```

   Also apply the `otel-collector` Service alias from the install file.

4. Apply the `Instrumentation` CR named `capstone` from the install file (sampler ratio `1.0` for the lab).

5. Deploy the two services with annotations:

   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata: { name: orders-api, namespace: capstone, labels: { app: orders-api } }
   spec:
     replicas: 2
     selector: { matchLabels: { app: orders-api } }
     template:
       metadata:
         labels: { app: orders-api }
         annotations:
           instrumentation.opentelemetry.io/inject-python: "observability/capstone"
       spec:
         securityContext: { runAsNonRoot: true, runAsUser: 10001, seccompProfile: { type: RuntimeDefault } }
         containers:
           - name: api
             image: ghcr.io/example/orders-api:0.1.0          # built WITHOUT opentelemetry-instrument in CMD
             ports: [{ containerPort: 8000 }]
             env:
               - { name: PAYMENTS_URL, value: http://payments-api.capstone.svc:8080 }
               - { name: OTEL_PYTHON_EXCLUDED_URLS, value: healthz }
             resources: { requests: { cpu: 100m, memory: 128Mi }, limits: { memory: 256Mi } }
             securityContext: { allowPrivilegeEscalation: false, capabilities: { drop: [ALL] } }
   ```

   Repeat for `payments-api` with `inject-java` and port 8080. Apply Services for both.

6. Verify injection:

   ```bash
   kubectl --namespace capstone get pod -l app=orders-api -o jsonpath='{.items[0].spec.initContainers[0].name}{"\n"}'
   kubectl --namespace capstone exec deploy/orders-api -- env | grep -E '^(OTEL_|PYTHONPATH)'
   ```

   Expected:

   ```text
   opentelemetry-auto-instrumentation-python
   OTEL_SERVICE_NAME=orders-api
   OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector.observability.svc.cluster.local:4318
   OTEL_RESOURCE_ATTRIBUTES=k8s.container.name=api,k8s.deployment.name=orders-api,k8s.namespace.name=capstone,k8s.node.name=...,k8s.pod.name=...,service.namespace=capstone,...
   PYTHONPATH=/otel-auto-instrumentation-python/opentelemetry/instrumentation/auto_instrumentation:/otel-auto-instrumentation-python
   ```

7. Generate traffic and confirm in Grafana that `resource.k8s.pod.name` is present on spans and that `kubectl --namespace observability logs deploy/otel-collector-collector` shows no export errors.

### Verify

```bash
kubectl run load --rm -it --restart=Never --image=curlimages/curl:8.10.1 -- sh -c 'for i in $(seq 1 100); do curl -s -o /dev/null http://orders-api.capstone.svc:8000/orders/42; done'
```

TraceQL `{ resource.k8s.namespace.name = "capstone" }` returns traces that include both services.

### Cleanup

Keep the cluster for Labs 7-8.

### What you learned

The Operator turns instrumentation into a one-line annotation; `k8sattributes` enriches telemetry with the metadata Grafana filters by; the gateway Collector centralises policy.

---

## Lab 7: Agent DaemonSet, Target Allocator and Infrastructure Telemetry

**Objective.** Add a DaemonSet Collector that tails container logs with `filelog`, collects `kubeletstats` and `hostmetrics`, and use the target allocator to scrape Prometheus endpoints via `ServiceMonitor`s.

**Prerequisites.** Lab 6.

**Time.** 60 minutes.

### Steps

1. Apply the agent CR:

   ```yaml
   apiVersion: opentelemetry.io/v1beta1
   kind: OpenTelemetryCollector
   metadata: { name: otel-agent, namespace: observability }
   spec:
     mode: daemonset
     serviceAccount: otel-collector
     image: otel/opentelemetry-collector-contrib:0.115.0
     tolerations: [{ operator: Exists }]
     env:
       - { name: K8S_NODE_NAME, valueFrom: { fieldRef: { fieldPath: spec.nodeName } } }
     volumeMounts:
       - { name: varlogpods, mountPath: /var/log/pods, readOnly: true }
       - { name: varlibdockercontainers, mountPath: /var/lib/docker/containers, readOnly: true }
       - { name: hostfs, mountPath: /hostfs, readOnly: true, mountPropagation: HostToContainer }
     volumes:
       - { name: varlogpods, hostPath: { path: /var/log/pods } }
       - { name: varlibdockercontainers, hostPath: { path: /var/lib/docker/containers } }
       - { name: hostfs, hostPath: { path: / } }
     securityContext:
       runAsUser: 0          # required to read /var/log/pods on most distros; document this exception
       readOnlyRootFilesystem: true
     config:
       receivers:
         filelog:
           include: [/var/log/pods/*/*/*.log]
           exclude: [/var/log/pods/observability_otel-agent*/*/*.log]
           start_at: end
           include_file_path: true
           operators:
             - { type: container, id: container-parser }
             - type: json_parser
               if: 'body matches "^\\{"'
               parse_to: attributes
               severity: { parse_from: attributes.level }
               trace: { trace_id: { parse_from: attributes.trace_id }, span_id: { parse_from: attributes.span_id } }
         kubeletstats:
           collection_interval: 30s
           auth_type: serviceAccount
           endpoint: https://${env:K8S_NODE_NAME}:10250
           insecure_skip_verify: true
           metric_groups: [node, pod, container]
         hostmetrics:
           collection_interval: 30s
           root_path: /hostfs
           scrapers: { cpu: {}, memory: {}, filesystem: {}, network: {}, load: {} }
       processors:
         memory_limiter: { check_interval: 1s, limit_percentage: 75, spike_limit_percentage: 15 }
         k8sattributes:
           auth_type: serviceAccount
           filter: { node_from_env_var: K8S_NODE_NAME }
           extract:
             metadata: [k8s.namespace.name, k8s.pod.name, k8s.deployment.name, k8s.container.name]
             labels: [{ tag_name: app, key: app, from: pod }]
           pod_association:
             - sources: [{ from: resource_attribute, name: k8s.pod.uid }]
         batch: {}
       exporters:
         otlp/gateway:
           endpoint: otel-collector.observability.svc.cluster.local:4317
           tls: { insecure: true }
       service:
         pipelines:
           logs:    { receivers: [filelog], processors: [memory_limiter, k8sattributes, batch], exporters: [otlp/gateway] }
           metrics: { receivers: [kubeletstats, hostmetrics], processors: [memory_limiter, k8sattributes, batch], exporters: [otlp/gateway] }
   ```

   Because logs now arrive via `filelog`, set `OTEL_LOGS_EXPORTER=none` in the `Instrumentation` CR env to avoid duplicate log lines in Loki.

2. Enable the target allocator on the gateway to scrape `/actuator/prometheus` through `ServiceMonitor`s. Patch the gateway CR:

   ```yaml
   spec:
     mode: statefulset        # target allocator requires statefulset mode
     targetAllocator:
       enabled: true
       serviceAccount: otel-collector
       prometheusCR:
         enabled: true
         serviceMonitorSelector: { matchLabels: { otel: "true" } }
         podMonitorSelector: {}
     config:
       receivers:
         prometheus:
           config:
             scrape_configs: []      # populated by the allocator
   ```

   and add `prometheus` to the metrics pipeline receivers. The allocator also needs `get/list/watch` on `servicemonitors`, `podmonitors` (monitoring.coreos.com).

3. Create a `ServiceMonitor` for `payments-api`:

   ```yaml
   apiVersion: monitoring.coreos.com/v1
   kind: ServiceMonitor
   metadata: { name: payments-api, namespace: capstone, labels: { otel: "true" } }
   spec:
     selector: { matchLabels: { app: payments-api } }
     endpoints: [{ port: http, path: /actuator/prometheus, interval: 15s }]
   ```

4. Verify the allocator assigned the target:

   ```bash
   kubectl --namespace observability port-forward svc/otel-collector-targetallocator 8080:80 &
   curl -s localhost:8080/jobs | jq .
   curl -s "localhost:8080/jobs/serviceMonitor%2Fcapstone%2Fpayments-api%2F0/targets" | jq '.[].targets'
   ```

5. Confirm in Prometheus:

   ```promql
   up{job=~".*payments-api.*"}
   container_cpu_usage_seconds_total{k8s_namespace_name="capstone"}   # from kubeletstats via remote write (name may be k8s_pod_cpu_time)
   ```

### Verify

Loki query `{k8s_namespace_name="capstone", k8s_container_name="api"} | json | level="INFO"` returns `orders-api` lines, each with `trace_id`; Prometheus has `k8s_pod_cpu_time_seconds_total` or equivalent from kubeletstats.

### Cleanup

Keep for Lab 8.

### What you learned

The agent/gateway split; `filelog` plus the `container` operator replaces a dedicated log shipper; the target allocator lets the Collector consume Prometheus Operator CRDs.

---

## Lab 8: Tracing an LLM Call with gen_ai Conventions

**Objective.** Add an endpoint to `orders-api` that calls an LLM (mocked or via Bedrock from cluster 02) and produce spans with `gen_ai.*` attributes, token-usage metrics and a TTFT histogram; build a Grafana panel of tokens per model.

**Prerequisites.** Lab 6; optional: AWS credentials via IRSA for Bedrock (never static keys).

**Time.** 60 minutes.

### Steps

1. Add `app/llm.py` using the snippet from the cheatsheet section 2 ("LLM call with gen_ai conventions"), with a mock client when `LLM_MOCK=true`:

   ```python
   class MockClient:
       def stream(self, model_id, messages):
           import time, random
           time.sleep(random.uniform(0.2, 0.9))          # TTFT
           for word in "Your order 42 has shipped and will arrive tomorrow".split():
               time.sleep(0.02)
               yield type("Chunk", (), {"text": word + " "})
       def last_usage(self):
           return type("Usage", (), {"input_tokens": 812, "output_tokens": 11, "stop_reason": "end_turn"})
   ```

2. Add the endpoint and a tool-call child span:

   ```python
   @app.post("/orders/{order_id}/assist")
   async def assist(order_id: str):
       with tracer.start_as_current_span("invoke_agent order-assistant", kind=SpanKind.INTERNAL,
                                         attributes={"gen_ai.operation.name": "invoke_agent", "gen_ai.agent.name": "order-assistant"}):
           with tracer.start_as_current_span("execute_tool lookup_order", attributes={"gen_ai.tool.name": "lookup_order", "gen_ai.tool.call.id": "call_1"}):
               order = await _lookup(order_id)
           return chat(client, MODEL_ID, [{"role": "user", "content": f"Where is order {order_id}?"}])
   ```

3. Call it 50 times and inspect a trace in Tempo. Expected span tree:

   ```text
   POST /orders/{order_id}/assist  (SERVER)
   └── invoke_agent order-assistant
       ├── execute_tool lookup_order
       │   └── SELECT orders
       └── chat <MODEL_ID>  (CLIENT)  gen_ai.usage.input_tokens=812 gen_ai.usage.output_tokens=11
           event: gen_ai.first_token
   ```

4. Query token usage and TTFT in Prometheus (metric names after OTLP -> Prometheus conversion):

   ```promql
   sum by (gen_ai_request_model, gen_ai_token_type) (rate(gen_ai_client_token_usage_token_sum[5m]))
   ```

   ```promql
   histogram_quantile(0.95, sum by (le) (rate(gen_ai_client_time_to_first_token_seconds_bucket[5m])))
   ```

5. Add a tail-sampling policy that keeps every LLM trace (`string_attribute` on `gen_ai.system`) so cost analysis has complete data, and confirm in `otelcol_processor_tail_sampling_count_traces_sampled{policy="llm-calls"}`.

### Verify

TraceQL `{ span.gen_ai.operation.name = "chat" } | avg(span.gen_ai.usage.output_tokens)` returns a value; the Grafana panel shows input vs output tokens per model.

### Cleanup

```bash
kubectl delete namespace capstone
helm --namespace observability uninstall opentelemetry-operator
kubectl delete namespace observability cert-manager
docker compose down -v
```

### What you learned

LLM calls are just spans with a standard attribute vocabulary; token counts and TTFT become metrics with the same cardinality discipline as HTTP metrics; tail sampling can treat them as always-interesting.

---

## Where to Go Next

- Harden what you built: [05-production-patterns.md](05-production-patterns.md).
- When something in these labs did not work: [06-troubleshooting.md](06-troubleshooting.md).
- Turn the span metrics into SLOs and burn-rate alerts: [../prometheus/05-production-patterns.md](../prometheus/05-production-patterns.md) and [../../11-sre/README.md](../../11-sre/README.md).
