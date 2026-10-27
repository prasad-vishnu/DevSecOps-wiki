# Production Support: Interview Questions

| | |
|---|---|
| **Cluster** | 12 Enterprise Ops |
| **Role** | Bridge (extends 11-sre) |
| **Prerequisites** | Whole folder |
| **Target version** | Kubernetes 1.31+, Argo Rollouts 1.7+, k6 0.5x+ |
| **Capstone phase** | Phase 5 (Months 17-20) |

> Thirty questions from junior to principal level with model answers. Junior questions test vocabulary; mid-level questions test the ability to run a shift; senior questions test design of the support system; principal questions test organisational trade-offs and the AI/L1 frontier. Answer out loud with numbers from the capstone where possible.

---

## Junior

### Q1. What are the Four Golden Signals and why are they called "golden"? (Junior)

**Answer.** Latency, traffic, errors and saturation. They are "golden" because, for any user-facing service, monitoring these four lets you detect most problems and localise them: traffic drives saturation, saturation drives latency, latency beyond timeouts becomes errors. Everything else (logs, traces) is drill-down after these four tell you which service and which class of problem you have.

### Q2. What is the difference between L1, L2 and L3 support? (Junior)

**Answer.** L1 handles known problems with scripted or automated steps and routes the rest; L2 is the on-call engineer with production access who mitigates (rollback, scale, failover); L3 is the owning development team who fixes root causes. In a "you build it, you run it" model the owning team covers L2 and L3 for its own service, the platform team covers L2 for the platform, and L1 is increasingly automation.

### Q3. What is an SLA and how is it different from an SLO? (Junior)

**Answer.** An SLA is a contract, often with penalties, about service levels and response times (for example P1 response in 15 minutes, 99.5% availability). An SLO is an internal engineering target (99.9% availability) deliberately stricter than the SLA so that burn-rate alerts protect the contract with a buffer.

### Q4. A customer reports their request failed at 10:42. What do you need from them and what do you do with it? (Junior)

**Answer.** The `X-Request-ID` (returned in the response) or, failing that, their account id and the exact time. Search Loki for `request_id`, take the `trace_id` from the log line, open the trace in Tempo, find the span with `status = error`, and read its exception. Then answer with route, component, time and fix.

### Q5. What is toil? Give two examples and two non-examples. (Junior)

**Answer.** Toil is manual, repetitive, automatable, tactical work with no enduring value that scales with service growth. Examples: restarting a stuck sync every week; manually resizing a volume. Non-examples: writing a postmortem; designing an autoscaling policy; both require judgement and produce lasting value.

### Q6. Why should every alert have a `runbook_url` annotation? (Junior)

**Answer.** So the person (or agent) paged has the diagnostic and mitigation steps one click away. It also makes runbook coverage measurable: a query for alerts without the annotation becomes a readiness gate.

### Q7. What does `kubectl debug` do that `kubectl exec` cannot? (Junior)

**Answer.** It attaches an ephemeral container with its own image (containing a shell and tools) to a running Pod, sharing the process namespace of a target container, so you can debug distroless images that have no shell, without restarting the Pod. It can also create a privileged debugging pod on a node with the host filesystem mounted.

### Q8. What is a change freeze and when is it appropriate? (Junior)

**Answer.** A period during which normal changes are not allowed, typically around peak business events or when the error budget is exhausted. It is appropriate when the cost of an incident is unusually high; it should be encoded as policy (for example a Kyverno rule) and allow emergency changes through a documented process.

## Mid-level

### Q9. Explain the USE and RED methods and how they relate to the golden signals. (Mid-level)

**Answer.** RED (Rate, Errors, Duration) describes a request-serving service from its callers' viewpoint. USE (Utilisation, Saturation, Errors) describes a resource (CPU, memory, pool, GPU) and finds bottlenecks. The golden signals are RED plus saturation, which is why they cover both. In practice a service dashboard is RED on top and USE below, and triage walks top-down.

### Q10. Write the PromQL for the error ratio of `fastapi-api` and explain one pitfall. (Mid-level)

**Answer.**

