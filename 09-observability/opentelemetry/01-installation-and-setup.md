# OpenTelemetry: Installation and Setup

| | |
|---|---|
| **Cluster** | 09 Observability |
| **Role** | Anchor |
| **Prerequisites** | Docker, Python 3.12+ with `pip`, Java 21 with Maven or Gradle, `kubectl` + Helm 3 for the Kubernetes section |
| **Target version** | Collector 0.110+ (contrib), Python SDK 1.27+, Java agent 2.x, Operator 0.110+ |
| **Capstone phase** | Phase 4 (Months 13-16) |

> After this file you will have the Collector running locally (binary and Docker), the Python SDK plus auto-instrumentation installed for `orders-api`, the Java agent attached to `payments-api`, a `debug` exporter proving data flows end to end, and the OTel Operator installed on a Kubernetes cluster. Each step ends with a verification command.

---

## 1. Choose Your Distribution

| Component | Distribution | When to use |
|---|---|---|
| Collector | `otelcol` (core) | Minimal: OTLP in/out, batch, memory_limiter, debug. Smallest attack surface |
| Collector | `otelcol-contrib` | Almost always what you want: `k8sattributes`, `filelog`, `tail_sampling`, `prometheusremotewrite`, `spanmetrics`, `redaction`, `transform` |
| Collector | Custom build with `ocb` | Production: include only the components you use, pin versions, smaller image, fewer CVEs |
| Collector | Vendor distros (ADOT, Grafana Alloy, Splunk, Elastic EDOT) | When you need vendor-specific exporters or support; all still speak OTLP |
| Python | `opentelemetry-distro` + `opentelemetry-instrument` | Zero-code; installs SDK, OTLP exporter and all detected instrumentations |
| Java | `opentelemetry-javaagent.jar` | Zero-code; 150+ libraries instrumented by bytecode manipulation |
| Java | `opentelemetry-spring-boot-starter` | When the agent is not allowed (GraalVM native images, startup-time constraints) |

---

## 2. Collector: Local Binary

### Linux / macOS

```bash
OTELCOL_VERSION=0.115.0
ARCH=$(uname -m | sed 's/x86_64/amd64/;s/aarch64/arm64/')
OS=$(uname -s | tr '[:upper:]' '[:lower:]')
curl -fsSLO "https://github.com/open-telemetry/opentelemetry-collector-releases/releases/download/v${OTELCOL_VERSION}/otelcol-contrib_${OTELCOL_VERSION}_${OS}_${ARCH}.tar.gz"
tar -xzf "otelcol-contrib_${OTELCOL_VERSION}_${OS}_${ARCH}.tar.gz" otelcol-contrib
sudo install -m 0755 otelcol-contrib /usr/local/bin/otelcol-contrib
otelcol-contrib --version
```

Expected output:

```text
otelcol-contrib version 0.115.0
```

On Debian/Ubuntu the `.deb` package installs a `systemd` unit and `/etc/otelcol-contrib/config.yaml`:

```bash
curl -fsSLO "https://github.com/open-telemetry/opentelemetry-collector-releases/releases/download/v${OTELCOL_VERSION}/otelcol-contrib_${OTELCOL_VERSION}_linux_amd64.deb"
sudo dpkg -i "otelcol-contrib_${OTELCOL_VERSION}_linux_amd64.deb"
sudo systemctl status otelcol-contrib --no-pager
```

### Windows (PowerShell)

```powershell
$v = "0.115.0"
Invoke-WebRequest -Uri "https://github.com/open-telemetry/opentelemetry-collector-releases/releases/download/v$v/otelcol-contrib_${v}_windows_amd64.tar.gz" -OutFile otelcol.tar.gz
tar -xzf otelcol.tar.gz
.\otelcol-contrib.exe --version
```

### Minimal configuration

Save as `otel-config.yaml`. The `debug` exporter prints every item to stdout, which is the fastest way to prove the pipeline works.

