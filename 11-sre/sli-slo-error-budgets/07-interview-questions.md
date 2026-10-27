# SLIs, SLOs and Error Budgets: Interview Questions

| | |
|---|---|
| **Cluster** | 11 SRE |
| **Role** | Anchor |
| **Prerequisites** | The rest of this folder |
| **Target version** | Prometheus 2.55+/3.x |
| **Capstone phase** | Phase 4 (Months 13-16) |

> Thirty questions from junior to principal. Model answers are deliberately concise; in an interview, expand each with a concrete example from the capstone (99.9% FastAPI availability SLO, 43.2 minutes, 14.4x burn rate).

---

## Junior

### Q1. Define SLI, SLO and SLA in one sentence each. (Junior)

**Answer.** An SLI is a measurement of service behaviour expressed as good events divided by valid events (for example the fraction of requests that returned non-5xx). An SLO is a target for that SLI over a window (99.9% over 30 days). An SLA is a contract that attaches consequences to missing a target, and it is always looser than the SLO so the SLO acts as the early warning.

### Q2. A service promises 99.9% availability over 30 days. How many minutes of downtime is that? (Junior)

**Answer.** 30 days = 43,200 minutes. The error budget is 1 - 0.999 = 0.001. 43,200 x 0.001 = 43.2 minutes. Equivalently 1,000 failed requests per million.

### Q3. Why is "average latency" a bad SLI? (Junior)

**Answer.** Averages hide the tail. A service with a 100 ms median and a 5-second 99th percentile can average 150 ms while 1% of users wait five seconds. An SLI should be a ratio such as "fraction of requests served under 300 ms", which is bounded, aggregates across pods and time, and captures the tail directly.

### Q4. What is an error budget? (Junior)

**Answer.** The amount of unreliability the SLO permits: 1 minus the target over the window. It is a budget because the team spends it deliberately on releases and experiments and stops spending when it runs out.

### Q5. What is the difference between a good event and a valid event? (Junior)

**Answer.** Valid events are the denominator: everything we have decided counts. Good events are the subset that met the condition. Health checks, load-test traffic, and requests rejected by a WAF are commonly excluded from valid events; 4xx responses are usually valid and good because the server did its job.

### Q6. Where would you measure availability for an HTTP service and why? (Junior)

**Answer.** At the load balancer or ingress by default. It sees every request including those the application never received: if all pods are down the app emits no metrics, but the ingress records 502s, so the SLI correctly drops. Application-side metrics add rich labels and are the right place for quality SLIs.

### Q7. What does a burn rate of 1.0 mean? (Junior)

**Answer.** The service is consuming its error budget at exactly the rate that would exhaust it at the end of the window. For a 99.9% SLO that is a sustained error ratio of 0.1%. A burn rate of 10 would exhaust a 30-day budget in 3 days.

---

## Mid-level

### Q8. Write PromQL for a latency SLI "fraction of requests under 300 ms" from a Prometheus histogram, and explain why it is better than `histogram_quantile`. (Mid-level)

**Answer.**

```promql
sum(rate(http_request_duration_seconds_bucket{service="fastapi",le="0.3"}[5m]))
/
sum(rate(http_request_duration_seconds_count{service="fastapi"}[5m]))
```

The `le="0.3"` bucket is cumulative and counts every request at or under 300 ms, so the ratio is exact as long as 0.3 is a real bucket boundary. `histogram_quantile(0.99, ...)` interpolates linearly within a bucket and returns a percentile, which cannot be averaged or aggregated and is not a good/valid ratio.

### Q9. Derive the 14.4x burn-rate number used for paging. (Mid-level)

**Answer.** The Workbook's fast-burn page fires when 2% of the 30-day budget is consumed in one hour. Burn rate = fraction consumed x period / window = 0.02 x 720 h / 1 h = 14.4. At a 99.9% SLO that corresponds to an error ratio of 14.4 x 0.001 = 1.44% sustained for an hour.

### Q10. Explain the difference between request-based and window-based SLIs and when you would choose each. (Mid-level)

**Answer.** Request-based counts good requests over valid requests; the budget is in failed requests and is weighted by traffic. Window-based counts good minutes over total minutes where a minute is good if a condition held; the budget is in minutes and ignores traffic volume. Request-based is the default because it is fair to users. Window-based suits batch pipelines and very low traffic services where a few requests would otherwise swing the ratio.

### Q11. Your SLI shows 99.95% but customers are complaining of errors. List three plausible causes. (Mid-level)

**Answer.** (1) Measuring in the application only, so a crashed deployment emits no errors. (2) The failing journey is not in the valid set (a new route, a WebSocket path, a different host). (3) Errors are returned with a 200 status and an error body, so the status-code SLI cannot see them; a correctness SLI is needed.

### Q12. Why should an SLO be stricter than the SLA? Give numbers. (Mid-level)