```promql
sum(rate(http_server_request_duration_seconds_count{service_name="fastapi-api",http_response_status_code=~"5.."}[5m]))
/ clamp_min(sum(rate(http_server_request_duration_seconds_count{service_name="fastapi-api"}[5m])), 1e-9)
```

Pitfall: at zero traffic the denominator is 0 and the result is `NaN`, so dashboards and `AnalysisTemplate` conditions break; `clamp_min` (or `or vector(0)`) avoids this. A second pitfall is computing latency over all requests, which hides timeouts behind fast failures.

### Q11. Your golden-signal dashboard shows traffic flat, saturation dropping, errors up, latency at the client timeout. What is happening and what is your first action? (Mid-level)

**Answer.** Dependency failure: the service is waiting on something (database, Bedrock, downstream API) that is not responding, so its own resources idle while requests time out. First action: check the dependency's status and the recent changes to it; mitigate by failing over (reader endpoint, alternate region/model), opening a circuit breaker, or serving degraded responses, rather than scaling the service (which would make it worse).

### Q12. Describe the expand/contract pattern and why support teams care about it. (Mid-level)

**Answer.** Schema changes are split into backward-compatible steps: add the new column (expand), dual-write, backfill, switch reads, then drop the old column (contract) only when no running code uses it. Each application release is compatible with the previous schema, so any single release can be rolled back. Support cares because a destructive migration makes rollback impossible, turning a five-minute mitigation into a multi-hour restore.

### Q13. How does Argo Rollouts decide to roll back automatically, and what happens when Prometheus returns no data? (Mid-level)

**Answer.** An `AnalysisTemplate` runs queries at intervals during canary steps; `successCondition` (for example `result[0] < 0.005`) must hold or the analysis fails and the rollout aborts, shifting traffic to the stable ReplicaSet. If the query returns an empty vector, `result[0]` is undefined and the measurement is an error or inconclusive, which can stall the rollout; use `len(result) == 0 || result[0] < 0.005`, set an `initialDelay`, and make sure canary pods carry the `rollouts_pod_template_hash` label on their metrics.

### Q14. What is the difference between MTTA and MTTR and how do you measure them? (Mid-level)

**Answer.** MTTA is the mean time from alert to acknowledgement (did someone start?); MTTR here is the mean time from incident start to mitigation (is the user impact over?). Both come from timestamps on tickets or the on-call tool; auto-creating a ticket for every alert is what makes them measurable. Watch for gaming: closing tickets early lowers MTTR without helping users.

### Q15. How would you integrate Alertmanager with Jira Service Management, and what grouping settings would you use? (Mid-level)

**Answer.** Route `severity=ticket` alerts to a `jira_configs` receiver (Alertmanager 0.27+) or a webhook adapter such as jiralert, with the token in a Secret. Group by `alertname, namespace, service_name` so one ticket covers all pods of a service; `group_wait` 30 s, `group_interval` 5 m, `repeat_interval` 12 h, `send_resolved: true` with an automation rule to transition the issue on resolve, and `reopen_duration` so flapping alerts reopen rather than duplicate.

### Q16. What belongs in a shift handover? (Mid-level)

**Answer.** Open incidents with next actions and owners, changes in the last and next 24 hours, alerts that fired and whether action was taken, toil observed (automation candidates), risks and watch items such as error budget remaining and storage headroom. It transfers state, not just the pager.

### Q17. Explain the k6 "arrival-rate" executors and why they matter for finding a capacity ceiling. (Mid-level)

**Answer.** `constant-arrival-rate` and `ramping-arrival-rate` are open-model executors: k6 starts iterations at a fixed rate regardless of how long each takes, allocating VUs as needed. Closed-model executors (`constant-vus`) slow down when the service slows down, hiding the ceiling. To measure the request rate at which p99 crosses the SLO you need the open model; `dropped_iterations` then tells you the generator ran out of VUs, which itself indicates the service is saturated.

### Q18. What are standard, normal and emergency changes? Give a capstone example of each. (Mid-level)

