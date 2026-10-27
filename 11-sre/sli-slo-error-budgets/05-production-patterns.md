# SLIs, SLOs and Error Budgets: Production Patterns

| | |
|---|---|
| **Cluster** | 11 SRE |
| **Role** | Anchor |
| **Prerequisites** | [02-core-concepts.md](02-core-concepts.md), [04-hands-on-labs.md](04-hands-on-labs.md) |
| **Target version** | Prometheus 2.55+/3.x, Sloth 0.12+, Pyrra 0.7+, Argo CD 2.12+ |
| **Capstone phase** | Phase 4 (Months 13-16) |

> How SLOs survive contact with production: a GitOps pipeline for SLOs as code, scaling recording rules across dozens of services, dependency and platform SLOs, multi-tenant SLOs, SLOs for AI workloads, the review process, security of the SLO pipeline, cost, anti-patterns, and a decision checklist.

---

## 1. SLOs as Code: The Pipeline

```mermaid
flowchart LR
    DEV[Service team edits slo/fastapi.yaml] --> PR[Pull request]
    PR --> CI1[sloth validate]
    CI1 --> CI2[sloth generate -> rules/]
    CI2 --> CI3[promtool check rules]
    CI3 --> CI4[promtool test rules<br/>unit tests from burn-rate-alerting]
    CI4 --> CI5[conftest: policy checks<br/>runbook_url present, severity in {page,ticket}]
    CI5 --> MERGE[Merge to main]
    MERGE --> ARGO[Argo CD syncs PrometheusRule]
    ARGO --> PROM[Prometheus reloads]
    PROM --> GRAF[Dashboard provisioned from the same spec]
```

### GitHub Actions job

```yaml
# .github/workflows/slo.yaml
name: slo
on:
  pull_request:
    paths: ["slo/**", "monitoring/**"]
permissions:
  contents: read
  pull-requests: write
jobs:
  generate-and-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Validate SLO specs
        run: docker run --rm -v "$PWD":/work ghcr.io/slok/sloth:v0.12.0 validate -i /work/slo
      - name: Generate rules
        run: |
          mkdir -p monitoring/prometheus-rules/generated
          docker run --rm -v "$PWD":/work ghcr.io/slok/sloth:v0.12.0 generate \
            -i /work/slo -o /work/monitoring/prometheus-rules/generated \
            --extra-labels env=prod
      - name: Fail if generated rules are not committed
        run: git diff --exit-code -- monitoring/prometheus-rules/generated
      - name: promtool check and test
        run: |
          docker run --rm -v "$PWD":/work --entrypoint promtool prom/prometheus:v3.0.0 \
            check rules /work/monitoring/prometheus-rules/generated/*.yaml
          docker run --rm -v "$PWD":/work --entrypoint promtool prom/prometheus:v3.0.0 \
            test rules /work/monitoring/tests/*.yaml
      - name: Policy checks
        run: |
          docker run --rm -v "$PWD":/work openpolicyagent/conftest:v0.56.0 \
            test /work/monitoring/prometheus-rules/generated --policy /work/monitoring/policy
```

### Conftest policy: every page alert has a runbook

```rego
# monitoring/policy/alerts.rego
package main

deny contains msg if {
  some group in input.groups
  some rule in group.rules
  rule.alert
  rule.labels.severity == "page"
  not startswith(rule.annotations.runbook_url, "https://runbooks.example.com/")
  msg := sprintf("alert %s (severity=page) has no runbook_url", [rule.alert])
}

deny contains msg if {
  some group in input.groups
  some rule in group.rules
  rule.alert
  not rule.labels.severity in {"page", "ticket"}
  msg := sprintf("alert %s has invalid severity %q", [rule.alert, rule.labels.severity])
}
```

### Where the artefacts live

| Artefact | Path | Owner | Reconciled by |
|---|---|---|---|
| Spec | `slo/<service>.yaml` | Service team | CI |
| Generated rules | `monitoring/prometheus-rules/generated/<service>.yaml` | CI (committed) | Argo CD |
| Unit tests | `monitoring/tests/<service>_test.yaml` | Service team | CI |
| SLO document | `docs/slo/<service>.md` | Service team + product | TechDocs |
| Error budget policy | `docs/slo/error-budget-policy.md` | Engineering management | TechDocs |
| Dashboards | `monitoring/dashboards/slo-<service>.json` | Platform (templated) | Grafana sidecar |

---

## 2. Scaling to Many Services

### Templated specs

Fifty services with the same shape should not be fifty copy-pasted specs. Options:

- **Sloth SLI plugins.** Write a plugin once (`sloth/plugins/http_availability`) and have each service's spec reference it with `sli.plugin.id` and options `{service: fastapi}`.
- **Helm chart for SLOs.** A chart with a values list of services generating `PrometheusServiceLevel` resources.
- **Backstage scaffolder.** The golden-path template creates `slo/<service>.yaml` alongside the service, so no service ships without an SLO.

