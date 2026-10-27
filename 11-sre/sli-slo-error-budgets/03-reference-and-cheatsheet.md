# SLIs, SLOs and Error Budgets: Reference and Cheatsheet

| | |
|---|---|
| **Cluster** | 11 SRE |
| **Role** | Anchor |
| **Prerequisites** | [02-core-concepts.md](02-core-concepts.md) |
| **Target version** | Prometheus 2.55+/3.x, Sloth 0.12+, Pyrra 0.7+, OpenSLO v1 |
| **Capstone phase** | Phase 4 (Months 13-16) |

> Everything you need to copy: formulas, the nines table, complete PromQL for availability and latency SLIs from counters and histograms, full Sloth / Pyrra / OpenSLO / Grafana SLO specs, hand-written recording rules, the SLO document template, and a budget calculator in Python.

---

## 1. Formulas

| Quantity | Formula | 99.9% / 30 d example |
|---|---|---|
| Error budget fraction | `1 - target` | 0.001 |
| Budget in minutes | `window_minutes x (1 - target)` | 43,200 x 0.001 = 43.2 |
| Budget in requests | `total_requests x (1 - target)` | 1,000,000 x 0.001 = 1,000 |
| Error ratio over window | `bad / valid` | 600 / 1,000,000 = 0.0006 |
| Budget remaining | `1 - error_ratio / (1 - target)` | 1 - 0.0006/0.001 = 0.4 |
| Burn rate | `error_ratio / (1 - target)` | 0.0144 / 0.001 = 14.4 |
| Time to exhaustion | `window / burn_rate` | 30 d / 14.4 = 2.08 d = 50 h |
| Budget consumed by an alert window | `burn_rate x window_len / period` | 14.4 x 1 h / 720 h = 2% |
| Burn rate for a budget fraction in a window | `fraction x period / window_len` | 0.02 x 720 h / 1 h = 14.4 |
| Serial dependency bound | `product of dependency availabilities` | 0.9995 x 0.995 = 0.9945 |
| Window-based SLI | `good_windows / total_windows` | 43,157 / 43,200 = 99.9% |

## 2. The Nines Table

| Target | Budget | Per 30 days | Per 90 days | Per year | ppm failed |
|---|---|---|---|---|---|
| 90% | 10% | 72 h | 216 h | 36.5 d | 100,000 |
| 95% | 5% | 36 h | 108 h | 18.25 d | 50,000 |
| 99% | 1% | 7 h 12 m | 21 h 36 m | 3.65 d | 10,000 |
| 99.5% | 0.5% | 3 h 36 m | 10 h 48 m | 1.83 d | 5,000 |
| 99.9% | 0.1% | 43.2 m | 2 h 9.6 m | 8 h 46 m | 1,000 |
| 99.95% | 0.05% | 21.6 m | 64.8 m | 4 h 23 m | 500 |
| 99.99% | 0.01% | 4.32 m | 12.96 m | 52.6 m | 100 |
| 99.999% | 0.001% | 25.9 s | 77.8 s | 5.26 m | 10 |

## 3. PromQL Reference

### 3.1 Availability SLI from a counter

```promql
# Good ratio (the SLI), 5-minute window, FastAPI at the ingress
sum(rate(http_requests_total{service="fastapi",code!~"5.."}[5m]))
/
sum(rate(http_requests_total{service="fastapi"}[5m]))

# Error ratio (what recording rules store; 1 - SLI)
sum(rate(http_requests_total{service="fastapi",code=~"5.."}[5m]))
/
sum(rate(http_requests_total{service="fastapi"}[5m]))

# Spring Boot (Micrometer naming)
sum(rate(http_server_requests_seconds_count{service="springboot",status=~"5.."}[5m]))
/
sum(rate(http_server_requests_seconds_count{service="springboot"}[5m]))

# Excluding health checks and a load-test route from the valid set
sum(rate(http_requests_total{service="fastapi",code=~"5..",route!~"/healthz|/metrics"}[5m]))
/
sum(rate(http_requests_total{service="fastapi",route!~"/healthz|/metrics"}[5m]))

# NGINX ingress controller as the measurement point
sum(rate(nginx_ingress_controller_requests{exported_service="fastapi",status=~"5.."}[5m]))
/
sum(rate(nginx_ingress_controller_requests{exported_service="fastapi"}[5m]))
```

