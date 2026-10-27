# Production Support: Getting Started — Adopting the Practice and Setting Up the Tooling

| | |
|---|---|
| **Cluster** | 12 Enterprise Ops |
| **Role** | Bridge (extends 11-sre) |
| **Prerequisites** | A kind/k3d or EKS cluster running `fastapi-api` and `spring-api` in namespace `capstone`; kube-prometheus-stack, Loki and Tempo in `monitoring`; `kubectl` 1.31+, Helm 3.15+ |
| **Target version** | Kubernetes 1.31+, krew 0.4.4+, k9s 0.32+, stern 1.30+, k6 0.5x+ |
| **Capstone phase** | Phase 5 (Months 17-20) |

> This file gets a team from "developers get paged randomly" to a working tiered support model in a week, and gets your workstation ready with the triage toolkit used in every later file: kubectl plugins via krew, `k9s`, `stern`, ephemeral debug containers, k6, and an Alertmanager webhook to a ticketing system. By the end you will have an on-call rota definition, a support SLA matrix, a runbook skeleton for every existing alert, and the tooling installed and verified against the capstone services.

---

## 1. Adopting the Practice: A One-Week Rollout Plan

| Day | Activity | Output |
|---|---|---|
| 1 | Inventory services and owners. Export the Backstage catalogue (`catalog-info.yaml` `spec.owner`) and reconcile with Alertmanager routes | `docs/support/service-ownership.md` table: service, owner team, tier, SLO, escalation contact (team alias, never a person) |
| 2 | Define tiers and the SLA matrix (section 2). Decide which alerts page (burn-rate) vs ticket (everything else) | `docs/support/support-model.md` |
| 3 | Install the toolkit (section 3) on every on-call engineer's machine; create a `support-tools` container image for break-glass pods | Verified toolchain, image `ghcr.io/example-org/support-tools:1.0.0` |
| 4 | Create a runbook skeleton for every alert with no `runbook_url` annotation (section 5) | One Markdown file per alert under `docs/runbooks/` |
| 5 | Wire Alertmanager to the ticketing system for non-paging alerts (section 6), and schedule the first shift handover | Tickets auto-created; handover template adopted |

The deliberately small scope matters: teams that try to introduce ITIL-style change boards, a new ticketing tool and a new on-call tool simultaneously stall. Start with ownership, tiers, tooling and runbooks.

## 2. Minimum Viable Support Model

### Tiers

| Tier | Who | Scope | Access | Target |
|---|---|---|---|---|
| L1 | Runbook automation, AI incident agent (Phase 6), service desk | Known problems with a runbook; information gathering; triage and routing | Read-only (metrics, logs, traces, `kubectl get/describe/logs`) | Acknowledge < 5 min, resolve or escalate < 15 min |
| L2 | On-call platform/SRE engineer (primary + secondary) | Mitigation: rollback, scale, restart, failover, feature flag off | Write access via JIT elevation, audited | Mitigate P1 < 30 min |
| L3 | Owning product team engineer | Root cause fix, data repair, hotfix release | Code and deploy via CI/CD; prod access only with L2 | Fix or durable workaround within SLA |

### Support SLA matrix (response / resolution)

| Priority | Definition | Response | Resolution target | Example |
|---|---|---|---|---|
| P1 | SLO-breaching outage or data loss for many users; security incident | 15 min, 24x7 | 4 h | `api.example.com` returning 5xx > 5% |
| P2 | Degradation; single feature down; error budget burning > 3x | 30 min, 24x7 | 8 h | p99 latency 3x baseline on `/v1/chat` |
| P3 | Single customer impacted; workaround exists | 4 business hours | 3 business days | One tenant's exports failing |
| P4 | Cosmetic, question, request | 1 business day | 10 business days | "How do I rotate my API key?" |

The SLA is a *contract with the business about response*; the SLO is a *target for the service's behaviour*. See [02-core-concepts.md](02-core-concepts.md#8-support-slas-vs-slos) for how they interact.

## 3. Installing the Triage Toolkit

### kubectl plugins via krew

