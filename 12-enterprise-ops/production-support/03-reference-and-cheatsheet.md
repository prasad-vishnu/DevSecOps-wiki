# Production Support: Reference and Cheat Sheet

| | |
|---|---|
| **Cluster** | 12 Enterprise Ops |
| **Role** | Bridge (extends 11-sre) |
| **Prerequisites** | [02-core-concepts.md](02-core-concepts.md) |
| **Target version** | Kubernetes 1.31+, Prometheus 2.5x / 3.x, Loki 3.x, Tempo 2.x, Argo Rollouts 1.7+, k6 0.5x+ |
| **Capstone phase** | Phase 5 (Months 17-20) |

> Everything you type during a support shift, organised by task: golden-signal PromQL, LogQL and TraceQL for request tracing, kubectl triage one-liners (long and short forms), Argo Rollouts and Argo CD rollback commands, k6 basics, and the templates and checklists (priority matrix, runbook, handover, change record, ORR summary). Each entry has a one-line explanation.

---

## 1. Frameworks at a Glance

| Framework | Elements | Use it for |
|---|---|---|
| Four Golden Signals | Latency, Traffic, Errors, Saturation | First dashboard, first five minutes of any incident |
| RED | Rate, Errors, Duration | Per-service health from the caller's viewpoint |
| USE | Utilisation, Saturation, Errors | Per-resource bottleneck hunting |
| DORA four keys | Deployment frequency, lead time, change failure rate, time to restore | Measuring delivery and stability together |
| Toil model | Manual, repetitive, automatable, tactical, no enduring value, scales with growth; 50% cap | Deciding what to automate |
| ITIL 4 change types | Standard (pre-approved), Normal (assessed), Emergency (post-reviewed) | Classifying changes without a weekly board |
| Priority matrix | Impact x urgency -> P1..P4 | Setting response/resolution SLAs |

## 2. Formulas

| Quantity | Formula | Example |
|---|---|---|
| Error ratio | 5xx / total over a window | 120 / 48,000 = 0.25% |
| Availability SLI | 1 - error ratio (good/valid events) | 99.75% |
| Error budget (30 d, 99.9%) | (1 - SLO) x 30 x 24 x 60 min | 43.2 min |
| Burn rate | observed error ratio / (1 - SLO) | 0.5% / 0.1% = 5x |
| MTTA | mean(ack time - alert time) | 3.2 min |
| MTTR | mean(mitigation time - start time) | 27 min |
| Change failure rate | failed changes / total changes | 4 / 62 = 6.5% |
| Toil % | toil hours / team hours | 38 / 160 = 24% |
| Automation rate | alerts auto-resolved / alerts | 91 / 210 = 43% |
| Required replicas | ceil(peak_rps x headroom / (per_replica_rps x target_util)) | ceil(1800 x 1.5 / (450 x 0.7)) = 9 |
| Headroom for N-1 AZ | capacity x (N / (N-1)) | 3 AZs: x 1.5 |

## 3. PromQL Cheat Sheet (Golden Signals and Beyond)