### 3.2 Latency SLI from a classic histogram

```promql
# SLI: fraction of requests at or under 300 ms (exact when 0.3 is a bucket boundary)
sum(rate(http_request_duration_seconds_bucket{service="fastapi",le="0.3"}[5m]))
/
sum(rate(http_request_duration_seconds_count{service="fastapi"}[5m]))

# Error ratio form for recording rules
1 - (
  sum(rate(http_request_duration_seconds_bucket{service="fastapi",le="0.3"}[5m]))
  /
  sum(rate(http_request_duration_seconds_count{service="fastapi"}[5m]))
)

# Equivalent using the +Inf bucket as the denominator (identical to _count)
sum(rate(http_request_duration_seconds_bucket{service="fastapi",le="0.3"}[5m]))
/
sum(rate(http_request_duration_seconds_bucket{service="fastapi",le="+Inf"}[5m]))

# Only count latency of successful requests as valid (fast 500s should not count as "good")
sum(rate(http_request_duration_seconds_bucket{service="fastapi",le="0.3",code!~"5.."}[5m]))
/
sum(rate(http_request_duration_seconds_count{service="fastapi",code!~"5.."}[5m]))

# Spring Boot Micrometer histogram (requires management.metrics.distribution.slo.http.server.requests=300ms)
sum(rate(http_server_requests_seconds_bucket{service="springboot",le="0.3"}[5m]))
/
sum(rate(http_server_requests_seconds_count{service="springboot"}[5m]))

# Percentile for dashboards (NOT an SLI)
histogram_quantile(0.99, sum by (le) (rate(http_request_duration_seconds_bucket{service="fastapi"}[5m])))

# Native histogram (Prometheus 3.x): fraction between 0 and 0.3 s
histogram_fraction(0, 0.3, sum(rate(http_request_duration_seconds{service="fastapi"}[5m])))
```

### 3.3 AI SLIs

```promql
# TTFT: fraction of completions with first token under 800 ms
sum(rate(vllm:time_to_first_token_seconds_bucket{le="0.8"}[5m]))
/
sum(rate(vllm:time_to_first_token_seconds_count[5m]))

# Completion success ratio
sum(rate(vllm:request_success_total{finished_reason=~"stop|length"}[5m]))
/
(sum(rate(vllm:request_success_total[5m])) + sum(rate(vllm:request_failure_total[5m])))

# Grounded answer rate (application counter, judge-sampled)
sum(rate(agent_answers_total{grounded="true"}[1h]))
/
sum(rate(agent_answers_total[1h]))

# Tool-call success
sum(rate(agent_tool_calls_total{status="ok"}[5m]))
/
sum(rate(agent_tool_calls_total[5m]))

# Cost per task: fraction of tasks under 5 cents
sum(rate(agent_task_cost_usd_bucket{le="0.05"}[1h]))
/
sum(rate(agent_task_cost_usd_count[1h]))

# Average cost per task (budget dashboard, not an SLI)
sum(rate(agent_task_cost_usd_sum[1h])) / sum(rate(agent_task_cost_usd_count[1h]))
```

### 3.4 Budget and burn