```bash
# Install krew (Linux/macOS; on Windows use the krew.exe release or WSL)
(
  set -x; cd "$(mktemp -d)" &&
  OS="$(uname | tr '[:upper:]' '[:lower:]')" &&
  ARCH="$(uname -m | sed -e 's/x86_64/amd64/' -e 's/\(arm\)\(64\)\?.*/\1\2/' -e 's/aarch64$/arm64/')" &&
  KREW="krew-${OS}_${ARCH}" &&
  curl -fsSLO "https://github.com/kubernetes-sigs/krew/releases/latest/download/${KREW}.tar.gz" &&
  tar zxvf "${KREW}.tar.gz" &&
  ./"${KREW}" install krew
)
export PATH="${KREW_ROOT:-$HOME/.krew}/bin:$PATH"

# Plugins used throughout this folder
kubectl krew install ctx ns          # fast context / namespace switching
kubectl krew install neat            # strip managed fields from YAML
kubectl krew install tree            # show owner-reference trees (Deployment -> RS -> Pods)
kubectl krew install resource-capacity   # requests vs allocatable per node
kubectl krew install view-secret     # decode a secret key without base64 gymnastics
kubectl krew install sniff           # tcpdump a pod via ephemeral sidecar
kubectl krew install get-all         # list every resource in a namespace, including CRDs
kubectl krew install rollout-restart # (optional) restart by label selector
```

Verify:

```bash
kubectl resource-capacity --namespace capstone --pods --util
```

Expected output (abridged):

```text
NODE                  NAMESPACE   POD                           CPU REQUESTS   CPU LIMITS   CPU UTIL   MEMORY REQUESTS   MEMORY LIMITS   MEMORY UTIL
ip-10-0-1-23.ec2...   capstone    fastapi-api-7c9d6f8b5-x2k9q   250m (6%)      1000m (25%)  41m (1%)   512Mi (7%)        1Gi (14%)       188Mi (2%)
```

### k9s

```bash
# Linux
curl -fsSL https://github.com/derailed/k9s/releases/latest/download/k9s_Linux_amd64.tar.gz | tar xz -C /usr/local/bin k9s
# macOS
brew install derailed/k9s/k9s
# Windows
winget install k9s
k9s --namespace capstone --readonly   # --readonly disables delete/edit: the right default for L1
```

Useful k9s keys for triage: `:pods`, `:events`, `l` (logs), `d` (describe), `y` (YAML), `:pulses` (cluster-wide health), `Shift-F` (port-forward), `/` (filter), `:xray deploy capstone` (dependency tree).

### stern (multi-pod log tailing)

```bash
# Linux/macOS
brew install stern   # or download from https://github.com/stern/stern/releases
# Tail every pod of a Deployment, JSON-decode, highlight errors
stern --namespace capstone --selector app=fastapi-api --since 15m --output json | jq -r 'select(.level=="error")'
# Follow a request id across both services
stern --namespace capstone 'fastapi-api|spring-api' --include 'request_id=req_7f3a' --since 1h
```

### kubectl debug (ephemeral containers)

Distroless images have no shell. Ephemeral containers (GA since Kubernetes 1.25) attach a debugging container to a running Pod without restarting it.

```bash
kubectl debug --namespace capstone -it fastapi-api-7c9d6f8b5-x2k9q \
  --image=ghcr.io/example-org/support-tools:1.0.0 \
  --target=fastapi-api \
  --profile=general -- bash
```

Inside the debug container you share the target container's process namespace (`--target`), so `ps`, `ss -tlnp`, `cat /proc/1/environ | tr '\0' '\n'` (careful: may contain secrets, do not paste into tickets) and `curl localhost:8000/health` all work. For node-level debugging:

```bash
kubectl debug node/ip-10-0-1-23.ec2.internal -it --image=ghcr.io/example-org/support-tools:1.0.0 --profile=sysadmin
# host filesystem is mounted at /host
chroot /host journalctl -u kubelet --since "15 min ago"
```

A minimal support-tools image (kept small, non-root, rebuilt weekly by CI so Trivy findings stay fresh):

