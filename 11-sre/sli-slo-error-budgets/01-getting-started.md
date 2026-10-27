# SLIs, SLOs and Error Budgets: Getting Started

| | |
|---|---|
| **Cluster** | 11 SRE |
| **Role** | Anchor |
| **Prerequisites** | A kind cluster with kube-prometheus-stack (see [Prometheus](../../09-observability/prometheus/README.md)); the capstone FastAPI service deployed and exposing metrics |
| **Target version** | Prometheus 2.55+/3.x, Kubernetes 1.31+, Sloth 0.12+, Pyrra 0.7+ |
| **Capstone phase** | Phase 4 (Months 13-16) |

> This file has two halves. The first is organisational: how to introduce SLOs to a team that has never had them, in four meetings. The second is technical: installing Sloth and Pyrra on kind, verifying Prometheus has the metrics you need, and generating your first recording rules. By the end you will have one SLO for the FastAPI service showing on a dashboard.

---

## Part A: Adopting SLOs in a Team

### The four-meeting rollout

SLOs fail when they are imposed by a platform team on product teams who did not choose them. They succeed when the team that owns the service writes the first draft and a facilitator keeps them honest. Budget four meetings of one hour each over two weeks.

| Meeting | Attendees | Output |
|---|---|---|
| 1. User journeys | Product owner, tech lead, one SRE/facilitator | List of the 3-5 things users do with the service, ranked by business impact ("submit order", "search catalog", "ask the agent a question") |
| 2. SLI candidates | Tech lead, two engineers, facilitator | For each journey: one availability SLI and one latency SLI, written as good/valid with a named measurement point |
| 3. Targets and window | Product owner, tech lead, facilitator | A target per SLI justified by current performance (measure 2-4 weeks first) and by user expectation; a 30-day rolling window |
| 4. Error budget policy | Product owner, engineering manager, tech lead | A one-page policy: what happens at 50%, 25%, 0% budget remaining; who can override |

Rules of thumb for meeting 2:

- Write the SLI **specification** first in plain English ("the proportion of HTTP requests that return a non-5xx status in under 300 ms, measured at the ingress"). Only then write the **implementation** (the PromQL).
- If you cannot say which user is unhappy when the SLI drops, it is not an SLI; it is a cause-based signal and belongs on a dashboard, not in an SLO.
- Start with **one availability SLI and one latency SLI per journey**. Resist more.

### Measure before you target

Deploy the recording rules for 2-4 weeks with no SLO attached and look at the achieved ratio. A service that currently achieves 99.7% should not launch a 99.99% SLO; launch 99.5% so the team can meet it, then ratchet up each quarter. An SLO you miss every month teaches everyone to ignore SLOs.

### Who owns what

| Artefact | Owner | Reviewer |
|---|---|---|
| SLI implementation (PromQL) | Service team | SRE / platform |
| SLO target and window | Product owner + service team | Engineering manager |
| Error budget policy | Engineering manager | Product leadership |
| Recording and alert rule generation | Platform team (tooling) | Service team (content) |
| Monthly SLO review | Service team | SRE |

---

## Part B: Tooling on kind

### 1. Verify Prometheus and the capstone metrics

The capstone FastAPI service is instrumented with OpenTelemetry and exports to Prometheus through the collector (see [OpenTelemetry](../../09-observability/opentelemetry/README.md)). Confirm the two metric families the SLIs depend on exist.

```bash
kubectl --namespace monitoring port-forward svc/kube-prometheus-stack-prometheus 9090:9090 &
sleep 2
curl --silent 'http://localhost:9090/api/v1/label/__name__/values' | jq -r '.data[]' | grep -E '^http_request(s_total|_duration_seconds_(bucket|count|sum))$'
```

Expected output:

```text
http_request_duration_seconds_bucket
http_request_duration_seconds_count
http_request_duration_seconds_sum
http_requests_total
```

Check the labels you will filter on and that the 300 ms bucket boundary exists (you cannot compute "under 300 ms" from a histogram that has no `le="0.3"` bucket).

```bash
curl --silent --get 'http://localhost:9090/api/v1/series' \
  --data-urlencode 'match[]=http_request_duration_seconds_bucket{service="fastapi"}' \
  | jq -r '.data[].le' | sort -n | uniq
```