**Answer.** Standard: pre-approved, low-risk, automated, such as a `fastapi-api` deploy through the pipeline with canary analysis. Normal: assessed and scheduled, such as an Aurora major version upgrade or a NetworkPolicy change affecting multiple teams. Emergency: executed under incident command and reviewed afterwards, such as scaling a NodePool limit during an overload incident.

## Senior

### Q19. Design a tiered support model for a platform team supporting eight product teams. (Senior)

**Answer.** Product teams own L2/L3 for their services; the platform team owns L2 for cluster, ingress, database platform, observability and CI/CD, and acts as advisor on application incidents. L1 is runbook automation triggered by Alertmanager webhooks plus, later, an AI agent with read-only tools. Alerts carry a `team` label used for routing; every alert has a runbook; an ORR gates production; toil is measured per team with a 50% ceiling; a weekly operations review looks at pages per shift, MTTA/MTTR, change failure rate and toil. The platform team publishes golden paths (SLO templates, dashboards, Rollout templates) so each product team inherits the model rather than reinventing it.

### Q20. How do you cut toil in a team that is at 70% operational load? (Senior)

**Answer.** First stop the bleeding: freeze new on-call responsibilities, delete or demote non-actionable alerts, and get leadership to accept a reliability sprint. Measure toil by category from shift reports, score the automation backlog by hours saved divided by effort, and ship the top items (often: auto-remediation Jobs for the top three alerts, self-service for the most common tickets, auto-rollback via progressive delivery). Change the ORR so new services cannot add manual steps. Report toil hours eliminated monthly until the team is under 50%.

### Q21. A stakeholder insists on a weekly CAB for all production changes. How do you respond? (Senior)

**Answer.** Acknowledge the goal (stability) and present the evidence: DORA research shows external approval boards correlate with lower stability and throughput, because batching increases blast radius and delays feedback. Propose a risk-based model: standard changes go through automated gates (tests, security scans, policy, canary analysis) with peer review; a CAB reviews only high-blast-radius normal changes (IAM, network, DNS, data engine upgrades). Offer metrics to prove it: change failure rate, time to restore and lead time reported monthly, with a rollback to the old process if stability worsens.

### Q22. How do you implement production access controls that satisfy both an auditor and an on-call engineer at 03:00? (Senior)

**Answer.** Read-only access by default through SSO groups mapped to the Kubernetes `view` role and a read-only AWS permission set. Just-in-time elevation (IAM Identity Center time-bound assignment, Entra PIM, or Teleport access requests) approved by a second engineer via Slack in under two minutes, time-boxed to the incident and tied to a ticket. Break-glass credentials under dual control for IdP outages, with an alert and mandatory postmortem on use. Kubernetes audit logs at `RequestResponse` for mutations and `exec` in production namespaces, plus CloudTrail, shipped to a separate log-archive account. The auditor gets attribution, approval evidence and tamper-evident logs; the engineer gets a fast path.

### Q23. How do you size the HPA for a service given a load-test ceiling of 450 rps per replica and a forecast peak of 1,800 rps across three AZs? (Senior)

**Answer.** Apply headroom for losing one of three AZs (x 1.5) and a steady-state utilisation target of 70%: ceil(1,800 x 1.5 / (450 x 0.7)) = ceil(8.57) = 9 replicas at peak, spread with topology constraints so each AZ holds 3. Set HPA min to the off-peak need (3), max above the peak requirement (12) to absorb forecast error, and verify with a 10-minute load test at 1,800 rps. Re-check Karpenter NodePool limits, database connection limits (9 replicas x pool size) and any downstream quotas.

### Q24. What makes a runbook "automatable", and how would you automate the top one safely? (Senior)

**Answer.** Deterministic read-only steps (status, metrics, logs), clearly specified decision criteria, and a mitigation that is idempotent and reversible (rollback, scale, restart). Automate it as a script in a versioned support-tools image, run as a Kubernetes Job triggered by the Alertmanager webhook under a ServiceAccount with `view` RBAC, posting output to the ticket. For the mitigation step, start with "propose, human approves"; once accuracy over a quarter is high, allow the mitigation to execute automatically for that alert only, with an audit trail and a kill switch.