```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

processors:
  memory_limiter:
    check_interval: 1s
    limit_percentage: 75
    spike_limit_percentage: 15
  batch:
    send_batch_size: 8192
    timeout: 5s

exporters:
  debug:
    verbosity: detailed      # basic | normal | detailed
    sampling_initial: 5
    sampling_thereafter: 50

extensions:
  health_check:
    endpoint: 0.0.0.0:13133
  zpages:
    endpoint: 0.0.0.0:55679

service:
  extensions: [health_check, zpages]
  telemetry:
    logs:
      level: info
    metrics:
      level: normal
      readers:
        - pull:
            exporter:
              prometheus:
                host: 0.0.0.0
                port: 8888
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [debug]
    metrics:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [debug]
    logs:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [debug]
```

Validate and run:

```bash
otelcol-contrib validate --config otel-config.yaml && echo "config OK"
otelcol-contrib --config otel-config.yaml
```

Expected log lines:

```text
info    service@v0.115.0/service.go:166 Setting up own telemetry...
info    extensions/extensions.go:39     Starting extensions...
info    healthcheck/healthcheck.go:...  Starting health_check extension
info    otlpreceiver@v0.115.0/otlp.go:112  Starting GRPC server {"endpoint": "0.0.0.0:4317"}
info    otlpreceiver@v0.115.0/otlp.go:169  Starting HTTP server {"endpoint": "0.0.0.0:4318"}
info    service@v0.115.0/service.go:192 Everything is ready. Begin running and processing data.
```

### Send a test span with `curl`

```bash
curl -s -X POST http://localhost:4318/v1/traces \
  -H "Content-Type: application/json" \
  -d '{
    "resourceSpans": [{
      "resource": {"attributes": [{"key": "service.name", "value": {"stringValue": "curl-test"}}]},
      "scopeSpans": [{
        "scope": {"name": "manual"},
        "spans": [{
          "traceId": "5b8aa5a2d2c872e8321cf37308d69df2",
          "spanId": "051581bf3cb55c13",
          "name": "GET /hello",
          "kind": 2,
          "startTimeUnixNano": "1700000000000000000",
          "endTimeUnixNano":   "1700000000500000000",
          "attributes": [{"key": "http.request.method", "value": {"stringValue": "GET"}}],
          "status": {"code": 1}
        }]
      }]
    }]
  }'
```

Expected: HTTP 200 with body `{"partialSuccess":{}}` and the Collector prints:

```text
info    ResourceSpans #0
Resource SchemaURL:
Resource attributes:
     -> service.name: Str(curl-test)
ScopeSpans #0
Span #0
    Trace ID       : 5b8aa5a2d2c872e8321cf37308d69df2
    Parent ID      :
    ID             : 051581bf3cb55c13
    Name           : GET /hello
    Kind           : Server
    Start time     : 2023-11-14 22:13:20 +0000 UTC
    End time       : 2023-11-14 22:13:20.5 +0000 UTC
    Status code    : Ok
Attributes:
     -> http.request.method: Str(GET)
```

---

## 3. Collector: Docker

```bash
docker run --rm --name otelcol \
  -p 4317:4317 -p 4318:4318 -p 13133:13133 -p 55679:55679 -p 8888:8888 \
  -v "$(pwd)/otel-config.yaml:/etc/otelcol-contrib/config.yaml:ro" \
  otel/opentelemetry-collector-contrib:0.115.0
```

Verify health and zPages:

```bash
curl -s http://localhost:13133/ ; echo
# {"status":"Server available","upSince":"...","uptime":"..."}
curl -s http://localhost:55679/debug/pipelinez | head -n 20
curl -s http://localhost:8888/metrics | grep -E '^otelcol_(receiver_accepted|exporter_sent)_spans'
```

The image runs as a non-root user (UID 10001) and the contrib image is distroless. Mount configs read-only.

---

## 4. Python SDK for `orders-api` (FastAPI)

### Install

Use a virtual environment and pin in `requirements.txt` or `pyproject.toml`.

```bash
python -m venv .venv && source .venv/bin/activate
pip install --upgrade pip
pip install \
  "fastapi>=0.115" "uvicorn[standard]>=0.32" "httpx>=0.27" "sqlalchemy[asyncio]>=2.0" "asyncpg>=0.30" "boto3>=1.35" \
  "opentelemetry-distro>=0.49b0" \
  "opentelemetry-exporter-otlp>=1.28" \
  "opentelemetry-instrumentation-fastapi" \
  "opentelemetry-instrumentation-httpx" \
  "opentelemetry-instrumentation-sqlalchemy" \
  "opentelemetry-instrumentation-asyncpg" \
  "opentelemetry-instrumentation-botocore" \
  "opentelemetry-instrumentation-logging"
```