```dockerfile
FROM debian:bookworm-slim
RUN apt-get update && apt-get install -y --no-install-recommends \
      curl jq dnsutils iproute2 procps strace tcpdump postgresql-client ca-certificates \
    && rm -rf /var/lib/apt/lists/*
RUN useradd --uid 10001 --create-home support
USER 10001
ENTRYPOINT ["bash"]
```

### k6

```bash
# Linux (Debian/Ubuntu)
sudo gpg -k && sudo gpg --no-default-keyring --keyring /usr/share/keyrings/k6-archive-keyring.gpg \
  --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys C5AD17C747E3415A3642D57D77C6C491D6AC1D69
echo "deb [signed-by=/usr/share/keyrings/k6-archive-keyring.gpg] https://dl.k6.io/deb stable main" | sudo tee /etc/apt/sources.list.d/k6.list
sudo apt-get update && sudo apt-get install k6
# macOS
brew install k6
# Windows
winget install k6 --source winget
# Container (CI)
docker run --rm -i grafana/k6:latest run - < smoke.js
k6 version
```

## 4. Confirming the Golden Signals Are Available

Before writing runbooks, confirm that Prometheus actually has the four signals for both capstone services. Open Grafana Explore (or `curl` the Prometheus API) and run:

```promql
# Traffic: requests per second by service
sum by (service_name) (rate(http_server_request_duration_seconds_count{namespace="capstone"}[5m]))
```

```promql
# Errors: 5xx ratio by service
sum by (service_name) (rate(http_server_request_duration_seconds_count{namespace="capstone", http_response_status_code=~"5.."}[5m]))
/
sum by (service_name) (rate(http_server_request_duration_seconds_count{namespace="capstone"}[5m]))
```

```promql
# Latency: p99 by service
histogram_quantile(0.99, sum by (service_name, le) (rate(http_server_request_duration_seconds_bucket{namespace="capstone"}[5m])))
```

```promql
# Saturation: CPU usage vs requests by pod
sum by (pod) (rate(container_cpu_usage_seconds_total{namespace="capstone", container!=""}[5m]))
/
sum by (pod) (kube_pod_container_resource_requests{namespace="capstone", resource="cpu"})
```

If the OTel metrics are missing, the services are not instrumented or the OTel Collector's Prometheus exporter is not scraped. Fix that first (see [09-observability/opentelemetry](../../09-observability/opentelemetry/README.md)); support without signals is guesswork.

Via the API, for scripts:

```bash
PROM=http://prometheus-operated.monitoring.svc:9090   # or port-forward to localhost:9090
curl -sG "$PROM/api/v1/query" --data-urlencode \
  'query=sum by (service_name) (rate(http_server_request_duration_seconds_count{namespace="capstone"}[5m]))' | jq '.data.result'
```

## 5. Runbook Skeleton for Every Alert

List alerts missing a runbook annotation:

```bash
kubectl get prometheusrules --all-namespaces -o json \
  | jq -r '.items[].spec.groups[].rules[] | select(.alert) | select(.annotations.runbook_url == null) | .alert' | sort -u
```

Generate a skeleton for each (store under `docs/runbooks/<AlertName>.md`):

```markdown
# Runbook: <AlertName>

**Severity:** page | ticket    **Service:** fastapi-api    **Owner:** team-platform
**SLO impacted:** availability 99.9%    **Last reviewed:** YYYY-MM-DD

## What this alert means
One paragraph in plain language. What the user sees.

## Quick triage (L1, read-only, < 5 minutes)
1. Golden-signal dashboard: <Grafana link with service variable pre-set>
2. What changed? `kubectl rollout history deployment/fastapi-api -n capstone`; Argo CD app history; change calendar.
3. Errors by route: <PromQL>
4. Sample failing trace: <Tempo TraceQL>

## Mitigation (L2)
- Rollback: `kubectl argo rollouts undo fastapi-api -n capstone`
- Scale: `kubectl scale deploy/fastapi-api -n capstone --replicas=6`
- Feature flag: disable `chat_streaming` in <flag tool>

## Escalation
L3: team-ai-apps (Slack `#ai-apps-oncall`). Page via PagerDuty service "fastapi-api".

## Known causes and fixes
| Cause | Signature | Fix |