**Answer.** The SLO is the engineering trigger; the SLA is the legal consequence. With a 99.9% SLO and a 99.5% SLA, the 0.4% gap (172.8 minutes per 30 days) is the time between "we are burning budget and must act" and "we owe credits". If they were equal you would find out about a breach from the customer.

### Q13. What goes into an error budget policy? (Mid-level)

**Answer.** Thresholds of remaining budget with pre-agreed actions (above 50%: normal cadence and encouraged experiments; 25-50%: mandatory canary analysis; 0-25%: reliability and security changes only; below 0: feature freeze and reliability sprint), notification targets at each threshold, how dependencies' failures are treated, whether maintenance counts (it should), and a single named override authority with written exceptions.

### Q14. How do you handle a histogram whose buckets do not include your SLO threshold? (Mid-level)

**Answer.** Preferably change the instrumentation buckets (OTel `View` with explicit boundaries; Micrometer `distribution.slo`). If the library is not configurable (vLLM), move the threshold to the nearest real boundary and document it. Do not interpolate with `histogram_quantile`, because the SLI would become an estimate with bucket-width error.

### Q15. Compare Sloth and Pyrra. (Mid-level)

**Answer.** Both generate the Workbook multi-window multi-burn-rate rules from a short spec. Sloth is primarily a CLI generator that fits CI review (spec and generated rules both committed) and has an SLI plugin system; it has no UI and a single configurable period. Pyrra is an operator with a UI showing budgets and burn graphs, with native latency indicators and a 4-week default window. Choose Sloth when you want Git review of rules; Pyrra when you want visibility for non-PromQL users. Do not run both for the same SLO.

### Q16. What SLIs would you define for an LLM inference endpoint? (Mid-level)

**Answer.** TTFT latency (fraction of completions with first token under 800 ms, or the nearest real bucket), completion success ratio (finished with stop/length rather than error), and, at the agent level, grounded answer rate from a sampled online judge, tool-call success ratio, and a cost-per-task SLI (fraction of tasks under a cost ceiling). Throughput (tokens per second) and queue depth are dashboard signals, not SLIs.

---

## Senior

### Q17. A service depends serially on Postgres (99.95%) and vLLM (99.5%). The product wants a 99.9% SLO. What do you tell them? (Senior)

**Answer.** With no mitigation the serial bound is 0.9995 x 0.995 = 0.9945, below 99.9%, so the target is unachievable as architected. Options: make vLLM non-critical (serve the non-AI answer and count the miss against a quality SLI instead of availability), negotiate a 99.9% SLO from the vLLM team, or set the availability SLO honestly at 99.5%. I would recommend the first and show the dependency table in the SLO document.

### Q18. How would you implement SLOs as code for 60 services without 60 hand-written specs? (Senior)

**Answer.** A Sloth SLI plugin (or a Helm chart generating `PrometheusServiceLevel` objects) parameterised by service name and route exclusions, created automatically by the Backstage scaffolder so no service ships without an SLO. CI validates, generates, checks with `promtool`, runs unit tests, and enforces policy with Conftest (every page alert has a `runbook_url`). Argo CD reconciles the generated `PrometheusRule` objects. Dashboards are templated on the `slo`, `service`, `team` labels.

### Q19. Explain why `sum(rate(x[5m]))` is correct and `rate(sum(x)[5m])` is not for an SLI. (Senior)

**Answer.** `rate()` must be applied per series so it can detect and correct counter resets (a pod restart drops the counter to zero). Summing first merges series, hiding resets and producing spurious negative or huge rates. Aggregation always goes outside `rate()`. Also, `rate(sum(x)[5m])` is a syntax error in PromQL without a subquery.

### Q20. How do you compute budget remaining cheaply when a 30-day `increase()` on raw counters is too expensive? (Senior)

**Answer.** Aggregate to a single series in the 5-minute rule, then derive longer windows from it: `avg_over_time(slo:sli_error:ratio_rate5m[30d])` is cheap but is an average of ratios (biased when traffic varies). Alternatively, record good and total rates separately at 5 m (`slo:sli_good:rate5m`, `slo:sli_total:rate5m`) and compute `1 - sum_over_time(good)/sum_over_time(total)`, which is traffic-weighted and still cheap. Run the long-window group every 5 minutes, separately from the 30-second alerting group.

### Q21. The grounded-answer SLI for an agent is judged on a 5% sample at 2 answers per minute. What problems arise and how do you adapt? (Senior)

**Answer.** Low volume: a 5-minute window may contain zero or one judged answer, so the ratio is 0, 1, or NaN, and short-window burn alerts are meaningless. Adapt by widening the SLO window (90 days), alerting only on long windows (6 h and 3 d), reporting confidence intervals on the dashboard, and raising the sampling rate during canaries. Also log judged examples so a quality incident has evidence for the postmortem.

### Q22. How do you design multi-tenant SLOs when one tenant can produce 80% of traffic? (Senior)

