# OpenTelemetry: Commands and Cheatsheet

| | |
|---|---|
| **Cluster** | 09 Observability |
| **Role** | Anchor |
| **Prerequisites** | [02-core-concepts.md](02-core-concepts.md) |
| **Target version** | Collector 0.110+, Python SDK 1.27+, Java agent 2.x, Operator 0.110+ |
| **Capstone phase** | Phase 4 (Months 13-16) |

> A task-organised reference: the `OTEL_*` environment variables, Python and Java instrumentation snippets, Collector component configurations with every commonly used option, OTTL statements, `kubectl` commands for the Operator, and the debugging endpoints. Every block is copy-pasteable.

---

## 1. Environment Variables (SDK and agents)

| Variable | Default | Meaning |
|---|---|---|
| `OTEL_SERVICE_NAME` | `unknown_service` | `service.name` resource attribute. Always set |
| `OTEL_RESOURCE_ATTRIBUTES` | | comma-separated `key=value` resource attributes: `service.namespace=capstone,deployment.environment=prod,service.version=1.4.2` |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | `http://localhost:4317` | base endpoint; per-signal overrides `OTEL_EXPORTER_OTLP_TRACES_ENDPOINT`, `..._METRICS_ENDPOINT`, `..._LOGS_ENDPOINT` (for HTTP append `/v1/traces` etc. yourself when using per-signal vars) |
| `OTEL_EXPORTER_OTLP_PROTOCOL` | `grpc` (Java: `grpc`; Python distro default `grpc`, auto-instr image `http/protobuf`) | `grpc`, `http/protobuf`, `http/json` |
| `OTEL_EXPORTER_OTLP_HEADERS` | | `Authorization=Bearer <TOKEN>,X-Scope-OrgID=tenant-a` (URL-encoded) |
| `OTEL_EXPORTER_OTLP_CERTIFICATE`, `..._CLIENT_KEY`, `..._CLIENT_CERTIFICATE` | | TLS CA / mTLS material paths |
| `OTEL_EXPORTER_OTLP_TIMEOUT` | `10000` ms | export timeout |
| `OTEL_EXPORTER_OTLP_COMPRESSION` | none | `gzip` |
| `OTEL_TRACES_EXPORTER` / `OTEL_METRICS_EXPORTER` / `OTEL_LOGS_EXPORTER` | `otlp` | `otlp`, `console`, `none`, `prometheus` (metrics), comma-separated for several |
| `OTEL_TRACES_SAMPLER` | `parentbased_always_on` | `always_on`, `always_off`, `traceidratio`, `parentbased_always_on`, `parentbased_always_off`, `parentbased_traceidratio` |
| `OTEL_TRACES_SAMPLER_ARG` | | ratio for `*traceidratio`, e.g. `0.1` |
| `OTEL_PROPAGATORS` | `tracecontext,baggage` | add `b3`, `b3multi`, `jaeger`, `xray`, `ottrace` |
| `OTEL_BSP_SCHEDULE_DELAY` | `5000` | ms between batch exports |
| `OTEL_BSP_MAX_QUEUE_SIZE` | `2048` | span queue size |
| `OTEL_BSP_MAX_EXPORT_BATCH_SIZE` | `512` | spans per export |
| `OTEL_BLRP_*` | same as BSP | for log records |
| `OTEL_METRIC_EXPORT_INTERVAL` | `60000` | ms between metric exports |
| `OTEL_METRICS_EXEMPLAR_FILTER` | `trace_based` | `always_on`, `always_off`, `trace_based` |
| `OTEL_EXPORTER_OTLP_METRICS_TEMPORALITY_PREFERENCE` | `cumulative` | `cumulative`, `delta`, `lowmemory` |
| `OTEL_EXPORTER_OTLP_METRICS_DEFAULT_HISTOGRAM_AGGREGATION` | `explicit_bucket_histogram` | or `base2_exponential_bucket_histogram` |
| `OTEL_ATTRIBUTE_VALUE_LENGTH_LIMIT` | unlimited | truncate attribute values |
| `OTEL_ATTRIBUTE_COUNT_LIMIT` | `128` | per span/log |
| `OTEL_SPAN_EVENT_COUNT_LIMIT`, `OTEL_SPAN_LINK_COUNT_LIMIT` | `128` | |
| `OTEL_SDK_DISABLED` | `false` | `true` turns the SDK into no-ops |
| `OTEL_LOG_LEVEL` | `info` | SDK's own logging (`debug` to see export errors) |
| `OTEL_SEMCONV_STABILITY_OPT_IN` | | `http`, `http/dup`, `database` to emit new attribute names |
| `OTEL_EXPERIMENTAL_RESOURCE_DETECTORS` | `otel`, `process` | Python: add `aws_ec2`, `aws_ecs`, `aws_eks`, `gcp`, `azure_vm`, `container` |