### Q25. How do you measure change failure rate accurately in a GitOps + progressive-delivery setup? (Senior)

**Answer.** Define a "deploy" as an Argo CD sync to production that resulted in a new Rollout revision. Define "failure" as any deploy that (a) was aborted by analysis, (b) was rolled back within 24 hours, (c) is linked to an incident ticket, or (d) required a hotfix within 24 hours. Collect (a) and (b) from Rollout events and Argo CD history, (c) and (d) from the ticket's `related_change` field. Report per service and per team; publish the definition so it cannot be gamed by relabelling incidents.

## Principal

### Q26. Where should the boundary between an AI incident agent and humans sit today, and how would you move it over time? (Principal)

**Answer.** Today the agent is L1: triage, evidence gathering, hypothesis and proposed mitigation using read-only tools, with every mutation behind a human approval. The boundary is enforced technically (RBAC `view`, MCP tool allow-list, separate identity, no egress) rather than by prompt. It moves outward per alert type, based on evidence: once the agent's proposed mitigation matches the human's action in, say, 95% of cases over a quarter for a given alert, allow it to execute that specific mitigation via the existing automation (for example `rollouts abort`) with auto-escalation on uncertainty, a kill switch and continuous evals. Never grant generic write tools; expand a list of specific, idempotent, reversible actions.

### Q27. How do you present the business case for a production-support programme to a CFO? (Principal)

**Answer.** Translate operations into money and risk: incident cost (hours of outage x revenue per hour plus SLA credits), toil cost (hours x loaded rate), change-freeze opportunity cost, and attrition cost from burnout. Show the levers with numbers: auto-rollback prevents N incidents per quarter; automation removes M toil hours per month per team; faster JIT reduces MTTR by X minutes. Tie to the FinOps and governance programmes: toil hours and incident counts appear next to cloud spend and control coverage in one operations review. Commit to three KPIs (MTTR, change failure rate, toil %) and report quarterly.

### Q28. An enterprise acquisition brings a classic ITIL NOC with 40 L1 staff. How do you integrate it with an SRE-style organisation? (Principal)

**Answer.** Do not abolish it overnight; redirect it. Keep the NOC as the 24x7 L1 for *detection and coordination* while moving their scripted steps into automation they operate and improve; train them on golden-signal triage and give them read-only tooling (k9s read-only, dashboards, triage scripts). Route application alerts to owning teams with the NOC as coordinator and communicator (status page, stakeholder updates), which they are good at. Over 12-18 months, convert the strongest L1 staff into automation engineers and incident commanders; measure the share of alerts resolved without escalation and the escalation context quality. The failure mode to avoid is a NOC that pages engineers for everything without context.

### Q29. How do you design support for a multi-tenant AI service where one tenant's load can degrade others? (Principal)

**Answer.** Isolate by design: per-tenant rate limits and token budgets at the gateway, priority queues, and separate Bedrock inference profiles or vLLM deployments for premium tiers. Observe by tier: golden signals with a bounded `tenant_tier` label, per-tenant investigation via logs and traces, and cost per tenant via token accounting (FinOps). Support SLAs by tier map to the priority matrix. Capacity planning includes a noisy-neighbour scenario in k6. Runbooks include "shed tenant X" as a reversible mitigation with product sign-off rules. Postmortems look for cross-tenant blast radius as a category.

### Q30. What would make you reject a service at an operational readiness review, even if the team is under schedule pressure? (Principal)

**Answer.** Any mandatory item whose absence makes incidents unmanageable or unsafe: no SLOs or alerting (we would learn of outages from customers), no rollback path (destructive migration, no progressive delivery), static cloud credentials or missing workload identity, no tested backup/restore for stateful data, no runbooks for paging alerts, or manual steps in the deploy. I would offer a time-boxed, risk-accepted exception only for recommended items, signed by the service owner and the platform lead with an expiry, and I would make the ORR faster next time by providing golden-path templates so compliance is the default rather than an afterthought.

---

Back to [README.md](README.md) or continue the cluster with [../disaster-recovery/README.md](../disaster-recovery/README.md).