**Answer.** Publish one aggregate SLO for the service but record per-tenant SLIs (`sum by (tenant)`) for diagnosis. Treat rate-limited responses (429) as good so an abusive tenant does not burn everyone's budget. Offer per-tenant or tiered SLOs only when capacity is isolated (separate node pools, quotas), because a shared-capacity promise to one tenant is a promise you cannot keep against another's burst.

### Q23. Describe what you review monthly for an SLO and what evidence changes the target. (Senior)

**Answer.** Attainment per SLI, budget remaining, every page and whether it corresponded to user impact (precision), every user complaint and whether the SLI saw it (recall), and open postmortem action items. Raise a target when the service has beaten it by a wide margin for two quarters and users would value it; lower it when it is missed repeatedly and the cost of meeting it exceeds the value; change the SLI when complaints and the SLI disagree.

---

## Principal

### Q24. Argue for and against excluding 4xx responses from "bad" events, and state your policy. (Principal)

**Answer.** For counting 4xx as good: the server behaved correctly; counting them as bad lets clients burn your budget with bad requests. Against: some 4xx are server-caused (429 from under-provisioning, 404 from a bad deploy that removed a route, 401 from a broken auth dependency). Policy: 4xx are good by default; specific codes become bad when caused by us, detected via separate SLIs (a 404-ratio alert after deploys, a 429 SLI tied to capacity), and reviewed monthly. Document the policy in the SLO specification so it is not relitigated during incidents.

### Q25. How would you make SLOs a lever for engineering investment at the organisational level? (Principal)

**Answer.** Publish a quarterly portfolio: every tier-1 service with target, attainment, budget consumption, and top budget-consuming incidents. Tie the error budget policy to planning: teams that breached twice in three months get reliability work prioritised in the next quarter with leadership sign-off; teams with chronically unused budget are asked whether the target is right or whether capacity can be reduced. Make the Backstage catalog show SLO status so the information is where decisions are made. Report SLO attainment, not uptime, to the board.

### Q26. A platform team wants to impose 99.95% on every service. Why is this wrong and what would you do instead? (Principal)

**Answer.** Uniform targets ignore user needs and dependency bounds, and an imposed SLO has no owner. Internal batch tools do not need four nines; a payments API might need more. Instead, the platform team provides tooling and a default template, publishes its own platform SLOs (ingress, Kubernetes API, Argo CD), and facilitates each service team's four-meeting process to set their own targets from measured data and user expectations. The platform's role is to make it easy, not to decide the number.

### Q27. Design the SLO set for the capstone's "ask the agent" journey end to end. (Principal)

**Answer.** Measure the journey directly at the edge with a counter `journey_requests_total{journey="ask-agent",outcome}` and a duration histogram, giving availability (outcome=ok) and latency SLIs that already include FastAPI, vLLM, and retrieval. Add component SLOs (FastAPI availability 99.9%, vLLM TTFT 95% < 0.75 s, Postgres acquisition latency) for diagnosis and for the dependency table. Add quality SLIs at the agent level (grounded rate 98% sampled, tool-call success 99%) and a cost SLI. Set the journey availability target from the dependency bound, likely 99.5%, with graceful degradation (non-AI answer) to keep availability high and move AI failures into the quality SLI. Alert on journey burn rate; ticket on component burn.

### Q28. How do native histograms change SLI engineering? (Principal)

**Answer.** Native histograms (Prometheus 2.40+ experimental, stable in 3.x) replace fixed `le` buckets with sparse exponential buckets, cutting series count 10-20x and removing the "my threshold is not a boundary" problem because `histogram_fraction(0, 0.3, ...)` interpolates within much narrower buckets. The cost is that the SLI is again an interpolation (very precise, but not exact), that client libraries and remote-write support vary, and that Sloth/Pyrra support is newer (Pyrra has `latencyNative`). For new services I would adopt them and record the precision trade-off in the SLO document.

### Q29. The CFO asks why you let the service be down 43 minutes a month "on purpose". Answer. (Principal)

**Answer.** We do not plan to be down; we accept that change carries risk and set a limit on the risk we tolerate so we can ship. Each additional nine costs roughly ten times more (multi-region, no daytime deploys, slower features) and beyond a point users cannot perceive the difference because their own networks are less reliable. The 43.2 minutes is the agreed allowance that lets us deploy daily; when we spend it, we stop and fix. The contractual SLA is looser (99.5%) so this internal target protects revenue rather than risking it.

### Q30. What are the most common ways SLO programmes fail after a year, and what prevents each? (Principal)

**Answer.** (1) SLOs become dashboard decoration because the policy has no teeth: prevent with named override authority and leadership review of breaches. (2) Too many SLOs, so none is prioritised: cap at 1-3 per journey. (3) SLIs drift from users (new journeys, new failure modes return 200 with errors): monthly complaint-versus-SLI review. (4) Rules rot when instrumentation changes: generated rules in CI with unit tests that fail on missing series. (5) Nobody owns the review: put it on the calendar with an agenda and an owner in the SLO document.
