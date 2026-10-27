# Production Support: Troubleshooting — Anti-Patterns and Failure Modes of the Practice

| | |
|---|---|
| **Cluster** | 12 Enterprise Ops |
| **Role** | Bridge (extends 11-sre) |
| **Prerequisites** | [02-core-concepts.md](02-core-concepts.md), [05-production-patterns.md](05-production-patterns.md) |
| **Target version** | Kubernetes 1.31+, Prometheus 2.5x / 3.x, Argo Rollouts 1.7+ |
| **Capstone phase** | Phase 5 (Months 17-20) |

> Production support fails in two ways: the *practice* degrades (hero culture, alert fatigue, runbook rot, change theatre) or the *tooling* lies to you (missing metrics, broken correlation, stuck rollouts). This file lists both as symptom -> cause -> diagnose -> fix tables, with the exact error messages you will see, and ends with a debugging workflow for the practice itself.

---

## 1. Failure Modes of the Practice

| Symptom | Likely cause | Diagnose | Fix |
|---|---|---|---|
| One engineer resolves most incidents; MTTR doubles when they are on leave | Hero culture; knowledge in heads, not runbooks | Count incidents resolved per person over 90 days; check runbook coverage | Pair the hero with others on-call; "runbook or it did not happen" rule; rotate incident command |
| Pages ignored or acknowledged late; MTTA creeping up | Alert fatigue: too many non-actionable pages | Pages per shift > 2; share of pages with no action taken | Convert cause-based alerts to SLO burn-rate; demote to tickets; delete alerts nobody acts on |
| Runbook steps do not match reality during an incident | Runbook rot | `Last reviewed` older than 180 days; commands reference renamed resources | Runbook review in every shift; CI check on review date; test runbooks in game days |
| Changes queue for days; releases batched and risky | CAB theatre: approval without technical assessment | Lead time for changes in DORA metrics; CAB minutes show rubber stamping | Standard changes via pipeline gates; CAB only for high-blast-radius |
| Same incident recurs monthly | Postmortem action items never done; toil accepted as normal | Open action items older than 30 days; repeat alert names | Action items tracked as Jira issues with owners; toil hours in sprint planning |
| Toil keeps growing although automation ships | Automation of the wrong things; new services add toil faster than removal | Toil hours by category; automation backlog scored by hours saved | Score backlog by hours saved / effort; ORR blocks launches that add manual toil |
| Support and dev blame each other | Separate orgs, no shared metrics, "throw over the wall" | Escalation ping-pong in tickets | Owning team on-call for own service; platform team as L2 for platform only |
| Customers learn of outages before support does | No synthetic monitoring; alerts only on internal metrics | Compare incident `detected_by` field: customer vs alert | Blackbox probes from outside; status page automation |
| Error budget exhausted but features still shipping | No error-budget policy or no enforcement | Budget remaining < 0 with deploys continuing | Agree the policy; encode freeze in Kyverno/Argo CD sync windows |
| Incident channel chaos: many people, no decisions | No incident commander | Postmortem timeline shows parallel uncoordinated actions | IC role in every P1/P2; one scribe; status updates every 15 min |
| On-call burnout; attrition | Too few people, too many pages, no compensation | Rota size < 4; pages at night per week | Minimum rota of 4-6; compensate; reduce pages; follow-the-sun for global teams |
| ORR becomes a 200-item form nobody reads | Checklist without risk-based weighting | Launches delayed weeks; items checked without evidence | Mandatory vs recommended split; evidence links required; platform SLA to review in 5 days |
| AI agent proposes dangerous mitigations | Poor evals; mutating tools exposed; untrusted content in prompt | Trace tool calls; Promptfoo red-team results | Read-only tools; approval gate; treat logs as data; regression evals in CI |
| Break-glass used routinely | JIT path too slow or unknown | Count break-glass events per month | Make JIT fast (< 2 min via Slack); alert on break-glass use; postmortem each use |

## 2. Tooling Failure Modes

### Metrics and dashboards

