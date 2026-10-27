# SLIs, SLOs and Error Budgets: Hands-On Labs

| | |
|---|---|
| **Cluster** | 11 SRE |
| **Role** | Anchor |
| **Prerequisites** | kind cluster with kube-prometheus-stack in namespace `monitoring`; capstone FastAPI service in namespace `capstone` exposing `http_requests_total` and `http_request_duration_seconds_*`; `sloth` CLI; `kubectl`, `helm`, `jq`, `curl` |
| **Target version** | Prometheus 2.55+/3.x, Kubernetes 1.31+, Sloth 0.12+, Pyrra 0.7+ |
| **Capstone phase** | Phase 4 (Months 13-16) |

> Six progressive labs. Lab 1 writes a single recording rule by hand so you understand what the generators do. Labs 2-4 build the capstone SLO set with Sloth and Pyrra. Lab 5 adds the vLLM and agent SLIs. Lab 6 builds a composite SLO and runs a monthly review against real data you generate with a load tool. Every lab ends with verification and cleanup. Total time about 6 hours.

Common setup used by every lab:

```bash
export KUBECONFIG=~/.kube/kind-capstone
kubectl config current-context           # kind-capstone
kubectl --namespace monitoring port-forward svc/kube-prometheus-stack-prometheus 9090:9090 >/dev/null 2>&1 &
kubectl --namespace monitoring port-forward svc/kube-prometheus-stack-grafana 3000:80 >/dev/null 2>&1 &
export PROM=http://localhost:9090
```

Helper to run an instant query:

```bash
pq() { curl --silent --get "$PROM/api/v1/query" --data-urlencode "query=$1" | jq -r '.data.result[] | "\(.metric) \(.value[1])"'; }
```

If your FastAPI service is not yet instrumented, deploy the lab stand-in which emits the same metric names and accepts a chaos flag to inject errors and latency:

```yaml
# labs/slo-target/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fastapi
  namespace: capstone
  labels: {app: fastapi}
spec:
  replicas: 2
  selector: {matchLabels: {app: fastapi}}
  template:
    metadata:
      labels: {app: fastapi}
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8000"
    spec:
      securityContext: {runAsNonRoot: true, runAsUser: 10001, seccompProfile: {type: RuntimeDefault}}
      containers:
        - name: api
          image: ghcr.io/example/fastapi-slo-lab:v1.0.0   # any image exposing /metrics with the capstone metric names
          ports: [{containerPort: 8000, name: http}]
          env:
            - {name: CHAOS_ERROR_RATIO, value: "0"}      # fraction of requests to fail with 500
            - {name: CHAOS_LATENCY_MS, value: "0"}       # extra latency added to every request
          resources: {requests: {cpu: 50m, memory: 64Mi}, limits: {cpu: 500m, memory: 256Mi}}
          securityContext: {allowPrivilegeEscalation: false, readOnlyRootFilesystem: true, capabilities: {drop: [ALL]}}
---
apiVersion: v1
kind: Service
metadata:
  name: fastapi
  namespace: capstone
  labels: {app: fastapi}
spec:
  selector: {app: fastapi}
  ports: [{port: 80, targetPort: http, name: http}]
---
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: fastapi
  namespace: monitoring
  labels: {release: kube-prometheus-stack}
spec:
  namespaceSelector: {matchNames: [capstone]}
  selector: {matchLabels: {app: fastapi}}
  endpoints:
    - port: http
      path: /metrics
      interval: 15s
      relabelings:
        - {targetLabel: service, replacement: fastapi}
```