```promql
# 30-day error ratio
sum(increase(http_requests_total{service="fastapi",code=~"5.."}[30d]))
/
sum(increase(http_requests_total{service="fastapi"}[30d]))

# Budget remaining (fraction) for 99.9%
1 - (
  sum(increase(http_requests_total{service="fastapi",code=~"5.."}[30d]))
  / sum(increase(http_requests_total{service="fastapi"}[30d]))
) / 0.001

# Budget remaining in minutes (approximate: fraction x 43.2)
43.2 * (1 - (
  sum(increase(http_requests_total{service="fastapi",code=~"5.."}[30d]))
  / sum(increase(http_requests_total{service="fastapi"}[30d]))
) / 0.001)

# Burn rate over 1 hour (from the recording rule)
slo:sli_error:ratio_rate1h{slo="fastapi-availability"} / 0.001

# Window-based SLI: fraction of 1-minute windows in 30 d where error ratio < 1%
avg_over_time(
  (
    (sum(rate(http_requests_total{service="fastapi",code=~"5.."}[1m]))
     / sum(rate(http_requests_total{service="fastapi"}[1m]))) < bool 0.01
  )[30d:1m]
)
```

### 3.5 Hand-written recording rules (what the generators produce)

```yaml
# monitoring/prometheus-rules/fastapi-slo-recordings.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: fastapi-slo-recordings
  namespace: monitoring
  labels:
    release: kube-prometheus-stack
spec:
  groups:
    - name: slo-fastapi-availability-recordings
      interval: 30s
      rules:
        - record: slo:sli_error:ratio_rate5m
          labels: {slo: fastapi-availability, service: fastapi, team: platform}
          expr: |
            sum(rate(http_requests_total{service="fastapi",code=~"5.."}[5m]))
            /
            sum(rate(http_requests_total{service="fastapi"}[5m]))
        - record: slo:sli_error:ratio_rate30m
          labels: {slo: fastapi-availability, service: fastapi, team: platform}
          expr: |
            sum(rate(http_requests_total{service="fastapi",code=~"5.."}[30m]))
            /
            sum(rate(http_requests_total{service="fastapi"}[30m]))
        - record: slo:sli_error:ratio_rate1h
          labels: {slo: fastapi-availability, service: fastapi, team: platform}
          expr: |
            sum(rate(http_requests_total{service="fastapi",code=~"5.."}[1h]))
            /
            sum(rate(http_requests_total{service="fastapi"}[1h]))
        - record: slo:sli_error:ratio_rate2h
          labels: {slo: fastapi-availability, service: fastapi, team: platform}
          expr: |
            sum(rate(http_requests_total{service="fastapi",code=~"5.."}[2h]))
            /
            sum(rate(http_requests_total{service="fastapi"}[2h]))
        - record: slo:sli_error:ratio_rate6h
          labels: {slo: fastapi-availability, service: fastapi, team: platform}
          expr: |
            sum(rate(http_requests_total{service="fastapi",code=~"5.."}[6h]))
            /
            sum(rate(http_requests_total{service="fastapi"}[6h]))
        - record: slo:sli_error:ratio_rate1d
          labels: {slo: fastapi-availability, service: fastapi, team: platform}
          expr: |
            sum(rate(http_requests_total{service="fastapi",code=~"5.."}[1d]))
            /
            sum(rate(http_requests_total{service="fastapi"}[1d]))
        - record: slo:sli_error:ratio_rate3d
          labels: {slo: fastapi-availability, service: fastapi, team: platform}
          expr: |
            sum(rate(http_requests_total{service="fastapi",code=~"5.."}[3d]))
            /
            sum(rate(http_requests_total{service="fastapi"}[3d]))
        # Metadata
        - record: slo:objective:ratio
          labels: {slo: fastapi-availability, service: fastapi, team: platform}
          expr: vector(0.999)
        - record: slo:error_budget:ratio
          labels: {slo: fastapi-availability, service: fastapi, team: platform}
          expr: vector(1 - 0.999)
        - record: slo:period:days
          labels: {slo: fastapi-availability, service: fastapi, team: platform}
          expr: vector(30)
        - record: slo:sli_error:ratio_rate30d
          labels: {slo: fastapi-availability, service: fastapi, team: platform}
          expr: |
            sum(increase(http_requests_total{service="fastapi",code=~"5.."}[30d]))
            /
            sum(increase(http_requests_total{service="fastapi"}[30d]))
        - record: slo:error_budget:remaining
          labels: {slo: fastapi-availability, service: fastapi, team: platform}
          expr: |
            1 - (
              slo:sli_error:ratio_rate30d{slo="fastapi-availability"}
              / on() group_left slo:error_budget:ratio{slo="fastapi-availability"}
            )
        - record: slo:burn_rate:1h
          labels: {slo: fastapi-availability, service: fastapi, team: platform}
          expr: |
            slo:sli_error:ratio_rate1h{slo="fastapi-availability"} / 0.001

    - name: slo-fastapi-latency-recordings
      interval: 30s
      rules:
        - record: slo:sli_error:ratio_rate5m
          labels: {slo: fastapi-latency, service: fastapi, team: platform}
          expr: |
            1 - (
              sum(rate(http_request_duration_seconds_bucket{service="fastapi",le="0.3"}[5m]))
              /
              sum(rate(http_request_duration_seconds_count{service="fastapi"}[5m]))
            )
        # ... repeat for 30m, 1h, 2h, 6h, 1d, 3d with the same expression and window
        - record: slo:objective:ratio
          labels: {slo: fastapi-latency, service: fastapi, team: platform}
          expr: vector(0.99)
        - record: slo:error_budget:ratio
          labels: {slo: fastapi-latency, service: fastapi, team: platform}
          expr: vector(1 - 0.99)
```