Alternatively, let the bootstrap tool detect installed libraries and install matching instrumentations:

```bash
opentelemetry-bootstrap --action=requirements   # prints what it would install
opentelemetry-bootstrap --action=install
```

### Zero-code run

```bash
export OTEL_SERVICE_NAME=orders-api
export OTEL_RESOURCE_ATTRIBUTES="service.namespace=capstone,deployment.environment=dev,service.version=0.1.0"
export OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317
export OTEL_EXPORTER_OTLP_PROTOCOL=grpc
export OTEL_TRACES_EXPORTER=otlp
export OTEL_METRICS_EXPORTER=otlp
export OTEL_LOGS_EXPORTER=otlp
export OTEL_PYTHON_LOGGING_AUTO_INSTRUMENTATION_ENABLED=true
export OTEL_PYTHON_LOG_CORRELATION=true

opentelemetry-instrument uvicorn app.main:app --host 0.0.0.0 --port 8000
```

A minimal `app/main.py`:

```python
from fastapi import FastAPI
import httpx
import logging

logger = logging.getLogger("orders")
app = FastAPI(title="orders-api")

@app.get("/orders/{order_id}")
async def get_order(order_id: str):
    logger.info("fetching order", extra={"order_id": order_id})
    async with httpx.AsyncClient(timeout=2.0) as client:
        resp = await client.get(f"http://localhost:8080/payments/{order_id}")
    return {"order_id": order_id, "payment": resp.json()}

@app.get("/healthz")
async def healthz():
    return {"status": "ok"}
```

Verify:

```bash
curl -s localhost:8000/healthz
```

The Collector's debug output should show a span named `GET /healthz` with `http.route: /healthz` and resource `service.name: orders-api`. The log record for `fetching order` arrives with `trace_id` and `span_id` populated because `OTEL_PYTHON_LOG_CORRELATION=true` installs a logging handler that injects them.

### Confirm which instrumentations loaded

```bash
opentelemetry-instrument --help | head -n 5
pip list 2>/dev/null | grep opentelemetry-instrumentation
```

---

## 5. Java Agent for `payments-api` (Spring Boot 3.3+)

### Download the agent

```bash
mkdir -p /opt/otel
curl -fsSL -o /opt/otel/opentelemetry-javaagent.jar \
  https://github.com/open-telemetry/opentelemetry-java-instrumentation/releases/latest/download/opentelemetry-javaagent.jar
java -jar /opt/otel/opentelemetry-javaagent.jar 2>&1 | head -n 1   # prints the version
```

### Run with the agent

Environment variables are identical to Python; the agent reads the same `OTEL_*` names. `JAVA_TOOL_OPTIONS` lets you attach without touching the start command, which is what the Kubernetes `Instrumentation` CR does.

```bash
export JAVA_TOOL_OPTIONS="-javaagent:/opt/otel/opentelemetry-javaagent.jar"
export OTEL_SERVICE_NAME=payments-api
export OTEL_RESOURCE_ATTRIBUTES="service.namespace=capstone,deployment.environment=dev,service.version=0.1.0"
export OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317
export OTEL_EXPORTER_OTLP_PROTOCOL=grpc
export OTEL_LOGS_EXPORTER=otlp
export OTEL_INSTRUMENTATION_MICROMETER_ENABLED=true
export OTEL_INSTRUMENTATION_LOGBACK_APPENDER_EXPERIMENTAL_LOG_ATTRIBUTES=true

./mvnw -q -DskipTests package
java -jar target/payments-api-0.1.0.jar
```

Expected first lines of stdout:

```text
Picked up JAVA_TOOL_OPTIONS: -javaagent:/opt/otel/opentelemetry-javaagent.jar
[otel.javaagent 2024-11-20 10:00:00:000 +0000] [main] INFO io.opentelemetry.javaagent.tooling.VersionLogger - opentelemetry-javaagent - version: 2.10.0
```

### Dockerfile pattern