Load generator (runs in-cluster so it does not depend on your laptop's network):

```yaml
# labs/slo-target/load.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: load
  namespace: capstone
spec:
  replicas: 1
  selector: {matchLabels: {app: load}}
  template:
    metadata: {labels: {app: load}}
    spec:
      securityContext: {runAsNonRoot: true, runAsUser: 10001}
      containers:
        - name: hey
          image: williamyeh/hey:latest
          args: ["-z", "100h", "-q", "20", "-c", "2", "http://fastapi.capstone.svc/items/1"]
          resources: {requests: {cpu: 20m, memory: 32Mi}, limits: {cpu: 200m, memory: 64Mi}}
```

```bash
kubectl apply --filename labs/slo-target/deployment.yaml --filename labs/slo-target/load.yaml
kubectl --namespace capstone rollout status deployment/fastapi
pq 'sum(rate(http_requests_total{service="fastapi"}[1m]))'
```

Expected: a value near `40` (2 connections x 20 qps).

---

## Lab 1: Your First SLI Recording Rule by Hand

**Objective.** Write, load, and verify one availability recording rule and one latency recording rule for the FastAPI service without any generator, so that the generated output in later labs is not magic.

**Prerequisites.** Common setup; `http_requests_total{service="fastapi"}` returns data.

**Time.** 30 minutes.

### Steps

1. Query the raw SLI interactively and confirm it is close to 1.

   ```bash
   pq 'sum(rate(http_requests_total{service="fastapi",code!~"5.."}[5m])) / sum(rate(http_requests_total{service="fastapi"}[5m]))'
   ```

   Expected output:

   ```text
   {} 1
   ```

2. Confirm the 300 ms bucket exists.

   ```bash
   pq 'sum(rate(http_request_duration_seconds_bucket{service="fastapi",le="0.3"}[5m])) / sum(rate(http_request_duration_seconds_count{service="fastapi"}[5m]))'
   ```

   Expected: `{} 1` or very close. If you get an empty result, your histogram lacks the `0.3` boundary; see [06-troubleshooting.md](06-troubleshooting.md#histogram-bucket-boundary-does-not-match-the-slo).

3. Create the rule file.

   ```yaml
   # labs/lab1-rules.yaml
   apiVersion: monitoring.coreos.com/v1
   kind: PrometheusRule
   metadata:
     name: lab1-fastapi-sli
     namespace: monitoring
     labels: {release: kube-prometheus-stack}
   spec:
     groups:
       - name: lab1-fastapi-sli
         interval: 30s
         rules:
           - record: slo:sli_error:ratio_rate5m
             labels: {slo: fastapi-availability, service: fastapi, team: platform}
             expr: |
               sum(rate(http_requests_total{service="fastapi",code=~"5.."}[5m]))
               /
               sum(rate(http_requests_total{service="fastapi"}[5m]))
           - record: slo:sli_error:ratio_rate5m
             labels: {slo: fastapi-latency, service: fastapi, team: platform}
             expr: |
               1 - (
                 sum(rate(http_request_duration_seconds_bucket{service="fastapi",le="0.3"}[5m]))
                 /
                 sum(rate(http_request_duration_seconds_count{service="fastapi"}[5m]))
               )
   ```

4. Validate locally before applying. `promtool` wants a plain rule file, so extract `spec`:

   ```bash
   kubectl create --dry-run=client --filename labs/lab1-rules.yaml -o json | jq '.spec' > /tmp/lab1-plain.json
   docker run --rm -v /tmp:/tmp --entrypoint promtool prom/prometheus:v3.0.0 check rules /tmp/lab1-plain.json
   ```

   Expected output:

   ```text
   Checking /tmp/lab1-plain.json
     SUCCESS: 2 rules found
   ```

5. Apply and wait one evaluation interval.

   ```bash
   kubectl apply --filename labs/lab1-rules.yaml
   sleep 45
   pq 'slo:sli_error:ratio_rate5m'
   ```

   Expected output:

   ```text
   {"__name__":"slo:sli_error:ratio_rate5m","service":"fastapi","slo":"fastapi-availability","team":"platform"} 0
   {"__name__":"slo:sli_error:ratio_rate5m","service":"fastapi","slo":"fastapi-latency","team":"platform"} 0
   ```

6. Inject 2% errors and watch the recorded value follow.

   ```bash
   kubectl --namespace capstone set env deployment/fastapi CHAOS_ERROR_RATIO=0.02
   kubectl --namespace capstone rollout status deployment/fastapi
   sleep 300
   pq 'slo:sli_error:ratio_rate5m{slo="fastapi-availability"}'
   ```

   Expected: a value around `0.02`. Burn rate at 99.9% is therefore 0.02 / 0.001 = 20x, which would page.

### Verify

- Both series exist with the `slo` label.
- The availability error ratio tracks the injected ratio within a few minutes.

### Cleanup

```bash
kubectl --namespace capstone set env deployment/fastapi CHAOS_ERROR_RATIO=0
kubectl delete prometheusrule --namespace monitoring lab1-fastapi-sli
```

### What you learned

A recording rule is just a saved PromQL expression with a new metric name and fixed labels. The `slo` label is the key every dashboard and alert will group on. The error-ratio form (not the good-ratio form) is stored because burn rate is computed from it directly.

---

## Lab 2: Generate the Full Multi-Window Rule Set with Sloth

**Objective.** Replace the hand-written rules with a Sloth spec for the FastAPI availability and latency SLOs, generate the rules in a reproducible way, and load them.

**Prerequisites.** Lab 1 cleaned up; `sloth` installed.

**Time.** 40 minutes.

### Steps

1. Create the spec.

   ```yaml
   # slo/fastapi.yaml
   version: "prometheus/v1"
   service: "fastapi"
   labels:
     team: "platform"
   slos:
     - name: "availability"
       objective: 99.9
       description: "Non-5xx ratio for FastAPI at the ingress."
       sli:
         events:
           error_query: sum(rate(http_requests_total{service="fastapi",code=~"5.."}[{{.window}}]))
           total_query: sum(rate(http_requests_total{service="fastapi"}[{{.window}}]))
       alerting:
         name: FastAPIHighErrorBudgetBurn
         labels: {service: fastapi}
         annotations:
           summary: "FastAPI availability error budget burning too fast"
           runbook_url: https://runbooks.example.com/fastapi-5xx-burn-rate
         page_alert:
           labels: {severity: page}
         ticket_alert:
           labels: {severity: ticket}
     - name: "latency"
       objective: 99
       description: "FastAPI requests under 300 ms."
       sli:
         events:
           error_query: |
             sum(rate(http_request_duration_seconds_count{service="fastapi"}[{{.window}}]))
             -
             sum(rate(http_request_duration_seconds_bucket{service="fastapi",le="0.3"}[{{.window}}]))
           total_query: sum(rate(http_request_duration_seconds_count{service="fastapi"}[{{.window}}]))
       alerting:
         name: FastAPIHighLatencyBudgetBurn
         labels: {service: fastapi}
         annotations:
           runbook_url: https://runbooks.example.com/fastapi-5xx-burn-rate
         page_alert:
           labels: {severity: page}
         ticket_alert:
           labels: {severity: ticket}
   ```

2. Validate and generate.

   ```bash
   sloth validate --input slo/fastapi.yaml
   sloth generate --input slo/fastapi.yaml --output monitoring/prometheus-rules/fastapi-slo.rules.yaml
   grep -c 'record:' monitoring/prometheus-rules/fastapi-slo.rules.yaml
   grep -c 'alert:' monitoring/prometheus-rules/fastapi-slo.rules.yaml
   ```

   Expected output (counts): `28` record rules (7 windows + 7 metadata per SLO x 2) and `4` alerts (page + ticket x 2).

3. Wrap the output into a `PrometheusRule`. The generated file has a top-level `groups:`; use `yq` to embed it.

   ```bash
   cat > /tmp/header.yaml <<'EOF'
   apiVersion: monitoring.coreos.com/v1
   kind: PrometheusRule
   metadata:
     name: fastapi-slo
     namespace: monitoring
     labels:
       release: kube-prometheus-stack
   spec: {}
   EOF
   yq eval-all 'select(fileIndex==0).spec = select(fileIndex==1) | select(fileIndex==0)' \
     /tmp/header.yaml monitoring/prometheus-rules/fastapi-slo.rules.yaml \
     > monitoring/prometheus-rules/fastapi-slo.prometheusrule.yaml
   kubectl apply --filename monitoring/prometheus-rules/fastapi-slo.prometheusrule.yaml
   ```

   (A shell heredoc is used here only to create a tiny local scratch file; the real header lives in Git.)

4. Confirm the rule groups loaded and inspect the metadata series.

   ```bash
   sleep 60
   pq 'slo:objective:ratio{sloth_service="fastapi"}'
   pq 'slo:period_error_budget_remaining:ratio{sloth_service="fastapi"}'
   ```

   Expected output:

   ```text
   {...,"sloth_slo":"availability",...} 0.999
   {...,"sloth_slo":"latency",...} 0.99
   {...,"sloth_slo":"availability",...} 1
   {...,"sloth_slo":"latency",...} 1
   ```

5. Read the generated alert rules and note the thresholds. For the 99.9% SLO:

   ```bash
   yq '.groups[] | select(.name | test("alerts")) | .rules[] | {alert: .alert, expr: .expr, for: .for, labels: .labels}' \
     monitoring/prometheus-rules/fastapi-slo.rules.yaml
   ```

   You should see `(slo:sli_error:ratio_rate5m{...} > (14.4 * 0.001)) and (slo:sli_error:ratio_rate1h{...} > (14.4 * 0.001))` ORed with the 6x/30m+6h pair for the page alert, and 3x/2h+1d ORed with 1x/6h+3d for the ticket alert. These are explained in [../burn-rate-alerting/02-core-concepts.md](../burn-rate-alerting/02-core-concepts.md).

### Verify

```bash
curl --silent "$PROM/api/v1/rules?type=alert" | jq -r '.data.groups[].rules[] | select(.name|test("FastAPI")) | "\(.name) \(.state)"'
```

Expected: four alerts, all `inactive`.

### Cleanup

Keep the rules; later labs use them. Delete the Lab 1 hand-written rule if you have not already.

### What you learned

Sloth turns 60 lines of spec into about 200 lines of rules that are exactly the Workbook recipe. Committing both means reviewers see the spec diff (what changed) and the rules diff (what Prometheus will evaluate).

---

## Lab 3: The Spring Boot SLO and Micrometer Histogram Buckets

**Objective.** Add the Spring Boot service's availability and latency SLOs, which requires fixing Micrometer's histogram configuration so a 300 ms boundary exists.

**Prerequisites.** Spring Boot capstone service running in `capstone` with Actuator and the Prometheus registry, scraped with `service="springboot"`.

**Time.** 45 minutes.

### Steps

1. Check the current buckets.

   ```bash
   curl --silent --get "$PROM/api/v1/series" --data-urlencode 'match[]=http_server_requests_seconds_bucket{service="springboot"}' | jq -r '.data[].le' | sort -n | uniq | head
   ```

   By default Micrometer publishes no histogram buckets for `http.server.requests` (only `_count`, `_sum`, `_max`). Expected: empty.

2. Enable buckets with an explicit SLO boundary. In `src/main/resources/application.yaml`:

   ```yaml
   management:
     endpoints:
       web:
         exposure:
           include: health,info,prometheus
     metrics:
       distribution:
         percentiles-histogram:
           http.server.requests: false
         slo:
           http.server.requests: 100ms,300ms,1s
       tags:
         service: springboot
   ```

   `percentiles-histogram: false` with explicit `slo` boundaries emits only the buckets you need (plus `+Inf`), keeping cardinality low. Rebuild and redeploy the image (see [Spring Boot](../../07-backend/spring-boot/README.md) for the pipeline).

3. Re-check buckets.

   ```bash
   curl --silent --get "$PROM/api/v1/series" --data-urlencode 'match[]=http_server_requests_seconds_bucket{service="springboot"}' | jq -r '.data[].le' | sort -n | uniq
   ```

   Expected output:

   ```text
   0.1
   0.3
   1.0
   +Inf
   ```

4. Add the spec.

   ```yaml
   # slo/springboot.yaml
   version: "prometheus/v1"
   service: "springboot"
   labels: {team: "platform"}
   slos:
     - name: "availability"
       objective: 99.9
       sli:
         events:
           error_query: sum(rate(http_server_requests_seconds_count{service="springboot",status=~"5..",uri!~"/actuator.*"}[{{.window}}]))
           total_query: sum(rate(http_server_requests_seconds_count{service="springboot",uri!~"/actuator.*"}[{{.window}}]))
       alerting:
         name: SpringBootHighErrorBudgetBurn
         labels: {service: springboot}
         annotations: {runbook_url: https://runbooks.example.com/fastapi-5xx-burn-rate}
         page_alert: {labels: {severity: page}}
         ticket_alert: {labels: {severity: ticket}}
     - name: "latency"
       objective: 99
       sli:
         events:
           error_query: |
             sum(rate(http_server_requests_seconds_count{service="springboot",uri!~"/actuator.*"}[{{.window}}]))
             -
             sum(rate(http_server_requests_seconds_bucket{service="springboot",le="0.3",uri!~"/actuator.*"}[{{.window}}]))
           total_query: sum(rate(http_server_requests_seconds_count{service="springboot",uri!~"/actuator.*"}[{{.window}}]))
       alerting:
         name: SpringBootHighLatencyBudgetBurn
         labels: {service: springboot}
         annotations: {runbook_url: https://runbooks.example.com/fastapi-5xx-burn-rate}
         page_alert: {labels: {severity: page}}
         ticket_alert: {labels: {severity: ticket}}
   ```

5. Generate, wrap, apply exactly as in Lab 2 with name `springboot-slo`.

### Verify

```bash
pq 'slo:sli_error:ratio_rate5m{sloth_service="springboot"}'
```

Expected: two series near `0`.

### Cleanup

None; keep the rules.

### What you learned

The SLI is only as good as the instrumentation. Micrometer's `slo` distribution setting exists precisely to emit a boundary at your SLO threshold without the cardinality of full percentile histograms. Excluding `/actuator.*` from valid events keeps probes from inflating the denominator.

---

## Lab 4: Pyrra UI and Latency SLI Native Support

**Objective.** Install Pyrra, express the FastAPI latency SLO as a `ServiceLevelObjective`, and compare its generated rules and UI with Sloth's.

**Prerequisites.** Lab 2.

**Time.** 40 minutes.

### Steps

1. Install Pyrra.

   ```bash
   kubectl apply --server-side --filename https://github.com/pyrra-dev/pyrra/releases/download/v0.7.0/setup.yaml
   kubectl --namespace monitoring wait --for=condition=available deployment/pyrra-api --timeout=120s
   ```

   Pyrra's default manifest expects a Prometheus at `http://prometheus-k8s.monitoring.svc.cluster.local:9090`. Patch it to the kube-prometheus-stack service:

   ```bash
   kubectl --namespace monitoring set env deployment/pyrra-api PROMETHEUS_URL=http://kube-prometheus-stack-prometheus.monitoring.svc:9090
   kubectl --namespace monitoring patch deployment pyrra-api --type=json \
     -p='[{"op":"replace","path":"/spec/template/spec/containers/0/args","value":["api","--prometheus-url=http://kube-prometheus-stack-prometheus.monitoring.svc:9090","--api-url=http://pyrra-kubernetes.monitoring.svc:9444"]}]'
   ```

2. Create the SLO.

   ```yaml
   # slo/pyrra/fastapi-latency.yaml
   apiVersion: pyrra.dev/v1alpha1
   kind: ServiceLevelObjective
   metadata:
     name: fastapi-latency-pyrra
     namespace: monitoring
     labels:
       prometheus: kube-prometheus-stack
       role: alert-rules
       release: kube-prometheus-stack
       team: platform
   spec:
     target: "99"
     window: 4w
     description: 99% of FastAPI requests under 300 ms (Pyrra).
     indicator:
       latency:
         success:
           metric: http_request_duration_seconds_bucket{service="fastapi",le="0.3"}
         total:
           metric: http_request_duration_seconds_count{service="fastapi"}
   ```

   ```bash
   kubectl apply --filename slo/pyrra/fastapi-latency.yaml
   kubectl --namespace monitoring get prometheusrule fastapi-latency-pyrra -o yaml | yq '.spec.groups[].rules[] | .record // .alert'
   ```

   Expected: recording rules named like `http_request_duration_seconds:burnrate5m` ... `burnrate3d` plus `ErrorBudgetBurn` alerts and a `pyrra_objective`/`pyrra_window` pair.

3. Open the UI.

   ```bash
   kubectl --namespace monitoring port-forward svc/pyrra-api 9099:9099 >/dev/null 2>&1 &
   ```

   Visit `http://localhost:9099`. You should see the SLO with availability, error budget remaining, and burn-rate graphs.

4. Inject 500 ms latency on 5% of requests and watch the budget graph fall.

   ```bash
   kubectl --namespace capstone set env deployment/fastapi CHAOS_LATENCY_MS=500 CHAOS_LATENCY_RATIO=0.05
   ```

   Within 5 minutes the Pyrra graph and `slo:sli_error:ratio_rate5m{sloth_slo="latency"}` should show about `0.05`, a 5x burn rate against the 1% budget.

### Verify

- Pyrra UI lists `fastapi-latency-pyrra` with a non-zero burn.
- The Sloth-generated latency rule and the Pyrra rule agree to within rounding.

### Cleanup

```bash
kubectl --namespace capstone set env deployment/fastapi CHAOS_LATENCY_MS=0 CHAOS_LATENCY_RATIO=0
kubectl delete servicelevelobjective --namespace monitoring fastapi-latency-pyrra   # keep Pyrra installed
```

### What you learned

Two generators, same Workbook recipe, different ergonomics. Pyrra's `latency` indicator removes the `count - bucket` arithmetic; Sloth's CLI model fits CI review. Pick one as the source of truth per cluster; running both in production produces duplicate alerts.

---

## Lab 5: AI SLIs for vLLM and the Agent

**Objective.** Define TTFT and grounded-rate SLOs with Pyrra for the capstone's AI components, and understand low-traffic effects.

**Prerequisites.** vLLM running in namespace `ai` (see [vLLM](../../10-ai-infra-rag-mcp/vllm/README.md)) or the lab stand-in that emits `vllm:*` metrics; the agent emitting `agent_answers_total{grounded}`.

**Time.** 45 minutes.

### Steps

1. Confirm the TTFT histogram has a 0.8 boundary. vLLM's default TTFT buckets include `0.75` and `1.0`, not `0.8`.

   ```bash
   curl --silent --get "$PROM/api/v1/series" --data-urlencode 'match[]=vllm:time_to_first_token_seconds_bucket' | jq -r '.data[].le' | sort -n | uniq | tr '\n' ' '
   ```

   If `0.8` is absent you have two choices: set the SLO threshold to a real boundary (0.75 s, slightly stricter) or front vLLM with the OTel Collector and re-bucket. The honest choice is to adopt 0.75 s and document it; interpolating with `histogram_quantile` would make the SLI an estimate.

2. Create the TTFT SLO.

   ```yaml
   # slo/pyrra/vllm-ttft.yaml
   apiVersion: pyrra.dev/v1alpha1
   kind: ServiceLevelObjective
   metadata:
     name: vllm-ttft
     namespace: monitoring
     labels: {prometheus: kube-prometheus-stack, role: alert-rules, release: kube-prometheus-stack, team: ai}
   spec:
     target: "95"
     window: 4w
     description: 95% of completions produce the first token within 0.75 s (SLO document says 800 ms; nearest bucket boundary used).
     indicator:
       latency:
         success:
           metric: vllm:time_to_first_token_seconds_bucket{le="0.75"}
         total:
           metric: vllm:time_to_first_token_seconds_count
   ```

3. Create the grounded-rate SLO.

   ```yaml
   # slo/pyrra/agent-grounded.yaml
   apiVersion: pyrra.dev/v1alpha1
   kind: ServiceLevelObjective
   metadata:
     name: agent-grounded
     namespace: monitoring
     labels: {prometheus: kube-prometheus-stack, role: alert-rules, release: kube-prometheus-stack, team: ai}
   spec:
     target: "98"
     window: 4w
     description: 98% of sampled agent answers judged grounded in retrieved context.
     indicator:
       ratio:
         errors:
           metric: agent_answers_total{grounded="false"}
         total:
           metric: agent_answers_total
   ```

   ```bash
   kubectl apply --filename slo/pyrra/vllm-ttft.yaml --filename slo/pyrra/agent-grounded.yaml
   ```

4. Observe the low-traffic problem. The agent might answer 2 questions per minute. One ungrounded answer in a 5-minute window is 1/10 = 10% error ratio, a 5x burn against a 2% budget.

   ```bash
   pq 'agent_answers_total:burnrate5m'
   ```

   Watch it jump between 0 and large values. This is why the grounded SLO should alert only on the longer windows (6h/3d) and why Lab 6's review discusses extending the window to 90 days.

### Verify

- Pyrra lists `vllm-ttft` and `agent-grounded`.
- `vllm:time_to_first_token_seconds:burnrate1h` exists.

### Cleanup

Keep both SLOs.

### What you learned

AI SLIs are ordinary good/valid ratios with two twists: the thresholds must align with real histogram boundaries, and low request volume makes short windows meaningless, so the alerting and review cadence must adapt.

---

## Lab 6: Composite SLO, Budget Burn-Down Dashboard and a Monthly Review

**Objective.** Build the "ask the agent" journey composite SLO, a Grafana budget burn-down panel, spend budget deliberately with a 20-minute injected incident, and run a review using the SLO document template.

**Prerequisites.** Labs 2-5.

**Time.** 60 minutes.

### Steps

1. Composite SLI: a request to the journey is good only if the FastAPI call succeeded and the vLLM TTFT was under threshold. The honest way is end-to-end instrumentation; the approximation is the product of the component SLIs. Record both:

   ```yaml
   # monitoring/prometheus-rules/journey-ask-agent.yaml
   apiVersion: monitoring.coreos.com/v1
   kind: PrometheusRule
   metadata:
     name: journey-ask-agent
     namespace: monitoring
     labels: {release: kube-prometheus-stack}
   spec:
     groups:
       - name: journey-ask-agent
         interval: 30s
         rules:
           - record: slo:sli_good:ratio_rate5m
             labels: {slo: journey-ask-agent, team: platform}
             expr: |
               (1 - max(slo:sli_error:ratio_rate5m{sloth_service="fastapi",sloth_slo="availability"}))
               *
               (1 - max(vllm:time_to_first_token_seconds:burnrate5m) * (1 - 0.95))
           - record: slo:sli_error:ratio_rate5m
             labels: {slo: journey-ask-agent, team: platform}
             expr: 1 - slo:sli_good:ratio_rate5m{slo="journey-ask-agent"}
   ```

   (Pyrra's `burnrate5m` is error ratio divided by budget, so multiply back by the budget to recover the error ratio.)

2. Grafana burn-down panel. Create a dashboard `SLO / FastAPI` with a time-series panel:

   ```promql
   43.2 * slo:period_error_budget_remaining:ratio{sloth_service="fastapi",sloth_slo="availability"}
   ```

   Unit: minutes. Add a constant threshold at 0. Add an annotation query `ALERTS{alertname=~"FastAPI.*",alertstate="firing"}`.

3. Spend budget deliberately. Announce in the team channel (practising the comms habit), then inject a 20-minute 50% error rate. At 40 rps this is 40 x 0.5 x 1200 = 24,000 failed requests out of 48,000, a burn rate of 500x over the window.

   ```bash
   kubectl --namespace capstone set env deployment/fastapi CHAOS_ERROR_RATIO=0.5
   sleep 1200
   kubectl --namespace capstone set env deployment/fastapi CHAOS_ERROR_RATIO=0
   ```

   The page alert fires within a few minutes (Lab 2 rules). Note the time it fired and the time it resolved; you will reuse these numbers in [../burn-rate-alerting/04-hands-on-labs.md](../burn-rate-alerting/04-hands-on-labs.md).

4. Compute the budget consumed. Over the full 30-day window, total requests at 40 rps are about 103.7 million, budget 103,680. The incident cost 24,000, about 23% of the month.

   ```bash
   pq 'slo:period_error_budget_remaining:ratio{sloth_service="fastapi",sloth_slo="availability"}'
   ```

   Expected: approximately `0.77` (lower if your 30-day history is shorter, because the window has fewer total requests).

5. Run the review. Copy the SLO document template from [03-reference-and-cheatsheet.md](03-reference-and-cheatsheet.md#slo-document-template) into `docs/slo/fastapi.md` and fill in: achieved SLI, budget remaining, incidents that consumed budget, whether the alert fired appropriately, and one proposed change (for example, exclude the `/items/{id}` 404s if they are being counted as 5xx by a misconfigured handler).

### Verify

- Dashboard shows a step down in remaining minutes at the injection time with the alert annotation.
- `docs/slo/fastapi.md` exists and has a history row.

### Cleanup

```bash
kubectl delete prometheusrule --namespace monitoring journey-ask-agent
kubectl --namespace capstone scale deployment/load --replicas=0
```

### What you learned

A composite SLO exposes how dependency reliability compounds. Spending budget on purpose is normal and healthy; the review is where the SLO becomes a management tool rather than a dashboard.

---

## Where to Go Next

The alert rules Sloth generated in Lab 2 are the subject of [../burn-rate-alerting/](../burn-rate-alerting/README.md). The page that fired in Lab 6 needs a runbook: [../postmortems-and-runbooks/](../postmortems-and-runbooks/README.md).