A 30-day `increase()` rule is expensive on a busy Prometheus; generators avoid it by deriving budget remaining from the 3-day rule or from a `sum_over_time` of the 5-minute rule. Sloth computes `slo:period_error_budget_remaining:ratio` as `1 - slo:period_burn_rate:ratio` where period burn rate uses the longest window available; accept the approximation or run the 30d rule at a 5-minute interval.

## 4. Sloth

### 4.1 Plain Prometheus spec (CLI)

```yaml
version: "prometheus/v1"
service: "fastapi"
labels:
  team: "platform"
slos:
  - name: "availability"
    objective: 99.9
    description: "Non-5xx ratio at the ingress."
    sli:
      events:
        error_query: sum(rate(http_requests_total{service="fastapi",code=~"5.."}[{{.window}}]))
        total_query: sum(rate(http_requests_total{service="fastapi"}[{{.window}}]))
    alerting:
      name: FastAPIHighErrorBudgetBurn
      labels: {service: fastapi}
      annotations:
        summary: "FastAPI is burning its availability error budget too fast"
        runbook_url: https://runbooks.example.com/fastapi-5xx-burn-rate
      page_alert:
        labels: {severity: page}
      ticket_alert:
        labels: {severity: ticket}

  - name: "latency"
    objective: 99
    description: "Requests served in under 300 ms."
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

Note the latency `error_query` is `count - bucket` (requests slower than 300 ms), which keeps the error/total form Sloth expects.

### 4.2 Sloth Kubernetes PrometheusRule spec

```yaml
apiVersion: sloth.slok.dev/v1
kind: PrometheusServiceLevel
metadata:
  name: fastapi
  namespace: monitoring
spec:
  service: "fastapi"
  labels:
    team: "platform"
  slos:
    - name: "availability"
      objective: 99.9
      sli:
        events:
          error_query: sum(rate(http_requests_total{service="fastapi",code=~"5.."}[{{.window}}]))
          total_query: sum(rate(http_requests_total{service="fastapi"}[{{.window}}]))
      alerting:
        name: FastAPIHighErrorBudgetBurn
        annotations:
          runbook_url: https://runbooks.example.com/fastapi-5xx-burn-rate
        page_alert:
          labels: {severity: page}
        ticket_alert:
          labels: {severity: ticket}