```dockerfile
FROM eclipse-temurin:21-jre-alpine AS runtime
ARG OTEL_AGENT_VERSION=2.10.0
ADD --chmod=0444 https://github.com/open-telemetry/opentelemetry-java-instrumentation/releases/download/v${OTEL_AGENT_VERSION}/opentelemetry-javaagent.jar /otel/opentelemetry-javaagent.jar
RUN addgroup -S app && adduser -S app -G app
USER app
COPY --chown=app:app target/payments-api-*.jar /app/app.jar
ENV JAVA_TOOL_OPTIONS="-javaagent:/otel/opentelemetry-javaagent.jar"
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "/app/app.jar"]
```

### Spring Boot starter (agent-free alternative)

```xml
<!-- pom.xml -->
<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>io.opentelemetry.instrumentation</groupId>
      <artifactId>opentelemetry-instrumentation-bom</artifactId>
      <version>2.10.0</version>
      <type>pom</type>
      <scope>import</scope>
    </dependency>
  </dependencies>
</dependencyManagement>
<dependencies>
  <dependency>
    <groupId>io.opentelemetry.instrumentation</groupId>
    <artifactId>opentelemetry-spring-boot-starter</artifactId>
  </dependency>
</dependencies>
```

```properties
# application.properties
otel.service.name=payments-api
otel.exporter.otlp.endpoint=http://localhost:4317
otel.resource.attributes=service.namespace=capstone,deployment.environment=dev
otel.instrumentation.micrometer.enabled=true
```

Verify with a request to an endpoint and check the Collector output for a `SERVER` span whose `http.route` is the Spring mapping (for example `/payments/{id}`).

---

## 6. Kubernetes: OTel Operator and Collector CRs

The Operator needs cert-manager for its admission webhooks.

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts
helm repo update

helm upgrade --install cert-manager jetstack/cert-manager \
  --namespace cert-manager --create-namespace --set crds.enabled=true

kubectl create namespace observability --dry-run=client -o yaml | kubectl apply -f -

helm upgrade --install opentelemetry-operator open-telemetry/opentelemetry-operator \
  --namespace observability \
  --set "manager.collectorImage.repository=otel/opentelemetry-collector-contrib" \
  --set "manager.collectorImage.tag=0.115.0"

kubectl --namespace observability rollout status deployment/opentelemetry-operator
kubectl get crd | grep opentelemetry.io
```

Expected:

```text
instrumentations.opentelemetry.io              2024-11-20T10:00:00Z
opampbridges.opentelemetry.io                  2024-11-20T10:00:00Z
opentelemetrycollectors.opentelemetry.io       2024-11-20T10:00:00Z
targetallocators.opentelemetry.io              2024-11-20T10:00:00Z
```

A first gateway Collector:

```yaml
apiVersion: opentelemetry.io/v1beta1
kind: OpenTelemetryCollector
metadata:
  name: otel-collector
  namespace: observability
spec:
  mode: deployment
  replicas: 2
  image: otel/opentelemetry-collector-contrib:0.115.0
  resources:
    requests: { cpu: 200m, memory: 512Mi }
    limits:   { cpu: "1",  memory: 1Gi }
  securityContext:
    runAsNonRoot: true
    runAsUser: 10001
    allowPrivilegeEscalation: false
    readOnlyRootFilesystem: true
    capabilities: { drop: ["ALL"] }
  config:
    receivers:
      otlp:
        protocols:
          grpc: { endpoint: 0.0.0.0:4317 }
          http: { endpoint: 0.0.0.0:4318 }
    processors:
      memory_limiter: { check_interval: 1s, limit_percentage: 75, spike_limit_percentage: 15 }
      batch: {}
    exporters:
      debug: { verbosity: basic }
    service:
      pipelines:
        traces:  { receivers: [otlp], processors: [memory_limiter, batch], exporters: [debug] }
        metrics: { receivers: [otlp], processors: [memory_limiter, batch], exporters: [debug] }
        logs:    { receivers: [otlp], processors: [memory_limiter, batch], exporters: [debug] }