```promql
# Traffic per service (rps)
sum by (service_name) (rate(http_server_request_duration_seconds_count{namespace="capstone"}[5m]))

# Traffic per route and method
sum by (service_name, http_request_method, http_route) (rate(http_server_request_duration_seconds_count{namespace="capstone"}[5m]))

# Error ratio per service
sum by (service_name) (rate(http_server_request_duration_seconds_count{namespace="capstone",http_response_status_code=~"5.."}[5m]))
/ clamp_min(sum by (service_name) (rate(http_server_request_duration_seconds_count{namespace="capstone"}[5m])), 1e-9)

# Error ratio per route (find the broken endpoint)
topk(5, sum by (http_route) (rate(http_server_request_duration_seconds_count{service_name="fastapi-api",http_response_status_code=~"5.."}[5m])))

# p50 / p95 / p99 latency
histogram_quantile(0.5,  sum by (service_name, le) (rate(http_server_request_duration_seconds_bucket{namespace="capstone"}[5m])))
histogram_quantile(0.95, sum by (service_name, le) (rate(http_server_request_duration_seconds_bucket{namespace="capstone"}[5m])))
histogram_quantile(0.99, sum by (service_name, le) (rate(http_server_request_duration_seconds_bucket{namespace="capstone"}[5m])))

# Latency SLI: share of requests under 300 ms
sum(rate(http_server_request_duration_seconds_bucket{service_name="spring-api", le="0.3"}[5m]))
/ sum(rate(http_server_request_duration_seconds_count{service_name="spring-api"}[5m]))

# Saturation: CPU vs request, memory vs limit
sum by (pod) (rate(container_cpu_usage_seconds_total{namespace="capstone",container!=""}[5m])) / sum by (pod) (kube_pod_container_resource_requests{namespace="capstone",resource="cpu"})
max by (pod) (container_memory_working_set_bytes{namespace="capstone",container!=""}) / max by (pod) (kube_pod_container_resource_limits{namespace="capstone",resource="memory"})

# CPU throttling (a hidden latency source)
sum by (pod) (rate(container_cpu_cfs_throttled_periods_total{namespace="capstone"}[5m])) / sum by (pod) (rate(container_cpu_cfs_periods_total{namespace="capstone"}[5m]))

# Pod restarts in the last hour
increase(kube_pod_container_status_restarts_total{namespace="capstone"}[1h]) > 0

# OOMKilled containers
kube_pod_container_status_last_terminated_reason{namespace="capstone", reason="OOMKilled"} == 1

# Pending pods (scheduling / capacity)
sum by (namespace) (kube_pod_status_phase{phase="Pending"}) > 0

# HPA at max (capacity ceiling reached)
kube_horizontalpodautoscaler_status_current_replicas == kube_horizontalpodautoscaler_spec_max_replicas

# Deployment generation mismatch (rollout stuck)
kube_deployment_status_observed_generation != kube_deployment_metadata_generation

# Spring Boot specifics
hikaricp_connections_pending{namespace="capstone"} > 0
rate(jvm_gc_pause_seconds_sum{namespace="capstone"}[5m])
jvm_memory_used_bytes{area="heap"} / jvm_memory_max_bytes{area="heap"}

# FastAPI / Python specifics (OTel runtime metrics)
rate(process_cpu_seconds_total{service_name="fastapi-api"}[5m])
python_gc_collections_total

# Aurora (CloudWatch exporter / YACE)
aws_rds_database_connections_average{dbinstance_identifier=~"capstone-db.*"}
aws_rds_cpuutilization_average{dbinstance_identifier=~"capstone-db.*"}

# AI endpoints
histogram_quantile(0.95, sum by (le) (rate(gen_ai_server_time_to_first_token_seconds_bucket{namespace="ai"}[5m])))
vllm:gpu_cache_usage_perc{namespace="ai"}
sum(rate(gen_ai_client_token_usage_sum{service_name="incident-agent"}[5m])) by (gen_ai_token_type)

# Capacity forecast (30 days ahead from 14 days of history)
predict_linear(sum(rate(http_server_request_duration_seconds_count{service_name="spring-api"}[1h]))[14d:1h], 30*24*3600)

# Cardinality check before adding labels
topk(10, count by (__name__)({__name__=~".+"}))
```

## 4. LogQL and TraceQL for Request Tracing

```logql
# All logs for a request id across services
{namespace="capstone"} | json | request_id="req_01J9X2M7K3"

# Errors for one service, parsed
{namespace="capstone", app="fastapi-api"} | json | level="error" | line_format "{{.ts}} {{.msg}} trace={{.trace_id}}"

# Error rate from logs (when metrics are missing)
sum by (app) (rate({namespace="capstone"} | json | level="error" [5m]))

# Top error messages
topk(10, sum by (msg) (count_over_time({namespace="capstone"} | json | level="error" [15m])))

# Logs for a specific trace
{namespace="capstone"} | json | trace_id="4bf92f3577b34da6a3ce929d0e0e4736"

# Slow requests from access logs (duration field in ms)
{namespace="capstone", app="spring-api"} | json | duration_ms > 2000
```

```traceql
# Exact trace
{ trace:id = "4bf92f3577b34da6a3ce929d0e0e4736" }

# Failed, slow requests on a route
{ resource.service.name = "fastapi-api" && span.http.route = "/v1/chat" && status = error && duration > 2s }

# Traces where the DB span was slow
{ resource.service.name = "spring-api" } >> { span.db.system = "postgresql" && duration > 500ms }

# Traces touching Bedrock with throttling
{ span.gen_ai.system = "aws.bedrock" && span.http.response.status_code = 429 }

# Traces by request id attribute (if propagated as span attribute)
{ span.http.request.header.x_request_id = "req_01J9X2M7K3" }
```

logcli equivalents:

```bash
logcli query --since=1h --limit=200 '{namespace="capstone"} | json | request_id="req_01J9X2M7K3"'
logcli series '{namespace="capstone"}' --analyze-labels     # label cardinality
```

## 5. kubectl Triage Cheat Sheet