```

Generate a `PrometheusRule` from this with `sloth generate -i spec.yaml` (output kind is `PrometheusRule` when input kind is `PrometheusServiceLevel`), or run the Sloth operator (`helm install sloth sloth/sloth`) to reconcile it in-cluster.

### 4.3 Sloth CLI

| Command | Purpose |
|---|---|
| `sloth generate -i slo/fastapi.yaml -o out.yaml` | Generate rules |
| `sloth generate -i slo/ -o rules/` | Generate for a directory |
| `sloth validate -i slo/` | Validate specs (use in CI) |
| `sloth generate --default-slo-period 28d ...` | Change period |
| `sloth generate --disable-recordings` / `--disable-alerts` | Partial output |
| `sloth generate --sli-plugins-path ./plugins ...` | Use SLI plugins |
| `sloth generate --extra-labels env=prod ...` | Add labels to every rule |

## 5. Pyrra

```yaml
apiVersion: pyrra.dev/v1alpha1
kind: ServiceLevelObjective
metadata:
  name: fastapi-latency
  namespace: monitoring
  labels:
    prometheus: kube-prometheus-stack
    role: alert-rules
spec:
  target: "99"
  window: 4w
  description: 99% of FastAPI requests under 300 ms.
  indicator:
    latency:
      success:
        metric: http_request_duration_seconds_bucket{service="fastapi",le="0.3"}
      total:
        metric: http_request_duration_seconds_count{service="fastapi"}
  alerting:
    name: FastAPIHighLatencyBudgetBurn
---
apiVersion: pyrra.dev/v1alpha1
kind: ServiceLevelObjective
metadata:
  name: vllm-ttft
  namespace: monitoring
  labels:
    prometheus: kube-prometheus-stack
    role: alert-rules
spec:
  target: "95"
  window: 4w
  indicator:
    latency:
      success:
        metric: vllm:time_to_first_token_seconds_bucket{le="0.8"}
      total:
        metric: vllm:time_to_first_token_seconds_count
---
apiVersion: pyrra.dev/v1alpha1
kind: ServiceLevelObjective
metadata:
  name: agent-grounded
  namespace: monitoring
  labels:
    prometheus: kube-prometheus-stack
    role: alert-rules
spec:
  target: "98"
  window: 4w
  indicator:
    ratio:
      errors:
        metric: agent_answers_total{grounded="false"}
      total:
        metric: agent_answers_total
```

Pyrra also supports `indicator.boolGauge` (for probe-style SLIs such as `probe_success`) and `latencyNative` for native histograms.

## 6. OpenSLO v1

```yaml
apiVersion: openslo/v1
kind: SLO
metadata:
  name: fastapi-availability
  displayName: FastAPI availability
  labels:
    team: platform
spec:
  description: Non-5xx ratio for the FastAPI service at the ingress.
  service: fastapi
  indicator:
    metadata:
      name: fastapi-non-5xx
    spec:
      ratioMetric:
        counter: true
        good:
          metricSource:
            type: Prometheus
            spec:
              query: sum(rate(http_requests_total{service="fastapi",code!~"5.."}[5m]))
        total:
          metricSource:
            type: Prometheus
            spec:
              query: sum(rate(http_requests_total{service="fastapi"}[5m]))
  timeWindow:
    - duration: 30d
      isRolling: true
  budgetingMethod: Occurrences
  objectives:
    - displayName: three nines
      target: 0.999
  alertPolicies:
    - fastapi-fast-burn
---
apiVersion: openslo/v1
kind: AlertPolicy
metadata:
  name: fastapi-fast-burn
spec:
  description: Page when burning 2% of budget in 1 hour.
  alertWhenBreaching: true
  conditions:
    - conditionRef: fast-burn
  notificationTargets:
    - targetRef: oncall-page
---
apiVersion: openslo/v1
kind: AlertCondition
metadata:
  name: fast-burn
spec:
  severity: page
  condition:
    kind: burnrate
    op: gte
    threshold: 14.4
    lookbackWindow: 1h
    alertAfter: 5m