```yaml
# Sloth plugin usage
slos:
  - name: availability
    objective: 99.9
    sli:
      plugin:
        id: "capstone/http_availability"
        options:
          service: "fastapi"
          exclude_routes: "/healthz|/metrics"
```

### Recording rule cost

Each SLO adds about 14 series and 14 rule evaluations per interval; trivial. The expensive part is long range vectors over high-cardinality raw metrics. Rules:

1. **Aggregate first.** Always `sum(...)` before the ratio so the rule produces one series, not one per pod.
2. **Chain windows from the 5m rule where exactness is not required.** `avg_over_time(slo:sli_error:ratio_rate5m[1h])` is cheap but is an average of ratios (wrong if traffic varies). The Workbook accepts this for alerting. For the budget-of-record, use true `increase()` over the window at a slower interval (5m).
3. **Separate rule groups by interval.** Alert-critical 5m/1h rules every 30 s; 1d/3d/30d rules every 5 m.
4. **Use Prometheus 3.x native histograms** for new services to cut bucket series 10-20x.

### Labels: the contract

Every SLO series carries `slo`, `service`, `team`. Dashboards, alerts, OnCall routing, and the Backstage SLO card all key on these. Enforce them in CI; never let a human add ad-hoc labels in generated files.

---

## 3. Dependency, Platform and Composite SLOs

### Platform SLOs

The platform team publishes SLOs for what product teams depend on: Kubernetes API availability, ingress availability and latency overhead (p99 added latency < 5 ms), Argo CD sync latency (95% of syncs complete < 3 min), Prometheus rule evaluation freshness, CI queue time. These are internal SLOs with internal error budget policies; when the ingress controller burns its budget, the platform team freezes its own changes.

### Dependency SLO negotiation

| Step | Example |
|---|---|
| Measure what you need | FastAPI needs Postgres connection acquisition < 50 ms for 99.9% of requests |
| Ask the owner for their SLO | DBA team publishes 99.95% availability, no latency SLO |
| Close the gap | Add a latency SLI on the FastAPI side (`db_acquire_seconds_bucket`), and a connection pool with a timeout so slow acquisitions become fast failures within budget |
| Record in the SLO document | Dependencies table with the published SLO and the mitigation |

### Composite journey SLOs

Prefer **end-to-end instrumentation** (one counter emitted at the edge of the journey: `journey_requests_total{journey="ask-agent",outcome}`) over multiplying component SLIs. Multiplication assumes independence and serial calls; a cache or fallback breaks both assumptions. When you must compute, document the formula and treat the result as an estimate.

---

## 4. Multi-Tenant SLOs

| Pattern | When | Implementation |
|---|---|---|
| Single aggregate SLO, per-tenant SLIs for diagnosis | Shared infrastructure, no isolation guarantees | `sum by (tenant)` recording rules, no per-tenant alerts; dashboard with tenant variable |
| Tiered SLOs | Premium tenants pay for a stronger target | Label requests with `tier`; separate SLO specs per tier; isolate capacity (separate node pools or rate limits) so the premium SLO is achievable |
| Per-tenant SLOs | Contractual commitments per customer | One spec per tenant via templating; alert routing per tenant; beware cardinality (hundreds of SLOs = thousands of series and alerts) |

Always exclude tenants from the valid set when their traffic is abusive and rate-limited (429 is good, not bad), otherwise one tenant's retry storm burns everyone's budget.

---

## 5. SLOs for AI Workloads

Treat the LLM endpoint and the agent as first-class services with the same discipline.

| Concern | Pattern |
|---|---|
| **TTFT versus total latency** | SLO on TTFT (user-perceived responsiveness); dashboard-only on tokens per second and total duration; a separate SLO on end-to-end task time for the agent |
| **Quality is sampled** | Online judge on a sample (5-10%); widen the window (90 days) and alert only on long windows; log judged examples for the postmortem |
| **Cost as a budget** | `agent_task_cost_usd` histogram; SLO "95% of tasks under $0.05"; FinOps dashboard sums `_sum`. Cost blowups are reliability incidents |
| **Model/provider changes** | A model version is a deploy. Canary it (see [../progressive-delivery/](../progressive-delivery/README.md)) with the grounded-rate SLI as the analysis metric |
| **Tool-call SLIs** | Per-MCP-server success ratio recorded with `sum by (tool)`; aggregate SLO at the agent level |
| **Guardrail refusals** | A refusal for a legitimate request is a quality failure; a refusal for an injection attempt is good. Label `refusal_reason` and decide which are valid events |
| **Capacity** | vLLM `num_requests_waiting` and `gpu_cache_usage_perc` are saturation signals for tickets, never SLIs |

---

## 6. The Review Process in Production

| Cadence | Attendees | Agenda | Artefact |
|---|---|---|---|
| Weekly (15 min) | Service team | Budget remaining, pages this week, false positives, open action items | Note in team channel |
| Monthly (45 min) | Team + product owner + SRE | SLO attainment per SLI; user complaints vs SLI (were we blind?); pages vs user impact (were we noisy?); target changes | Updated `docs/slo/<service>.md` history row |
| Quarterly (90 min) | Engineering leadership | Portfolio view across services; SLOs consistently exceeded (raise), consistently missed (invest or lower); retire unused SLOs | Portfolio report in TechDocs |