| Symptom | Likely cause | Diagnose | Fix |
|---|---|---|---|
| Golden-signal panels empty for a service | OTel metrics not scraped; `service_name` label differs; wrong namespace | `curl $PROM/api/v1/label/service_name/values`; `kubectl get servicemonitor -A` | Fix ServiceMonitor/PodMonitor selector; align `service.name` with catalogue |
| Error ratio shows `NaN` | Division by zero at low traffic | Query `sum(rate(...count[5m]))` returns 0 | Wrap denominator in `clamp_min(..., 1e-9)` or use `or vector(0)` |
| p99 latency stuck at a bucket boundary (e.g. exactly 10 s) | Histogram buckets too coarse; requests above the top bucket | Inspect `le` values | Define explicit bucket boundaries (OTel View) or use native histograms |
| Latency looks fine but users complain | Measured server-side only; failures excluded | Compare with blackbox probe and client timings | Measure at load balancer/ingress; include errors in a separate latency series |
| Dashboard slow or Prometheus OOM | High-cardinality labels (user id, request id) | `topk(10, count by (__name__)({__name__=~".+"}))`; `/api/v1/status/tsdb` | Drop labels via OTel processor or `metric_relabel_configs`; use logs/traces for ids |
| `predict_linear` forecasts negative traffic | Window too short; step change in data | Plot the input series | Use a longer range with smoothing (`[14d:1h]`), exclude incident windows |

### Logs and traces

| Symptom | Likely cause | Diagnose | Fix |
|---|---|---|---|
| `request_id` found in Loki but no `trace_id` | OTel log correlation not configured; logs not structured | Check a raw log line for `trace_id` | Enable OTel logging instrumentation (Python `LoggingInstrumentor`, Java agent `otel.instrumentation.logback-appender`) |
| Trace id in log but Tempo returns `trace not found` | Sampling dropped the trace; Tempo retention; id format mismatch (hex vs decimal) | Check collector `tail_sampling` policies; `tempo-query` logs | Sample 100% of errors in tail sampling; align id formats |
| Trace has only the ingress span | Context propagation broken between services | Check `traceparent` header on inter-service calls | Instrument HTTP clients; ensure propagators `tracecontext,baggage` |
| Loki query `maximum of series (500) reached` | Too many label values in `sum by` | Reduce cardinality in the aggregation | Aggregate by `app`, not by `pod` or `request_id` |
| `logcli: error: 401 Unauthorized` | Multi-tenant Loki requires `X-Scope-OrgID` | Check `auth_enabled` | `export LOKI_ORG_ID=capstone` |

### kubectl triage

| Error (verbatim) | Cause | Fix |
|---|---|---|
| `error: ephemeral containers are disabled for this cluster` | Feature not enabled (pre-1.25) or managed cluster policy | Upgrade; on EKS 1.25+ it is on by default |
| `Error from server (Forbidden): pods "x" is forbidden: User "..." cannot create resource "pods/ephemeralcontainers"` | RBAC lacks `pods/ephemeralcontainers` | Add verb `update` on `pods/ephemeralcontainers` to the on-call role (write-ish: JIT only) |
| `kubectl debug node/... : container has runAsNonRoot and image will run as root` | Pod Security admission in target namespace | Use `--profile=sysadmin` in a namespace exempted for debugging, or `kube-system` |
| `Warning: would violate PodSecurity "restricted:latest"` when running debug pods | Namespace PSA labels | Run debug pods in a dedicated `debug` namespace with `baseline` level |
| `kubectl: command not found: kubectl-resource_capacity` | krew bin not on PATH | `export PATH="${KREW_ROOT:-$HOME/.krew}/bin:$PATH"` |
| `stern` shows nothing | Label selector mismatch or wrong namespace | `kubectl get pods -n capstone --show-labels` |
| `k9s` cannot connect / `Boom!! ... context deadline exceeded` | Kubeconfig expired OIDC token | Re-login (`aws eks update-kubeconfig`, `kubelogin`) |

### Rollouts and rollbacks