```

`budgetingMethod: Occurrences` is request-based; `Timeslices` is window-based. Convert with `oslo convert` or the Sloth OpenSLO plugin.

## 7. Grafana SLO (Cloud / Enterprise)

```yaml
# Terraform provider grafana, resource grafana_slo
resource "grafana_slo" "fastapi_availability" {
  name        = "FastAPI availability"
  description = "Non-5xx ratio at the ingress"
  query {
    type = "ratio"
    ratio {
      success_metric = "http_requests_total{service=\"fastapi\",code!~\"5..\"}"
      total_metric   = "http_requests_total{service=\"fastapi\"}"
    }
  }
  objectives {
    value  = 0.999
    window = "30d"
  }
  alerting {
    fastburn {
      labels { key = "severity" value = "page" }
      annotations { key = "runbook_url" value = "https://runbooks.example.com/fastapi-5xx-burn-rate" }
    }
    slowburn {
      labels { key = "severity" value = "ticket" }
    }
  }
}
```

## 8. Grafana Dashboard Panels (PromQL)

| Panel | Query | Visual |
|---|---|---|
| Budget remaining % | `100 * slo:error_budget:remaining{slo="fastapi-availability"}` | Stat, thresholds 0 / 25 / 50 |
| Achieved SLI 30 d | `100 * (1 - slo:sli_error:ratio_rate30d{slo="fastapi-availability"})` | Stat vs objective line |
| Burn rate (1h, 6h, 3d) | `slo:sli_error:ratio_rate1h{slo="fastapi-availability"} / 0.001` and 6h, 3d | Time series with threshold lines at 14.4, 6, 1 |
| Error ratio 5m | `slo:sli_error:ratio_rate5m{slo="fastapi-availability"}` | Time series, log scale |
| Budget burn-down | `43.2 * slo:error_budget:remaining{...}` | Time series, last 30 d |
| Latency distribution | `histogram_quantile(0.5/0.9/0.99, sum by (le)(rate(...bucket[5m])))` | Time series with 0.3 reference line |
| Good vs bad request rate | `sum(rate(http_requests_total{service="fastapi",code!~"5.."}[5m]))` / `code=~"5.."` | Stacked |

Add a Grafana annotation query on `ALERTS{alertname=~".*BudgetBurn", alertstate="firing"}` so pages appear on the burn-down chart.

## 9. SLO Document Template

Store at `docs/slo/<service>.md`. One per service.

```markdown
# SLO: <service name>

| | |
|---|---|
| **Service owner** | <team> |
| **Status** | Draft / Active / Retired |
| **Effective from** | YYYY-MM-DD |
| **Review cadence** | Monthly (owner + product), quarterly (targets) |
| **SLA offered** | 99.5% availability (stricter SLO below) |
| **Dashboards** | <Grafana link> |
| **Spec as code** | `slo/<service>.yaml` |

## User journeys covered

1. <journey> - <why it matters>

## SLIs

### Availability
- **Specification.** Proportion of HTTP requests that return a non-5xx status, measured at the ingress controller. Health checks and `/metrics` are excluded from valid events.
- **Implementation.** `sum(rate(http_requests_total{service="<svc>",code=~"5.."}[W])) / sum(rate(http_requests_total{service="<svc>"}[W]))`
- **Measurement point.** Ingress (NGINX / Gateway API). Rationale: counts requests the app never saw.

### Latency
- **Specification.** Proportion of requests served in under 300 ms, measured at the ingress.
- **Implementation.** `sum(rate(http_request_duration_seconds_bucket{service="<svc>",le="0.3"}[W])) / sum(rate(http_request_duration_seconds_count{service="<svc>"}[W]))`

## SLOs

| SLI | Target | Window | Budget / 30 d | Rationale |
|---|---|---|---|---|
| Availability | 99.9% | Rolling 30 d | 43.2 min / 1,000 ppm | Achieved 99.94% over the last 60 d; users notice outages > 5 min |
| Latency | 99% < 300 ms | Rolling 30 d | 432 min / 10,000 ppm | p99 currently 240 ms |

## Error budget policy

| Remaining | Action |
|---|---|
| > 50% | Normal cadence; spend budget on experiments |
| 25-50% | Canary + analysis mandatory |
| 0-25% | Reliability and security changes only |
| < 0% | Feature freeze until positive; reliability sprint |