```

```bash
kubectl apply -f otel-collector-gateway.yaml
kubectl --namespace observability get opentelemetrycollector,deploy,svc -l app.kubernetes.io/name=otel-collector-collector
```

The Operator creates `Deployment/otel-collector-collector` and `Service/otel-collector-collector` (ports 4317, 4318). The DNS name used by every other folder is `otel-collector-collector.observability.svc.cluster.local`; to get the shorter `otel-collector.observability.svc.cluster.local` used in this wiki, add a Service of that name selecting the same pods, or name the CR accordingly and accept the `-collector` suffix. Labs use the explicit Service below for clarity:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: otel-collector
  namespace: observability
spec:
  selector:
    app.kubernetes.io/component: opentelemetry-collector
    app.kubernetes.io/instance: observability.otel-collector
  ports:
    - { name: otlp-grpc, port: 4317, targetPort: 4317 }
    - { name: otlp-http, port: 4318, targetPort: 4318 }
```

### Auto-instrumentation CR

```yaml
apiVersion: opentelemetry.io/v1alpha1
kind: Instrumentation
metadata:
  name: capstone
  namespace: observability
spec:
  exporter:
    endpoint: http://otel-collector.observability.svc.cluster.local:4317
  propagators: [tracecontext, baggage]
  sampler:
    type: parentbased_traceidratio
    argument: "0.25"
  resource:
    addK8sUIDAttributes: false
  env:
    - name: OTEL_RESOURCE_ATTRIBUTES
      value: service.namespace=capstone,deployment.environment=dev
  python:
    image: ghcr.io/open-telemetry/opentelemetry-operator/autoinstrumentation-python:0.49b0
    env:
      - { name: OTEL_EXPORTER_OTLP_PROTOCOL, value: http/protobuf }   # Python auto-instr image ships the HTTP exporter
      - { name: OTEL_EXPORTER_OTLP_ENDPOINT, value: http://otel-collector.observability.svc.cluster.local:4318 }
      - { name: OTEL_PYTHON_LOGGING_AUTO_INSTRUMENTATION_ENABLED, value: "true" }
  java:
    image: ghcr.io/open-telemetry/opentelemetry-operator/autoinstrumentation-java:2.10.0
    env:
      - { name: OTEL_INSTRUMENTATION_MICROMETER_ENABLED, value: "true" }
```

Pods opt in by annotation; the Operator's mutating webhook adds an init container that copies the agent into a shared volume and sets `JAVA_TOOL_OPTIONS` / `PYTHONPATH` plus the `OTEL_*` env:

```bash
kubectl --namespace capstone patch deployment orders-api --type merge -p \
  '{"spec":{"template":{"metadata":{"annotations":{"instrumentation.opentelemetry.io/inject-python":"observability/capstone"}}}}}'
kubectl --namespace capstone patch deployment payments-api --type merge -p \
  '{"spec":{"template":{"metadata":{"annotations":{"instrumentation.opentelemetry.io/inject-java":"observability/capstone"}}}}}'
kubectl --namespace capstone get pod -l app=orders-api -o jsonpath='{.items[0].spec.initContainers[*].name}'
```

Expected: `opentelemetry-auto-instrumentation-python`.

---

## 7. Verification Checklist

| Check | Command | Expect |
|---|---|---|
| Collector config valid | `otelcol-contrib validate --config otel-config.yaml` | exit 0 |
| Collector healthy | `curl -s localhost:13133/` | `"status":"Server available"` |
| Receiver accepting | `curl -s localhost:8888/metrics \| grep otelcol_receiver_accepted_spans` | counter > 0 after traffic |
| Exporter succeeding | `curl -s localhost:8888/metrics \| grep otelcol_exporter_send_failed` | 0 |
| Python SDK present | `python -c "import opentelemetry.sdk; print(opentelemetry.sdk.__file__)"` | path printed |
| Java agent attached | first stdout line `Picked up JAVA_TOOL_OPTIONS` | present |
| Operator webhooks | `kubectl get mutatingwebhookconfiguration \| grep opentelemetry` | present |
| Injection happened | `kubectl describe pod <pod> \| grep -A2 OTEL_` | env vars present |

---

## 8. What's Next

Proceed to [02-core-concepts.md](02-core-concepts.md) for the data model, context propagation and the Collector's processing model; the labs in [04-hands-on-labs.md](04-hands-on-labs.md) build on the pieces installed here.