| Task | Long form | Short form |
|---|---|---|
| What changed? | `kubectl rollout history deployment/fastapi-api --namespace capstone` | `kubectl rollout history deploy/fastapi-api -n capstone` |
| Current image | `kubectl get deployment fastapi-api --namespace capstone --output jsonpath='{.spec.template.spec.containers[0].image}'` | `kubectl get deploy fastapi-api -n capstone -o jsonpath=...` |
| Pods with node and IP | `kubectl get pods --namespace capstone --selector app=fastapi-api --output wide` | `kubectl get po -n capstone -l app=fastapi-api -o wide` |
| Why is it not ready? | `kubectl describe pod <pod> --namespace capstone` (read Events, Conditions, Last State) | `kubectl describe po <pod> -n capstone` |
| Previous crash logs | `kubectl logs <pod> --namespace capstone --previous --container fastapi-api` | `kubectl logs <pod> -n capstone -p -c fastapi-api` |
| All pods' logs | `stern --namespace capstone --selector app=fastapi-api --since 10m` | `stern -n capstone -l app=fastapi-api -s 10m` |
| Warning events sorted | `kubectl get events --namespace capstone --field-selector type=Warning --sort-by=.lastTimestamp` | same |
| Resource usage | `kubectl top pods --namespace capstone --containers` | `kubectl top po -n capstone --containers` |
| Node pressure | `kubectl describe node <node> \| grep -A5 Conditions` | same |
| Capacity vs requests | `kubectl resource-capacity --util --pods --namespace capstone` | same |
| Ephemeral debug | `kubectl debug --namespace capstone -it <pod> --image ghcr.io/example-org/support-tools:1.0.0 --target fastapi-api --profile general` | same |
| Node debug | `kubectl debug node/<node> -it --image ghcr.io/example-org/support-tools:1.0.0 --profile sysadmin` | same |
| Endpoints behind a Service | `kubectl get endpointslices --namespace capstone --selector kubernetes.io/service-name=fastapi-api` | `kubectl get endpointslice -n capstone -l kubernetes.io/service-name=fastapi-api` |
| Test service from inside | `kubectl run curl --rm -it --image curlimages/curl --restart Never -- curl -s http://fastapi-api.capstone.svc:8000/health` | same |
| DNS check | `kubectl run dns --rm -it --image busybox:1.36 --restart Never -- nslookup fastapi-api.capstone.svc.cluster.local` | same |
| Owner tree | `kubectl tree deployment fastapi-api --namespace capstone` | `kubectl tree deploy fastapi-api -n capstone` |
| Everything in namespace | `kubectl get-all --namespace capstone` | `kubectl get-all -n capstone` |
| NetworkPolicy affecting pod | `kubectl get networkpolicy --namespace capstone --output yaml` | `kubectl get netpol -n capstone -o yaml` |
| Who can do what | `kubectl auth can-i delete pods --namespace capstone --as alice@example.com` | same |
| Recent audit events (EKS -> CloudWatch) | `aws logs filter-log-events --log-group-name /aws/eks/capstone-eks/cluster --filter-pattern '{ $.verb = "delete" && $.objectRef.namespace = "capstone" }' --start-time $(date -d '1 hour ago' +%s000)` | same |

## 6. Release and Rollback Commands

```bash
# Deployment
kubectl rollout status deployment/fastapi-api -n capstone
kubectl rollout undo deployment/fastapi-api -n capstone                   # previous revision
kubectl rollout undo deployment/fastapi-api -n capstone --to-revision=12  # specific
kubectl rollout pause deployment/fastapi-api -n capstone                  # stop a bad rollout mid-way
kubectl rollout restart deployment/fastapi-api -n capstone                # new pods, same spec (config reload)

# Argo Rollouts (kubectl plugin: kubectl krew install argo-rollouts... or download from releases)
kubectl argo rollouts get rollout fastapi-api -n capstone --watch   # live canary status
kubectl argo rollouts promote fastapi-api -n capstone               # advance a paused step
kubectl argo rollouts promote fastapi-api -n capstone --full        # skip remaining steps
kubectl argo rollouts abort fastapi-api -n capstone                 # send all traffic back to stable
kubectl argo rollouts undo fastapi-api -n capstone                  # roll back to previous stable
kubectl argo rollouts retry rollout fastapi-api -n capstone         # retry after abort

# Argo CD
argocd app get capstone-fastapi-api
argocd app history capstone-fastapi-api
argocd app rollback capstone-fastapi-api 14          # history ID; app becomes OutOfSync vs Git
argocd app sync capstone-fastapi-api --prune         # re-sync to Git after a revert commit
argocd app set capstone-fastapi-api --sync-policy none   # disable auto-sync during an incident

# Helm (if used directly)
helm history fastapi-api -n capstone
helm rollback fastapi-api 7 -n capstone --wait

# Git revert is the preferred GitOps rollback
git revert --no-edit <BAD_COMMIT_SHA> && git push origin main
```

## 7. k6 Quick Reference