## After the incident
- Postmortem required if SLO impact > 10% of monthly budget.
- Update this runbook; file toil/automation ticket if any step was manual and repeatable.
```

Add the annotation back to the alert so the page links to the runbook:

```yaml
annotations:
  runbook_url: https://backstage.example.com/docs/default/component/fastapi-api/runbooks/HighErrorRate/
```

## 6. Alertmanager to Ticketing (First Integration)

Route everything that is not a page into Jira Service Management (or ServiceNow) so that nothing is lost and ticket volume becomes a metric. Alertmanager has no native Jira receiver in older versions; use the `webhook_configs` receiver pointing at a small adapter (or the `jiralert` project, or Alertmanager 0.27+ `jira_configs`).

```yaml
# alertmanager.yaml (excerpt)
route:
  receiver: jira-tickets
  group_by: [alertname, namespace, service_name]
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 12h
  routes:
    - matchers: [severity="page"]
      receiver: pagerduty-l2
      continue: true
    - matchers: [severity="ticket"]
      receiver: jira-tickets
receivers:
  - name: pagerduty-l2
    pagerduty_configs:
      - routing_key_file: /etc/alertmanager/secrets/pagerduty-routing-key
        severity: critical
  - name: jira-tickets
    jira_configs:
      - api_url: https://example.atlassian.net/rest/api/2
        project: OPS
        issue_type: Incident
        summary: '{{ .GroupLabels.alertname }} on {{ .GroupLabels.service_name }}'
        description: '{{ range .Alerts }}{{ .Annotations.description }}\nRunbook: {{ .Annotations.runbook_url }}\n{{ end }}'
        priority: '{{ if eq .CommonLabels.severity "ticket" }}Medium{{ else }}High{{ end }}'
        labels: ['alertmanager', '{{ .GroupLabels.namespace }}']
        reopen_duration: 24h
```

Store the Jira token in a Kubernetes Secret delivered by External Secrets Operator (never in the ConfigMap). Verify with a test alert:

```bash
curl -XPOST http://alertmanager-operated.monitoring.svc:9093/api/v2/alerts -H 'Content-Type: application/json' -d '[{
  "labels": {"alertname":"SupportIntegrationTest","severity":"ticket","namespace":"capstone","service_name":"fastapi-api"},
  "annotations": {"description":"Test ticket from Alertmanager","runbook_url":"https://example.com/runbooks/test"}
}]'
```

Expected: a new OPS ticket within `group_wait` (30 s). Delete the test alert by waiting for it to resolve or by posting with `endsAt` in the past.

## 7. Shift Handover: Minimum Template

Adopt this on day 5 and keep it in `docs/support/handover/YYYY-MM-DD.md` (or the on-call tool's notes):

```markdown
# Shift handover — 2026-10-27 09:00 UTC — from <team alias> to <team alias>

## Open incidents
- INC-4821 (P2): spring-api p99 latency since 02:10 UTC; mitigated by scaling 3->6; RCA pending. Next action: L3 team-payments to review connection pool at 10:00.

## Changes in the last 24 h / planned next 24 h
- Deployed fastapi-api v2.14.0 (canary completed 07:40). Freeze window starts Friday 17:00 UTC.

## Alerts that fired (count, actioned?)
- HighErrorRate fastapi-api x2 (auto-resolved, runbook step 3 confirmed Bedrock throttling, ticket OPS-1193).

## Toil observed
- Manually restarted stuck Argo sync twice -> automation ticket PLAT-402.

## Risks / watch items
- Error budget fastapi-api: 61% remaining. Aurora storage 78%.
```

## 8. Verification Checklist

- [ ] Every service in `capstone` has an owner team and escalation alias in Backstage.
- [ ] SLA matrix published and linked from the on-call tool.
- [ ] `kubectl krew list` shows the plugins above; `k9s --readonly` starts; `stern` tails both services; `kubectl debug` attaches to a distroless pod.
- [ ] Golden-signal PromQL from section 4 returns data for both services.
- [ ] Zero alerts without `runbook_url`.
- [ ] Alertmanager test alert created a ticket.
- [ ] First shift handover written.

Continue with [02-core-concepts.md](02-core-concepts.md).