Expected output (the default OTel/Prometheus HTTP server duration buckets include 0.3; if yours do not, fix the instrumentation before continuing):

```text
0.005
0.01
0.025
0.05
0.1
0.25
0.3
0.5
1
2.5
5
10
+Inf
```

If `0.3` is missing, configure explicit bucket boundaries in the SDK or in the collector (see [06-troubleshooting.md](06-troubleshooting.md#histogram-bucket-boundary-does-not-match-the-slo)).

### 2. Install Sloth (CLI mode for CI, operator optional)

Sloth is a generator: it reads a short spec and emits a `PrometheusRule`. Use it as a CLI in CI so the generated rules are reviewed in pull requests.

```bash
# Linux / macOS (pick the release asset for your platform)
SLOTH_VERSION=v0.12.0
curl --location --output sloth \
  "https://github.com/slok/sloth/releases/download/${SLOTH_VERSION}/sloth-linux-amd64"
chmod +x sloth && sudo mv sloth /usr/local/bin/
sloth version
```

Windows (PowerShell):

```powershell
$v = "v0.12.0"
Invoke-WebRequest "https://github.com/slok/sloth/releases/download/$v/sloth-windows-amd64.exe" -OutFile sloth.exe
.\sloth.exe version
```

Container (for CI):

```bash
docker run --rm -v "$PWD":/work ghcr.io/slok/sloth:v0.12.0 generate -i /work/slo/fastapi.yaml -o /work/monitoring/prometheus-rules/fastapi-slo.yaml
```

### 3. Write the first SLO spec

```yaml
# capstone/slo/fastapi.yaml
version: "prometheus/v1"
service: "fastapi"
labels:
  team: "platform"
  tier: "1"
slos:
  - name: "availability"
    objective: 99.9
    description: "Proportion of HTTP requests served by the FastAPI service that did not return 5xx, measured at the ingress."
    sli:
      events:
        error_query: sum(rate(http_requests_total{service="fastapi",code=~"5.."}[{{.window}}]))
        total_query: sum(rate(http_requests_total{service="fastapi"}[{{.window}}]))
    alerting:
      name: FastAPIHighErrorBudgetBurn
      labels:
        service: fastapi
      annotations:
        runbook_url: https://runbooks.example.com/fastapi-5xx-burn-rate
      page_alert:
        labels:
          severity: page
      ticket_alert:
        labels:
          severity: ticket
```

Generate the rules:

```bash
sloth generate --input slo/fastapi.yaml --output monitoring/prometheus-rules/fastapi-slo.yaml
```

Expected output:

```text
INFO[0000] Generating from Prometheus spec                 version=v0.12.0
INFO[0000] Multiwindow-multiburn alerts generated           out=monitoring/prometheus-rules/fastapi-slo.yaml slo=fastapi-availability
INFO[0000] Prometheus rules written                          out=monitoring/prometheus-rules/fastapi-slo.yaml
```

Inspect the output: you will see recording rules at 5m, 30m, 1h, 2h, 6h, 1d and 3d, metadata rules (`slo:objective:ratio`, `slo:error_budget:ratio`, `slo:time_period:days`, `slo:current_burn_rate:ratio`, `slo:period_burn_rate:ratio`, `slo:period_error_budget_remaining:ratio`), and two alert rules. The CLI output is a plain Prometheus rule file; add `--default-slo-period 30d` (default) and, to produce a Kubernetes `PrometheusRule` for the Prometheus Operator, use the `prometheus-operator/v1` spec kind (shown in [03-reference-and-cheatsheet.md](03-reference-and-cheatsheet.md#sloth-kubernetes-prometheusrule-spec)).

### 4. Load the rules into kube-prometheus-stack

Wrap the generated rule groups into a `PrometheusRule` object with the labels your Prometheus selects on (kube-prometheus-stack defaults to `release: kube-prometheus-stack`).

```bash
kubectl --namespace monitoring get prometheus -o jsonpath='{.items[0].spec.ruleSelector}'
```

```yaml
# monitoring/prometheus-rules/fastapi-slo-prometheusrule.yaml (header; paste Sloth's groups under spec)
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: fastapi-slo
  namespace: monitoring
  labels:
    release: kube-prometheus-stack
spec:
  groups: []   # <- replace with the groups: from the Sloth output
```

```bash
kubectl apply --filename monitoring/prometheus-rules/fastapi-slo-prometheusrule.yaml
kubectl --namespace monitoring get prometheusrule fastapi-slo
```

Expected output:

```text
NAME          AGE
fastapi-slo   5s
```

Verify Prometheus loaded the group (rule evaluation may take one interval):

```bash
curl --silent 'http://localhost:9090/api/v1/rules' | jq -r '.data.groups[].name' | grep sloth
```

```text
sloth-slo-sli-recordings-fastapi-availability
sloth-slo-meta-recordings-fastapi-availability
sloth-slo-alerts-fastapi-availability
```

### 5. Query the error budget

```promql
slo:period_error_budget_remaining:ratio{sloth_service="fastapi"}
```

A value of `1` means the whole budget is intact; `0.4` means 60% has been spent in the last 30 days; negative means the SLO is breached.

### 6. Install Pyrra (optional, UI-first alternative)

Pyrra is a Kubernetes operator with a web UI that shows every SLO, its budget, and burn-rate graphs. It is the quickest way to make SLOs visible to people who do not write PromQL.

```bash
kubectl apply --server-side --filename https://github.com/pyrra-dev/pyrra/releases/download/v0.7.0/setup.yaml
kubectl --namespace monitoring get pods -l app.kubernetes.io/name=pyrra
```

```yaml
# capstone/slo/pyrra/fastapi-availability.yaml
apiVersion: pyrra.dev/v1alpha1
kind: ServiceLevelObjective
metadata:
  name: fastapi-availability
  namespace: monitoring
  labels:
    prometheus: kube-prometheus-stack
    role: alert-rules
    team: platform
spec:
  target: "99.9"
  window: 4w
  description: Non-5xx ratio for the FastAPI service at the ingress.
  indicator:
    ratio:
      errors:
        metric: http_requests_total{service="fastapi",code=~"5.."}
      total:
        metric: http_requests_total{service="fastapi"}
```

```bash
kubectl apply --filename capstone/slo/pyrra/fastapi-availability.yaml
kubectl --namespace monitoring port-forward svc/pyrra-api 9099:9099
# open http://localhost:9099
```

Pyrra's `window: 4w` is 28 days, not 30; the arithmetic in this wiki uses 30 days (43,200 minutes). Either is fine as long as the policy and dashboards agree.

### 7. First dashboard panel

In Grafana, add a stat panel. Sloth does not emit a 30-day SLI recording rule directly; it emits the derived budget-remaining metadata rule, which is what you want on a stat panel:

```promql
100 * slo:period_error_budget_remaining:ratio{sloth_service="fastapi"}
```

For the 30-day achieved availability itself, compute it from the raw counters (this is the one place a long-range query is acceptable, because it is evaluated once per dashboard refresh rather than every rule interval):

```promql
100 * (1 - (
  sum(increase(http_requests_total{service="fastapi",code=~"5.."}[30d]))
  /
  sum(increase(http_requests_total{service="fastapi"}[30d]))
))
```

Title it "FastAPI availability: error budget remaining (%)". Thresholds: red below 0, orange below 25, green above.

---

## Verification Checklist

- [ ] `http_requests_total` and `http_request_duration_seconds_bucket` exist with `service="fastapi"` and `le="0.3"`.
- [ ] `sloth generate` runs in CI and the generated file is committed and reviewed.
- [ ] `PrometheusRule/fastapi-slo` is loaded and the three rule groups appear in Prometheus.
- [ ] `slo:period_error_budget_remaining:ratio` returns a value between 0 and 1.
- [ ] A one-page SLO document exists in `docs/slo/fastapi.md` using the template from [03-reference-and-cheatsheet.md](03-reference-and-cheatsheet.md#slo-document-template).
- [ ] The team has agreed an error budget policy and knows where it is.

## Next

Proceed to [02-core-concepts.md](02-core-concepts.md) for the theory behind the numbers, then [04-hands-on-labs.md](04-hands-on-labs.md) to build the latency SLO, the Spring Boot SLO, and the vLLM TTFT SLO.