### Python-specific

| Variable | Meaning |
|---|---|
| `OTEL_PYTHON_LOGGING_AUTO_INSTRUMENTATION_ENABLED=true` | attach OTLP handler to root logger |
| `OTEL_PYTHON_LOG_CORRELATION=true` | inject trace ids into `logging` records |
| `OTEL_PYTHON_LOG_LEVEL=info` | minimum level exported |
| `OTEL_PYTHON_DISABLED_INSTRUMENTATIONS=urllib3,redis` | skip instrumentations |
| `OTEL_PYTHON_EXCLUDED_URLS=healthz,readyz,metrics` | FastAPI/ASGI: do not trace these paths |
| `OTEL_PYTHON_FASTAPI_EXCLUDED_URLS` | per-framework variant |
| `OTEL_PYTHON_ID_GENERATOR=xray` | for AWS X-Ray compatible IDs |

### Java agent-specific

| Variable / property | Meaning |
|---|---|
| `JAVA_TOOL_OPTIONS=-javaagent:/otel/opentelemetry-javaagent.jar` | attach agent |
| `OTEL_JAVAAGENT_DEBUG=true` | verbose agent logs |
| `OTEL_JAVAAGENT_ENABLED=false` | disable agent at runtime |
| `OTEL_INSTRUMENTATION_<NAME>_ENABLED=false` | disable one instrumentation, e.g. `OTEL_INSTRUMENTATION_JDBC_DATASOURCE_ENABLED` |
| `OTEL_INSTRUMENTATION_COMMON_DEFAULT_ENABLED=false` | opt-in mode: disable all, enable selectively |
| `OTEL_INSTRUMENTATION_MICROMETER_ENABLED=true` | bridge Micrometer registry to OTel metrics |
| `OTEL_INSTRUMENTATION_RUNTIME_TELEMETRY_EMIT_EXPERIMENTAL_TELEMETRY=true` | JVM GC/memory/thread metrics (newer semconv) |
| `OTEL_INSTRUMENTATION_HTTP_SERVER_CAPTURE_REQUEST_HEADERS=x-request-id` | capture headers as attributes |
| `OTEL_INSTRUMENTATION_HTTP_KNOWN_METHODS` | allowed method names |
| `OTEL_INSTRUMENTATION_LOGBACK_APPENDER_EXPERIMENTAL_LOG_ATTRIBUTES=true` | include MDC and thread attributes in OTLP logs |
| `OTEL_JAVAAGENT_EXCLUDE_CLASSES` | skip bytecode instrumentation for classes |
| `OTEL_JAVAAGENT_EXTENSIONS=/otel/ext.jar` | load custom extensions (samplers, span processors) |

---

## 2. Python Snippets

### Manual tracer, spans, events, status

```python
from opentelemetry import trace
from opentelemetry.trace import SpanKind, Status, StatusCode

tracer = trace.get_tracer("orders-api.checkout", "0.1.0")

def charge(order_id: str, amount_cents: int) -> None:
    with tracer.start_as_current_span(
        "charge order",
        kind=SpanKind.INTERNAL,
        attributes={"order.id": order_id, "order.amount_cents": amount_cents},
    ) as span:
        span.add_event("validation complete", {"rules.evaluated": 7})
        try:
            _call_payment_gateway(order_id, amount_cents)
            span.set_status(Status(StatusCode.OK))
        except TimeoutError as exc:
            span.record_exception(exc)                     # adds exception.* event
            span.set_status(Status(StatusCode.ERROR, "payment gateway timeout"))
            raise
```

### Span links (batch consumer)

```python
from opentelemetry.trace import Link
links = [Link(msg_ctx) for msg_ctx in extracted_producer_contexts]
with tracer.start_as_current_span("process batch", links=links, kind=SpanKind.CONSUMER):
    ...
```

### Metrics instruments

```python
from opentelemetry import metrics
meter = metrics.get_meter("orders-api", "0.1.0")

orders_created = meter.create_counter("orders.created", unit="{order}", description="Orders accepted")
in_flight      = meter.create_up_down_counter("orders.in_flight", unit="{order}")
charge_latency = meter.create_histogram("orders.charge.duration", unit="s")

def observe_queue_depth(options):
    yield metrics.Observation(queue.qsize(), {"queue": "charges"})
meter.create_observable_gauge("orders.queue.depth", callbacks=[observe_queue_depth], unit="{item}")

orders_created.add(1, {"tenant": tenant, "channel": "web"})   # keep attribute sets small
charge_latency.record(0.231, {"gateway": "stripe-sandbox", "outcome": "ok"})
```