Signals that an SLO needs changing:

- Users complained while the SLI looked fine for three months: measurement point or specification is wrong.
- Budget never below 90%: target too loose or the service is over-engineered; raise the target or redeploy capacity elsewhere.
- Budget exhausted every month and nobody acted: the policy is not real; fix the policy or lower the target to something you will defend.

---

## 7. Security of the SLO Pipeline

- Recording and alert rules are **code with blast radius**: a malicious or careless rule can suppress every page. Protect `monitoring/` with CODEOWNERS (platform + SRE) and branch protection.
- The Sloth/Pyrra operators run with RBAC limited to `PrometheusRule` in `monitoring`; they need nothing else.
- Prometheus is read-only to Grafana and to the incident agent's MCP server; use a ServiceAccount token projected with a 1-hour lifetime, not a static token.
- SLO dashboards may be shared outside the team; they should not include tenant identifiers or user data. The SLI pipeline must not create labels from request content (no `user_id`, no query text).
- Audit: every change to a target is a commit with a review; the SLO document history table is the human-readable audit log.

---

## 8. Cost

| Item | Driver | Typical |
|---|---|---|
| Prometheus storage | Series count x retention; SLO rules add ~14 series per SLO | Negligible beside raw metrics |
| Long-range queries | 30d `increase()` on raw counters | Run as a rule every 5 m, never ad hoc on dashboards with many viewers |
| Pyrra / Sloth operators | One small pod each | < 100 m CPU, < 128 Mi |
| Judge calls for grounded rate | LLM tokens per judged answer | Sample 5-10%; use a small model for judging; cache by (question, answer) hash |
| The error budget policy itself | Feature freezes cost delivery | This is the point; make the cost visible to leadership so the target is chosen consciously |

---

## 9. Anti-Patterns

| Anti-pattern | Why it fails | Fix |
|---|---|---|
| Too many SLOs (one per metric) | Nothing is prioritised; alerts multiply; reviews stall | 1-3 SLOs per user journey; everything else is a dashboard |
| 100% target | No error budget; forbids all change; still unachievable | Pick a target from measured performance and user needs |
| Averages instead of percentiles or ratios | Averages hide the tail; p50 at 100 ms with p99 at 5 s looks fine | Ratio of requests under threshold (bucket / count) |
| Percentile as the SLI | Cannot be aggregated or averaged; interpolated | Ratio of good events; percentiles for dashboards only |
| Measuring in the app only | App down = no metrics = perfect SLI | Measure at the ingress; add synthetic probes |
| Including health checks in valid events | Inflates the denominator; hides real error ratio | Exclude probe routes |
| Excluding "planned maintenance" | Users do not care that it was planned | Count it; the budget is the maintenance allowance |
| SLO set by the platform team alone | No ownership; ignored by the service team | Service team writes the first draft; product owner signs the target |
| Error budget policy with no teeth | Budget exhausted, features ship anyway, SLO becomes a dashboard decoration | Named override authority; written exceptions; leadership reviews breaches |
| Changing the SLI implementation silently | Historical comparisons break; alerts shift | History row in the SLO document; version the spec |
| Alerting on budget remaining directly | Fires once at 3 a.m. when a slow leak crosses 0% | Burn-rate alerts (next topic); budget remaining is for reviews |
| One SLO for all tenants with a per-tenant SLA | Noisy neighbour breaches everyone | Tiering and isolation, or no per-tenant promises |

---

## 10. Decision Checklist

Before an SLO goes live:

- [ ] Specification written in plain English and agreed with the product owner.
- [ ] Implementation is a good/valid ratio; not an average, not a percentile.
- [ ] Measurement point chosen deliberately (ingress by default) and documented.
- [ ] Histogram boundaries include the latency threshold exactly.
- [ ] Valid events exclude probes and load tests; 4xx policy decided.
- [ ] Target set from 2-4 weeks of measured data; SLO stricter than any SLA.
- [ ] Window chosen (rolling 30 d default); budget computed in minutes and ppm.
- [ ] Spec committed to `slo/`, rules generated in CI, unit tests pass.
- [ ] Labels `slo`, `service`, `team` present on every series.
- [ ] Error budget policy written, with thresholds, actions, and override authority.
- [ ] Dashboard exists with budget remaining and burn-down; linked from the Backstage catalog entry.
- [ ] Burn-rate alerts exist with `runbook_url` (next topic).
- [ ] Review cadence on the calendar.

See also [../burn-rate-alerting/05-production-patterns.md](../burn-rate-alerting/05-production-patterns.md) for the alerting side and [../../12-enterprise-ops/production-support/README.md](../../12-enterprise-ops/production-support/README.md) for how SLOs feed the support model.