| Symptom | Likely cause | Diagnose | Fix |
|---|---|---|---|
| `kubectl rollout undo` does nothing | `revisionHistoryLimit: 0` or only one revision | `kubectl rollout history` | Set `revisionHistoryLimit: 10`; roll back via Git |
| Argo Rollouts canary stuck at `Paused` | Manual pause step or analysis `Inconclusive` | `kubectl argo rollouts get rollout ... ` shows `Inconclusive` | Fix the query returning no data (`successCondition` on empty result); `promote` manually if safe |
| Analysis fails immediately: `result[0] < 0.005` evaluated on empty | Prometheus returned no series for canary (no traffic yet) | Check `rollouts_pod_template_hash` label present on metrics | Add `initialDelay`; use `len(result) == 0 || result[0] < 0.005` |
| Argo CD rollback leaves app `OutOfSync` forever | Rollback is not in Git; auto-sync disabled | `argocd app get` | Revert in Git; re-enable auto-sync |
| `helm rollback` fails: `has no deployed releases` | Previous release failed; status `pending-upgrade` | `helm history` | `helm rollback <rel> <last good>`; or `--force`; migrate to Argo CD |
| Expand/contract: new release crashes with `column does not exist` | Contract step deployed before all readers upgraded | Check running image versions vs migration version | Roll back the contract migration (restore column), gate contract on zero old replicas |
| Freeze policy blocks an emergency fix | Missing `change.example.com/emergency: approved` annotation | Kyverno message in `kubectl apply` output | Follow emergency change process: approver adds annotation; reviewed after |

### Load testing

| Symptom | Likely cause | Diagnose | Fix |
|---|---|---|---|
| `k6` reports `dropped_iterations` | `maxVUs` too low for arrival rate given latency | Summary shows dropped iterations | Raise `maxVUs`; the service is slow, which is the finding |
| Thresholds pass locally, fail in CI | Review env smaller than staging; cold caches | Compare resources | Separate smoke (functional) and load (performance) thresholds |
| Load generator saturates before the service | k6 pod CPU-limited | `kubectl top pod k6-...` | Give k6 more CPU or distribute (k6-operator) |
| Prometheus remote-write `400 Bad Request` from k6 | Receiver not enabled | Prometheus logs | `enableRemoteWriteReceiver: true` |
| Load test tripped WAF/rate limits | Test traffic indistinguishable from attack | 429s / 403s | Allow-list the load generator IP or send a header the WAF trusts in staging only |

### Ticketing integration

| Symptom | Likely cause | Diagnose | Fix |
|---|---|---|---|
| No ticket created | Receiver misconfigured; token invalid; route never matched | `amtool config routes test --config.file alertmanager.yaml severity=ticket`; Alertmanager logs `notify retry` | Fix matcher order; rotate token; check `continue: true` |
| Duplicate tickets per alert | `group_by` too fine; `repeat_interval` too short; `reopen_duration` | Ticket labels | Group by `alertname, namespace, service_name`; raise `repeat_interval` |
| Tickets never close | No resolve handling | Receiver `send_resolved` | Enable `send_resolved: true` and an automation rule transitioning on resolve |

## 3. Debugging the Practice Itself: A Workflow

1. **Pull the numbers**: pages per shift, MTTA, MTTR, toil %, change failure rate, runbook coverage, tickets by alert, detected-by source.
2. **Find the largest deviation** from targets (for example pages per shift 7 vs target 2).
3. **Classify**: is it a *signal* problem (alerts not actionable), a *knowledge* problem (no runbook), a *process* problem (no owner, CAB delays), a *capacity* problem (chronic overload) or a *tooling* problem (broken correlation)?
4. **Fix the system, not the shift**: delete or demote alerts; write the runbook and automate it; assign ownership; add capacity; fix the pipeline.
5. **Re-measure next month** and record the change in the operations review.

## 4. Quick Diagnostic Commands

```bash
# Alertmanager: which route matches these labels?
amtool config routes test --config.file=alertmanager.yaml severity=page team=platform

# Alerts firing right now, grouped
amtool alert query --alertmanager.url=http://localhost:9093 -o extended

# Silence a noisy alert during an incident (always with a comment and expiry)
amtool silence add alertname=HighLatency service_name=spring-api --duration=2h --comment="INC-4821 known cause" --alertmanager.url=http://localhost:9093

# Which rules have no runbook_url?
kubectl get prometheusrules -A -o json | jq -r '.items[].spec.groups[].rules[] | select(.alert and (.annotations.runbook_url==null)) | .alert'

# Prometheus series cardinality by metric
curl -s $PROM/api/v1/status/tsdb | jq '.data.seriesCountByMetricName[:10]'

# Rollout status for everything in the namespace
kubectl argo rollouts list rollouts -n capstone

# Who has edit rights in capstone?
kubectl get rolebindings,clusterrolebindings -A -o json | jq -r '.items[] | select(.roleRef.name|test("edit|admin")) | "\(.kind) \(.metadata.name) -> \(.subjects // [] | map(.name) | join(","))"'
```

Continue with [07-interview-questions.md](07-interview-questions.md).