### Programmatic SDK setup (when not using `opentelemetry-instrument`)

```python
from opentelemetry import trace, metrics
from opentelemetry.sdk.resources import Resource, SERVICE_NAME
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.sdk.trace.sampling import ParentBasedTraceIdRatio
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.sdk.metrics import MeterProvider
from opentelemetry.sdk.metrics.export import PeriodicExportingMetricReader
from opentelemetry.exporter.otlp.proto.grpc.metric_exporter import OTLPMetricExporter
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor
from opentelemetry.instrumentation.httpx import HTTPXClientInstrumentor
from opentelemetry.instrumentation.sqlalchemy import SQLAlchemyInstrumentor
from opentelemetry.instrumentation.logging import LoggingInstrumentor

resource = Resource.create({SERVICE_NAME: "orders-api", "service.namespace": "capstone"})

trace.set_tracer_provider(TracerProvider(resource=resource, sampler=ParentBasedTraceIdRatio(0.25)))
trace.get_tracer_provider().add_span_processor(BatchSpanProcessor(OTLPSpanExporter()))  # endpoint from env

metrics.set_meter_provider(MeterProvider(
    resource=resource,
    metric_readers=[PeriodicExportingMetricReader(OTLPMetricExporter(), export_interval_millis=15000)],
))

app = FastAPI()
FastAPIInstrumentor.instrument_app(app, excluded_urls="healthz,readyz,metrics")
HTTPXClientInstrumentor().instrument()
SQLAlchemyInstrumentor().instrument(engine=engine.sync_engine, enable_commenter=True)
LoggingInstrumentor().instrument(set_logging_format=True)
```

### Baggage

```python
from opentelemetry import baggage, context
ctx = baggage.set_baggage("tenant.id", tenant_id)
token = context.attach(ctx)
try:
    ...   # outbound calls now carry baggage: tenant.id=...
finally:
    context.detach(token)

# downstream: copy onto span
tenant = baggage.get_baggage("tenant.id")
trace.get_current_span().set_attribute("tenant.id", tenant)
```

### LLM call with gen_ai conventions (manual)

```python
import time
def chat(client, model_id: str, messages: list[dict]) -> dict:
    with tracer.start_as_current_span(f"chat {model_id}", kind=SpanKind.CLIENT) as span:
        span.set_attributes({
            "gen_ai.system": "aws.bedrock",
            "gen_ai.operation.name": "chat",
            "gen_ai.request.model": model_id,
            "gen_ai.request.max_tokens": 1024,
            "gen_ai.request.temperature": 0.2,
        })
        t0 = time.perf_counter()
        first_token_at = None
        output_chunks = []
        for chunk in client.stream(model_id, messages):      # provider streaming API
            if first_token_at is None:
                first_token_at = time.perf_counter()
                span.add_event("gen_ai.first_token")
                ttft_hist.record(first_token_at - t0, {"gen_ai.request.model": model_id})
            output_chunks.append(chunk.text)
        usage = client.last_usage()
        span.set_attributes({
            "gen_ai.usage.input_tokens": usage.input_tokens,
            "gen_ai.usage.output_tokens": usage.output_tokens,
            "gen_ai.response.finish_reasons": [usage.stop_reason],
        })
        token_usage.record(usage.input_tokens,  {"gen_ai.token.type": "input",  "gen_ai.request.model": model_id})
        token_usage.record(usage.output_tokens, {"gen_ai.token.type": "output", "gen_ai.request.model": model_id})
        return {"text": "".join(output_chunks)}

token_usage = meter.create_histogram("gen_ai.client.token.usage", unit="{token}")
ttft_hist   = meter.create_histogram("gen_ai.client.time_to_first_token", unit="s")
```

Tool calls made by an agent get a child span `execute_tool {tool_name}` with `gen_ai.tool.name` and `gen_ai.tool.call.id`. Do not record prompt or completion text unless `OTEL_INSTRUMENTATION_GENAI_CAPTURE_MESSAGE_CONTENT=true` is a deliberate, reviewed decision.

---

## 3. Java Snippets

### Manual spans with the API (agent present)

```java
import io.opentelemetry.api.GlobalOpenTelemetry;
import io.opentelemetry.api.trace.*;
import io.opentelemetry.context.Scope;

Tracer tracer = GlobalOpenTelemetry.getTracer("payments-api.ledger", "0.1.0");

public void settle(String paymentId) {
    Span span = tracer.spanBuilder("settle payment")
        .setSpanKind(SpanKind.INTERNAL)
        .setAttribute("payment.id", paymentId)
        .startSpan();
    try (Scope ignored = span.makeCurrent()) {
        ledger.post(paymentId);
        span.addEvent("ledger posted");
        span.setStatus(StatusCode.OK);
    } catch (Exception e) {
        span.recordException(e);
        span.setStatus(StatusCode.ERROR, e.getMessage());
        throw e;
    } finally {
        span.end();
    }
}
```