Override authority: <role>. Exceptions recorded in `docs/slo/exceptions.md`.

## Alerting

Multi-window multi-burn-rate per the SRE Workbook; see `monitoring/prometheus-rules/<service>-slo.yaml`. Page: 14.4x/1h+5m, 6x/6h+30m. Ticket: 3x/1d+2h, 1x/3d+6h.

## Dependencies and their SLOs

| Dependency | Their SLO | Critical path? | Mitigation |
|---|---|---|---|
| postgres | 99.95% | Yes | Connection pool, read replica |
| vllm | 99.5% | No (degrades to non-AI answer) | Fallback + quality SLI |

## Known gaps

- <e.g. client-side failures invisible>

## History

| Date | Change | Reason |
|---|---|---|
| YYYY-MM-DD | Created at 99.5% | Baseline |
| YYYY-MM-DD | Raised to 99.9% | Achieved 99.95% for two quarters |
```

## 10. Error Budget Calculator (Python)

```python
#!/usr/bin/env python3
"""Error budget calculator. Usage: python slo_calc.py 99.9 30 50"""
import sys

def calc(target_pct: float, window_days: int, rps: float | None = None) -> None:
    target = target_pct / 100
    budget = 1 - target
    minutes = window_days * 24 * 60
    print(f"Target            : {target_pct}% over {window_days} days")
    print(f"Budget fraction   : {budget:.6f}")
    print(f"Budget minutes    : {minutes * budget:.2f} min")
    print(f"Budget per million: {budget * 1_000_000:.0f} failed requests")
    if rps:
        total = rps * 86_400 * window_days
        print(f"Requests/window   : {total:,.0f}")
        print(f"Budget requests   : {total * budget:,.0f}")
    print("\nBurn-rate table (SRE Workbook):")
    hours = window_days * 24
    for frac, win_h, action in [(0.02, 1, "page"), (0.05, 6, "page"), (0.10, 24, "ticket"), (0.10, 72, "ticket")]:
        br = frac * hours / win_h
        print(f"  {frac*100:>4.0f}% of budget in {win_h:>3}h -> burn rate {br:>5.1f}x ({action})")

if __name__ == "__main__":
    args = [float(a) for a in sys.argv[1:]]
    calc(args[0] if args else 99.9, int(args[1]) if len(args) > 1 else 30, args[2] if len(args) > 2 else None)
```

Expected output for `python slo_calc.py 99.9 30 50`:

```text
Target            : 99.9% over 30 days
Budget fraction   : 0.001000
Budget minutes    : 43.20 min
Budget per million: 1000 failed requests
Requests/window   : 129,600,000
Budget requests   : 129,600

Burn-rate table (SRE Workbook):
     2% of budget in   1h -> burn rate  14.4x (page)
     5% of budget in   6h -> burn rate   6.0x (page)
    10% of budget in  24h -> burn rate   3.0x (ticket)
    10% of budget in  72h -> burn rate   1.0x (ticket)
```

## 11. Quick Decision Tables

**Which SLI type?**

| User complaint | SLI |
|---|---|
| "It's down" / "I got an error" | Availability |
| "It's slow" | Latency |
| "It works but the answer is wrong / stale" | Quality or freshness |
| "Half my batch never arrived" | Coverage / throughput |

**Which measurement point?**

| Situation | Point |
|---|---|
| Default for HTTP services | Ingress / LB |
| Need per-tenant or route labels | Application (with ingress as the SLO of record) |
| Service receives < 1 request per minute | Synthetic probe + window-based SLI |
| Front end, CDN in path | RUM as secondary SLI |

**Which tool?**

| Need | Tool |
|---|---|
| Rules reviewed in Git, CI-generated | Sloth CLI |
| UI for non-PromQL users on OSS | Pyrra |
| Portable spec across vendors | OpenSLO |
| Already on Grafana Cloud | Grafana SLO |