```bash
k6 run smoke.js                                   # run locally
k6 run --vus 50 --duration 5m load.js             # override VUs/duration
k6 run --env BASE_URL=https://staging.example.com load.js
k6 run --out json=results.json load.js            # raw results
k6 run --out experimental-prometheus-rw load.js   # remote-write to Prometheus (set K6_PROMETHEUS_RW_SERVER_URL)
k6 run --summary-export=summary.json load.js      # thresholds result for CI
k6 cloud load.js                                  # Grafana Cloud k6 (optional)
```

Test types: **smoke** (1-5 VUs, 1 min, correctness), **load** (expected peak, 10-30 min), **stress** (ramp past peak until failure, find the ceiling), **soak** (expected load, 2-8 h, leaks), **spike** (sudden 10x, autoscaling behaviour). A full script with stages and thresholds is in [05-production-patterns.md](05-production-patterns.md#5-capacity-planning-and-load-testing-with-k6).

## 8. Priority Matrix

| | Urgency: high (SLO breaching / growing) | Urgency: medium (degraded, stable) | Urgency: low (workaround, cosmetic) |
|---|---|---|---|
| **Impact: all users / data loss / security** | P1 | P1 | P2 |
| **Impact: many users / one feature** | P1 | P2 | P3 |
| **Impact: one customer / internal** | P2 | P3 | P4 |

## 9. Templates

### Change record (normal change)

```markdown
# CHG-0231: Upgrade Aurora PostgreSQL 15 -> 16 (capstone-db)
Owner: team-platform  | Reviewer (affected team): team-payments | Window: 2026-11-04 02:00-04:00 UTC
Risk: medium (engine upgrade; ~60 s writer unavailability) | Blast radius: spring-api, fastapi-api writes
Pre-checks: snapshot taken; blue/green deployment created; error budget > 50%; no freeze active
Steps: 1) switch over blue/green 2) verify connections 3) run smoke k6 4) monitor 30 min
Rollback: switch back to blue (available 24 h) ; restore from snapshot (RTO 45 min)
Communication: #ops-changes 24 h before; status page maintenance notice
Post-change: verify golden signals; close record with evidence links
```

### Incident ticket fields (minimum)

`id`, `priority`, `service`, `started_at`, `detected_at`, `acknowledged_at`, `mitigated_at`, `resolved_at`, `detected_by` (alert / customer / synthetic), `related_change`, `root_cause_category`, `runbook_used`, `automation_candidate` (bool), `postmortem_link`.

### Runbook and handover templates

See [01-getting-started.md](01-getting-started.md) sections 5 and 7.

## 10. Checklists

### First five minutes of a page (L2)

- [ ] Acknowledge; join the incident channel; state you are IC if P1.
- [ ] Open the golden-signal dashboard for the service; identify the incident shape (overload / change / dependency / upstream).
- [ ] Ask "what changed?": rollout history, Argo CD history, change calendar, infrastructure deploys, feature flags.
- [ ] If a change correlates: roll back first, diagnose later.
- [ ] If overload: scale out, enable load shedding/rate limits.
- [ ] If dependency: check its status, fail over or degrade gracefully (circuit breaker, cached responses).
- [ ] Post a status update every 15 min (P1) with impact, actions, next update time.

### Before closing a ticket

- [ ] Root cause or durable workaround recorded.
- [ ] Runbook updated (or created).
- [ ] Toil/automation ticket filed if any manual repeatable step.
- [ ] Postmortem scheduled if SLO impact > 10% of monthly budget or customer-visible P1.
- [ ] Metrics fields complete (MTTA/MTTR computable).

### Operational readiness review (summary; full list in 05)

- [ ] Golden-signal dashboard and SLOs defined and alerting on burn rate.
- [ ] Runbook for every alert; owner and escalation defined.
- [ ] Rollback tested; progressive delivery configured.
- [ ] Load test establishes per-replica ceiling; HPA/Karpenter limits set with headroom.
- [ ] Backups and restore tested; RPO/RTO documented.
- [ ] Security gates passing; no static credentials; least-privilege IAM.
- [ ] Cost tags present; budget alert configured.
- [ ] Docs in TechDocs; on-call rota live.

## 11. Useful Environment and Aliases

```bash
# ~/.bashrc or ~/.zshrc for on-call
export KUBECONFIG=~/.kube/capstone-eks.yaml
alias k=kubectl
alias kc='kubectl ctx'
alias kn='kubectl ns'
alias kgp='kubectl get pods -o wide'
alias kev='kubectl get events --sort-by=.lastTimestamp --field-selector type=Warning'
alias triage='~/capstone-platform/tools/triage.sh'
export PROM=http://localhost:9090   # after: kubectl port-forward -n monitoring svc/prometheus-operated 9090
export LOKI_ADDR=http://localhost:3100   # after: kubectl port-forward -n monitoring svc/loki-gateway 3100:80
```

Continue with [04-hands-on-labs.md](04-hands-on-labs.md).