### Annotations (agent extension `opentelemetry-instrumentation-annotations`)

```java
import io.opentelemetry.instrumentation.annotations.WithSpan;
import io.opentelemetry.instrumentation.annotations.SpanAttribute;

@WithSpan("risk check")
public RiskResult check(@SpanAttribute("payment.id") String paymentId) { ... }
```

### Micrometer (bridged automatically when `OTEL_INSTRUMENTATION_MICROMETER_ENABLED=true`)

```java
@Component
public class PaymentMetrics {
    private final Counter settled;
    private final Timer gatewayLatency;
    public PaymentMetrics(MeterRegistry registry) {
        settled = Counter.builder("payments.settled").tag("gateway", "sandbox").register(registry);
        gatewayLatency = Timer.builder("payments.gateway.latency")
            .publishPercentileHistogram()
            .serviceLevelObjectives(Duration.ofMillis(100), Duration.ofMillis(300), Duration.ofSeconds(1))
            .register(registry);
    }
}
```

```properties
# application.properties: keep the Prometheus endpoint too, so Prometheus can scrape directly
management.endpoints.web.exposure.include=health,info,prometheus
management.metrics.distribution.percentiles-histogram.http.server.requests=true
management.observations.key-values.application=payments-api
management.tracing.sampling.probability=1.0
```

### Baggage in Java

```java
Baggage.current().toBuilder().put("tenant.id", tenantId).build().makeCurrent();
String tenant = Baggage.current().getEntryValue("tenant.id");
```

---

## 4. Collector CLI

```bash
otelcol-contrib --config=file:/etc/otelcol/config.yaml                 # single file
otelcol-contrib --config=/etc/otelcol/base.yaml --config=/etc/otelcol/overlay.yaml   # merged in order
otelcol-contrib --config="yaml:exporters::debug::verbosity: detailed"  # inline override
otelcol-contrib --config=env:OTEL_CONFIG                               # from env var
otelcol-contrib validate --config=/etc/otelcol/config.yaml             # parse + component validation
otelcol-contrib components                                             # list compiled-in components
otelcol-contrib --version
otelcol-contrib --feature-gates=+exporter.prometheusremotewrite.exemplars   # toggle gates
```

Environment expansion inside config: `${env:LOKI_ENDPOINT}` or `${env:TENANT:-default}`.

Debug endpoints (with extensions enabled):

```bash
curl -s localhost:13133/                      # health_check
curl -s localhost:55679/debug/servicez        # zpages: build info, pipelines
curl -s localhost:55679/debug/pipelinez
curl -s localhost:55679/debug/tracez          # recent spans processed by the collector itself
curl -s localhost:8888/metrics | grep -E 'otelcol_(receiver_accepted|receiver_refused|exporter_sent|exporter_send_failed|exporter_queue_size|processor_dropped|processor_batch)'
go tool pprof http://localhost:1777/debug/pprof/heap     # pprof extension
```

Key self-metrics:

| Metric | Meaning |
|---|---|
| `otelcol_receiver_accepted_spans/_metric_points/_log_records` | items received |
| `otelcol_receiver_refused_*` | rejected (memory_limiter backpressure or bad data) |
| `otelcol_processor_dropped_*` (or `otelcol_processor_filter_*`) | dropped by processors |
| `otelcol_exporter_sent_*`, `otelcol_exporter_send_failed_*` | export success/failure |
| `otelcol_exporter_queue_size`, `otelcol_exporter_queue_capacity` | sending queue health |
| `otelcol_exporter_enqueue_failed_*` | queue full, data lost |
| `otelcol_process_memory_rss`, `otelcol_process_runtime_heap_alloc_bytes` | memory |
| `otelcol_processor_tail_sampling_count_traces_sampled` | tail sampling decisions by policy |

---

## 5. Collector Component Reference

### Receivers

```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
        max_recv_msg_size_mib: 16
        tls:
          cert_file: /etc/otelcol/tls/tls.crt
          key_file: /etc/otelcol/tls/tls.key
          client_ca_file: /etc/otelcol/tls/ca.crt    # mTLS
        auth:
          authenticator: bearertokenauth
      http:
        endpoint: 0.0.0.0:4318
        cors:
          allowed_origins: ["https://app.example.com"]

  prometheus:
    config:
      scrape_configs:
        - job_name: otel-collector-self
          scrape_interval: 15s
          static_configs: [{ targets: ["localhost:8888"] }]
        - job_name: payments-api
          metrics_path: /actuator/prometheus
          kubernetes_sd_configs:
            - role: pod
              namespaces: { names: [capstone] }
          relabel_configs:
            - source_labels: [__meta_kubernetes_pod_label_app]
              regex: payments-api
              action: keep
            - source_labels: [__meta_kubernetes_pod_ip]
              target_label: __address__
              replacement: $1:8080

  filelog:
    include: [/var/log/pods/*/*/*.log]
    exclude: [/var/log/pods/observability_*/*/*.log]
    start_at: end
    include_file_path: true
    retry_on_failure: { enabled: true }
    operators:
      - type: container          # parses CRI-O / containerd / docker formats and sets k8s.* attributes from the path
        id: container-parser
      - type: json_parser
        if: 'body matches "^\\{"'
        parse_from: body
        parse_to: attributes
        timestamp: { parse_from: attributes.ts, layout: '%Y-%m-%dT%H:%M:%S%z' }
        severity: { parse_from: attributes.level }
        trace: { trace_id: { parse_from: attributes.trace_id }, span_id: { parse_from: attributes.span_id } }
      - type: recombine         # Java stack traces
        combine_field: body
        is_first_entry: 'body matches "^\\d{4}-\\d{2}-\\d{2}"'
        source_identifier: attributes["log.file.path"]
        max_log_size: 102400

  kubeletstats:
    collection_interval: 30s
    auth_type: serviceAccount
    endpoint: https://${env:K8S_NODE_NAME}:10250
    insecure_skip_verify: true   # kubelet serving certs are often self-signed; prefer a CA bundle in production
    metric_groups: [node, pod, container]

  k8s_cluster:
    auth_type: serviceAccount
    node_conditions_to_report: [Ready, MemoryPressure, DiskPressure]
    allocatable_types_to_report: [cpu, memory]

  hostmetrics:
    collection_interval: 30s
    root_path: /hostfs
    scrapers:
      cpu: {}
      memory: {}
      load: {}
      disk: {}
      filesystem:
        exclude_mount_points: { mount_points: ["/var/lib/kubelet/*"], match_type: regexp }
      network: {}
      processes: {}

  k8sobjects:
    objects:
      - name: events
        mode: watch
        group: events.k8s.io
```

### Processors

```yaml
processors:
  memory_limiter:
    check_interval: 1s
    limit_mib: 1536            # hard limit (or limit_percentage)
    spike_limit_mib: 384       # limit - spike = soft limit where GC is forced

  batch:
    send_batch_size: 8192
    send_batch_max_size: 16384
    timeout: 5s

  k8sattributes:
    auth_type: serviceAccount
    passthrough: false
    filter:
      node_from_env_var: K8S_NODE_NAME        # agent mode: only pods on this node
    extract:
      metadata: [k8s.namespace.name, k8s.pod.name, k8s.pod.uid, k8s.deployment.name, k8s.node.name, k8s.container.name]
      labels:
        - { tag_name: app, key: app, from: pod }
        - { tag_name: team, key: team, from: namespace }
      annotations:
        - { tag_name: runbook, key: runbook.example.com/url, from: pod }
    pod_association:
      - sources: [{ from: resource_attribute, name: k8s.pod.uid }]
      - sources: [{ from: resource_attribute, name: k8s.pod.ip }]
      - sources: [{ from: connection }]

  resource:
    attributes:
      - { key: deployment.environment, value: prod, action: upsert }
      - { key: k8s.cluster.name, value: capstone-eks, action: insert }
      - { key: process.command_line, action: delete }        # can contain secrets

  resourcedetection:
    detectors: [env, system, eks, ec2]
    timeout: 5s
    override: false

  attributes/pii:
    actions:
      - { key: user.email, action: delete }
      - { key: user.id, action: hash }
      - { key: http.request.header.authorization, action: delete }
      - key: enduser.id
        action: extract
        pattern: ^(?P<tenant>[a-z0-9-]+):

  redaction:
    allow_all_keys: true
    blocked_values:
      - "4[0-9]{12}(?:[0-9]{3})?"                # 16-digit card-like numbers
      - "[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\\.[A-Za-z]{2,}"   # email addresses
      - "(?i)bearer\\s+[a-z0-9._-]+"
    ignored_keys: [http.route, url.path]
    summary: debug                               # add redaction summary attributes

  filter/drop-noise:
    error_mode: ignore
    traces:
      span:
        - 'attributes["http.route"] == "/healthz"'
        - 'attributes["http.route"] == "/readyz"'
        - 'IsMatch(attributes["url.path"], "^/metrics$")'
    logs:
      log_record:
        - 'severity_number < SEVERITY_NUMBER_INFO and resource.attributes["deployment.environment"] == "prod"'
    metrics:
      metric:
        - 'name == "http.server.active_requests" and resource.attributes["service.name"] == "noisy-svc"'

  transform/normalise:
    error_mode: ignore
    trace_statements:
      - context: span
        statements:
          - set(attributes["http.request.method"], attributes["http.method"]) where attributes["http.method"] != nil
          - delete_key(attributes, "http.method")
          - truncate_all(attributes, 512)
          - set(status.code, STATUS_CODE_ERROR) where attributes["http.response.status_code"] >= 500
          - replace_pattern(attributes["db.query.text"], "'[^']*'", "?")
    log_statements:
      - context: log
        statements:
          - merge_maps(attributes, ParseJSON(body), "upsert") where IsMatch(body, "^\\{")
          - set(severity_text, attributes["level"]) where attributes["level"] != nil
          - set(trace_id.string, attributes["trace_id"]) where attributes["trace_id"] != nil
          - set(span_id.string, attributes["span_id"]) where attributes["span_id"] != nil
          - delete_matching_keys(attributes, "^(password|secret|token).*")
    metric_statements:
      - context: datapoint
        statements:
          - delete_key(attributes, "client.address")      # unbounded cardinality
          - delete_key(attributes, "user_agent.original")

  probabilistic_sampler:
    sampling_percentage: 20
    mode: proportional          # hash_seed | proportional | equalizing

  tail_sampling:
    decision_wait: 10s
    num_traces: 100000
    expected_new_traces_per_sec: 1000
    policies:
      - { name: errors,        type: status_code, status_code: { status_codes: [ERROR] } }
      - { name: slow,          type: latency, latency: { threshold_ms: 2000 } }
      - name: five-xx
        type: numeric_attribute
        numeric_attribute: { key: http.response.status_code, min_value: 500, max_value: 599 }
      - name: llm-calls
        type: string_attribute
        string_attribute: { key: gen_ai.system, values: [".*"], enabled_regex_matching: true }
      - { name: baseline,      type: probabilistic, probabilistic: { sampling_percentage: 5 } }
      - { name: cap,           type: rate_limiting, rate_limiting: { spans_per_second: 2000 } }
      - name: and-example
        type: and
        and:
          and_sub_policy:
            - { name: tenant-acme, type: string_attribute, string_attribute: { key: tenant.id, values: [acme] } }
            - { name: p50,         type: probabilistic, probabilistic: { sampling_percentage: 50 } }

  cumulativetodelta: {}
  deltatocumulative: { max_stale: 5m }
```

### Connectors

```yaml
connectors:
  spanmetrics:
    namespace: traces.spanmetrics
    histogram:
      explicit:
        buckets: [5ms, 10ms, 25ms, 50ms, 100ms, 250ms, 300ms, 500ms, 1s, 2500ms, 5s, 10s]
    dimensions:
      - { name: http.request.method }
      - { name: http.route }
      - { name: http.response.status_code }
      - { name: deployment.environment, default: unknown }
    exemplars: { enabled: true }
    metrics_flush_interval: 15s
    exclude_dimensions: [span.kind]
    resource_metrics_key_attributes: [service.name, service.namespace]

  servicegraph:
    latency_histogram_buckets: [10ms, 50ms, 100ms, 250ms, 500ms, 1s, 2s, 5s]
    dimensions: [deployment.environment]
    store: { ttl: 2s, max_items: 10000 }

  count:
    logs:
      logs.errors:
        description: Error log records
        conditions: ['severity_number >= SEVERITY_NUMBER_ERROR']
        attributes: [{ key: service.name }]
```

### Exporters

```yaml
exporters:
  debug:
    verbosity: detailed
    sampling_initial: 5
    sampling_thereafter: 200

  otlp/tempo:
    endpoint: tempo.observability.svc.cluster.local:4317
    tls: { insecure: true }                # in-cluster lab; use cert_file/ca_file in production
    sending_queue: { enabled: true, num_consumers: 10, queue_size: 5000, storage: file_storage }
    retry_on_failure: { enabled: true, initial_interval: 5s, max_interval: 30s, max_elapsed_time: 300s }
    headers: { X-Scope-OrgID: capstone }

  otlphttp/loki:
    endpoint: http://loki-gateway.observability.svc/otlp
    tls: { insecure: true }
    headers: { X-Scope-OrgID: capstone }

  prometheusremotewrite:
    endpoint: http://kps-kube-prometheus-stack-prometheus.observability.svc:9090/api/v1/write
    resource_to_telemetry_conversion: { enabled: true }   # copy resource attrs to metric labels (watch cardinality)
    target_info: { enabled: true }
    add_metric_suffixes: true
    external_labels: { cluster: capstone-eks }
    remote_write_queue: { enabled: true, queue_size: 10000, num_consumers: 5 }
    max_batch_size_bytes: 3000000

  prometheus:
    endpoint: 0.0.0.0:8889
    namespace: otel
    enable_open_metrics: true     # required for exemplars on the scrape endpoint
    resource_to_telemetry_conversion: { enabled: true }

  elasticsearch:
    endpoints: ["https://elasticsearch-es-http.observability.svc:9200"]
    api_key: ${env:ELASTIC_API_KEY}
    mapping: { mode: otel }
    logs_dynamic_index: { enabled: true }
    tls: { ca_file: /etc/otelcol/es-ca.crt }

  loadbalancing:
    routing_key: traceID
    protocol:
      otlp: { tls: { insecure: true } }
    resolver:
      k8s:
        service: otel-gateway-headless.observability
        ports: [4317]

  otlphttp/grafana-cloud:
    endpoint: https://otlp-gateway-<REGION>.grafana.net/otlp
    auth: { authenticator: basicauth/grafana }
```

### Extensions

```yaml
extensions:
  health_check: { endpoint: 0.0.0.0:13133, path: /healthz }
  pprof: { endpoint: 0.0.0.0:1777 }
  zpages: { endpoint: 0.0.0.0:55679 }
  file_storage:
    directory: /var/lib/otelcol/queue
    timeout: 10s
  bearertokenauth:
    scheme: Bearer
    filename: /var/run/secrets/otel/token        # mounted secret; never inline
  basicauth/grafana:
    client_auth:
      username: "${env:GRAFANA_INSTANCE_ID}"
      password: "${env:GRAFANA_API_TOKEN}"
  oauth2client:
    client_id: ${env:OAUTH_CLIENT_ID}
    client_secret: ${env:OAUTH_CLIENT_SECRET}
    token_url: https://idp.example.com/oauth2/token
    scopes: [telemetry.write]
  sigv4auth:
    region: <REGION>
    service: aps                                   # Amazon Managed Prometheus
```

---

## 6. OTTL Quick Reference

| Need | Statement |
|---|---|
| Set attribute | `set(attributes["env"], "prod")` |
| Copy resource attr to span | `set(attributes["svc"], resource.attributes["service.name"])` |
| Delete key / pattern | `delete_key(attributes, "x")`, `delete_matching_keys(attributes, "^tmp_")` |
| Keep only some keys | `keep_keys(attributes, ["http.route", "http.request.method"])` |
| Rename metric | `set(name, "http_server_duration_seconds") where name == "http.server.request.duration"` |
| Parse JSON body | `merge_maps(attributes, ParseJSON(body), "upsert")` |
| Regex extract | `merge_maps(attributes, ExtractPatterns(body, "^(?P<level>\\w+) (?P<msg>.*)$"), "insert")` |
| Replace pattern | `replace_pattern(attributes["url.full"], "token=[^&]+", "token=REDACTED")` |
| Replace in all string attrs | `replace_all_patterns(attributes, "value", "\\d{16}", "****")` |
| Truncate | `truncate_all(attributes, 256)`, `limit(attributes, 50, [])` |
| Severity from text | `set(severity_number, SEVERITY_NUMBER_ERROR) where IsMatch(body, "(?i)error")` |
| Span status | `set(status.code, STATUS_CODE_ERROR) where attributes["http.response.status_code"] >= 500` |
| Span name from route | `set(name, Concat([attributes["http.request.method"], attributes["http.route"]], " ")) where attributes["http.route"] != nil` |
| Hash | `set(attributes["user.id"], SHA256(attributes["user.id"]))` |
| Time | `set(time, Now()) where time_unix_nano == 0` |
| Convert metric type | `convert_sum_to_gauge()`, `convert_gauge_to_sum("cumulative", true)` |
| Conditions | `where attributes["x"] != nil and IsMatch(attributes["y"], "^a")` |

Contexts: `resource`, `scope`, `span`, `spanevent`, `metric`, `datapoint`, `log`. Error mode `ignore` (skip failing statement), `silent`, `propagate` (drop batch).

---

## 7. Kubernetes and Operator Commands

```bash
# Operator and CRDs
kubectl get opentelemetrycollectors --all-namespaces            # short: otelcol
kubectl get instrumentations --all-namespaces                   # short: otelinst
kubectl --namespace observability describe otelcol otel-collector
kubectl --namespace observability logs deploy/opentelemetry-operator -c manager --tail=100

# Rendered config the Operator produced
kubectl --namespace observability get configmap otel-collector-collector -o jsonpath='{.data.collector\.yaml}'

# Collector pods and logs
kubectl --namespace observability get pods -l app.kubernetes.io/component=opentelemetry-collector
kubectl --namespace observability logs -l app.kubernetes.io/component=opentelemetry-collector --tail=200 -f
kubectl --namespace observability port-forward svc/otel-collector 4317:4317 4318:4318 13133:13133 55679:55679

# Was auto-instrumentation injected?
kubectl --namespace capstone get pod -l app=orders-api -o jsonpath='{range .items[0].spec.containers[*]}{.name}{"\n"}{range .env[*]}  {.name}={.value}{"\n"}{end}{end}' | grep -E 'OTEL_|PYTHONPATH|JAVA_TOOL'
kubectl --namespace capstone get pod -l app=payments-api -o jsonpath='{.items[0].spec.initContainers[*].image}'

# Force re-injection after editing the Instrumentation CR
kubectl --namespace capstone rollout restart deployment/orders-api deployment/payments-api

# Target allocator (when enabled)
kubectl --namespace observability port-forward svc/otel-collector-targetallocator 8080:80
curl -s localhost:8080/jobs | jq .
curl -s localhost:8080/scrape_configs | jq 'keys'

# Send a test trace from inside the cluster
kubectl run otel-test --rm -it --restart=Never --image=curlimages/curl:8.10.1 -- \
  curl -s -X POST http://otel-collector.observability.svc.cluster.local:4318/v1/traces \
  -H 'Content-Type: application/json' \
  -d '{"resourceSpans":[{"resource":{"attributes":[{"key":"service.name","value":{"stringValue":"k8s-test"}}]},"scopeSpans":[{"spans":[{"traceId":"00000000000000000000000000000abc","spanId":"0000000000000abc","name":"test","kind":1,"startTimeUnixNano":"1700000000000000000","endTimeUnixNano":"1700000001000000000"}]}]}]}'
```

### Minimal RBAC for `k8sattributes`, `kubeletstats`, `k8s_cluster`

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata: { name: otel-collector }
rules:
  - apiGroups: [""]
    resources: [pods, namespaces, nodes, nodes/stats, nodes/proxy, services, endpoints, events, replicationcontrollers, resourcequotas]
    verbs: [get, list, watch]
  - apiGroups: [apps]
    resources: [replicasets, deployments, daemonsets, statefulsets]
    verbs: [get, list, watch]
  - apiGroups: [batch]
    resources: [jobs, cronjobs]
    verbs: [get, list, watch]
  - apiGroups: [autoscaling]
    resources: [horizontalpodautoscalers]
    verbs: [get, list, watch]
  - apiGroups: [events.k8s.io]
    resources: [events]
    verbs: [get, list, watch]
```

---

## 8. Testing and Debugging Commands

```bash
# Python: print spans to console instead of exporting
OTEL_TRACES_EXPORTER=console OTEL_METRICS_EXPORTER=console OTEL_LOGS_EXPORTER=console \
  opentelemetry-instrument python -m app.main

# Python: see SDK export errors
OTEL_LOG_LEVEL=debug opentelemetry-instrument uvicorn app.main:app

# Java: console exporter and agent debug
OTEL_TRACES_EXPORTER=logging OTEL_JAVAAGENT_DEBUG=true java -javaagent:/otel/opentelemetry-javaagent.jar -jar app.jar

# gRPC reachability from the app container
grpcurl -plaintext otel-collector.observability.svc.cluster.local:4317 list    # should list opentelemetry.proto.collector.*
nc -zv otel-collector.observability.svc.cluster.local 4317

# HTTP reachability
curl -sv http://otel-collector.observability.svc.cluster.local:4318/v1/traces -X POST -H 'Content-Type: application/json' -d '{}'
# expect HTTP 200 and {"partialSuccess":{}}

# Generate load locally
hey -z 60s -q 20 http://localhost:8000/orders/42
# or
for i in $(seq 1 200); do curl -s -o /dev/null localhost:8000/orders/$i; done

# telemetrygen: synthetic OTLP traffic (from opentelemetry-collector-contrib)
go install github.com/open-telemetry/opentelemetry-collector-contrib/cmd/telemetrygen@latest
telemetrygen traces --otlp-endpoint localhost:4317 --otlp-insecure --traces 100 --child-spans 3
telemetrygen metrics --otlp-endpoint localhost:4317 --otlp-insecure --duration 30s
telemetrygen logs --otlp-endpoint localhost:4317 --otlp-insecure --logs 50
```

---

## 9. One-Line Reminders

- `memory_limiter` first, `batch` last.
- `spanmetrics` before `tail_sampling` if you want exact RED metrics.
- `parentbased_traceidratio`, never bare `traceidratio`, in distributed systems.
- `resource_to_telemetry_conversion` turns every resource attribute into a Prometheus label: enable it only after trimming resource attributes.
- The Collector's own metrics on `:8888` answer "where did my data go" faster than any log.
- `otelcol-contrib validate` in CI before every deploy.
