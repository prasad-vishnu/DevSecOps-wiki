# ADR-0002: Loki for operational logs, ELK for SIEM and full-text analytics

| | |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-10-27 |
| **Deciders** | Platform lead, security lead |
| **Cluster** | 09 Observability |
| **Capstone phase** | Phase 4 |

## Context

The capstone emits structured JSON logs from the FastAPI and Spring Boot services, Kubernetes audit logs, Falco runtime alerts, and (in cloud) CloudTrail and VPC flow logs. Two very different consumers exist:

1. **Operators debugging incidents** need to jump from a Grafana latency panel to the exact pod's logs for the trace that was slow, usually filtered by `namespace`, `pod`, `trace_id`. Query volume is low, time ranges are short, and label-based filtering plus a `|= "error"` line filter is enough.
2. **Security analysts** need full-text search across every parsed field, high-cardinality aggregations (top source IPs, rare user agents), detection rules evaluated continuously, and long retention for compliance. This is the SIEM use case.

Forces: the lab budget is under $15/month, so storage cost per GB matters. Grafana is already the single pane of glass for metrics and traces. The roadmap states Loki is roughly 5 to 10 times cheaper for routine operational debugging because it indexes only labels and stores compressed chunks in object storage, while ELK's inverted Lucene indexes deliver sub-second full-text search at higher storage and JVM memory cost.

## Decision

We will use **Grafana Loki as the operational log store** for all application and platform logs, shipped by the OpenTelemetry Collector (or Grafana Alloy) with a small, low-cardinality label set (`cluster`, `namespace`, `app`, `container`, `level`) and `trace_id` as structured metadata rather than a label. We will run **Elasticsearch + Kibana (via ECK) as the SIEM and analytics store**, receiving only security-relevant streams: Kubernetes audit logs, Falco alerts, Wazuh events and cloud audit logs. Logstash or the OTel Collector `elasticsearch` exporter handles parsing into ECS.

The mental model transfers between the two: both are label-or-field-indexed time-partitioned stores; LogQL stream selectors map to KQL field filters; Loki's `| json` parser maps to Elasticsearch mappings. Learners get both query languages against the same data during the Phase 4 bridge exercise.

## Alternatives Considered

| Option | Pros | Cons | Why rejected (or deferred) |
|---|---|---|---|
| ELK for everything | One system; best full-text search; Elastic Security detection rules; vector search for RAG | JVM heap sizing, shard management and mapping explosions are heavy for one engineer; storage 5 to 10 times Loki for the same logs; licence boundary for alerting/SSO features | Rejected as the operational store on cost and operational load |
| Loki for everything | Cheapest; perfect Grafana integration; same labels as Prometheus | No full-text index: `|~` regex over large ranges is slow; high-cardinality security aggregations are painful; no mature detection-rule ecosystem | Rejected for SIEM |
| OpenSearch instead of Elasticsearch | Apache 2.0; AWS-managed option; Security Analytics plugin | Diverging feature set from Elastic; fewer tutorials; Kibana/Dashboards drift | Acceptable substitute; the ADR holds if an employer mandates OpenSearch |
| Cloud-native (CloudWatch Logs Insights / Azure Log Analytics) | Zero ops | Per-GB ingest pricing is the most expensive option at scale; vendor-specific query languages; weaker Grafana correlation | Used only for cloud control-plane logs that land there anyway |
| ClickHouse-based (SigNoz, HyperDX) | Fast columnar analytics; cheap storage | Another query dialect; smaller ecosystem; not in the roadmap | Deferred |

## Consequences

### Positive

- Operational debugging stays inside Grafana with trace-to-logs and logs-to-trace links.
- Log storage cost for the noisy application streams is minimised.
- Security gets a purpose-built detection and hunting platform without polluting it with debug logs.

### Negative / Risks

- Two systems to run and secure. Mitigation: ECK operator for ELK, single-binary or simple-scalable Loki, both in the `observability` namespace with resource limits.
- Engineers must know two query languages. Mitigation: the Phase 4 bridge exercise writes each LogQL query's KQL twin; cheat sheets live in [../../09-observability/loki/](../../09-observability/loki/) and [../../09-observability/elk-stack/](../../09-observability/elk-stack/).
- Label cardinality mistakes (putting `trace_id` or `user_id` in Loki labels) explode index size. Mitigation: OTel Collector `attributes` processor allow-list for labels; alert on `loki_ingester_memory_streams`.

### Neutral / Follow-ups

- Revisit when log volume exceeds 50 GB/day or when Elastic's vector search is needed for RAG (compare with Qdrant in ADR-0003).

## Compliance and Security Impact

Audit and security logs are retained in Elasticsearch for 365 days with ILM moving them to frozen tier, satisfying typical audit requirements. Application logs in Loki retain 14 days. PII is redacted at the OTel Collector before either store.

## Cost Impact

Local lab: both run on kind with 2 GB RAM each. Cloud estimate at 10 GB/day: Loki on S3 roughly $3 to $5/month storage; Elasticsearch for 1 GB/day of security logs on a single `t4g.medium`-class node roughly $25/month, run only during security labs and destroyed nightly.

## Validation

Mean time to find the log line for a slow trace from a Grafana panel is under 60 seconds. A Falco `Terminal shell in container` alert appears as an Elastic Security detection within 1 minute. Review at end of Phase 5.

## References

- [../../09-observability/README.md](../../09-observability/README.md)
- Roadmap Cluster 9: Loki vs ELK comparison
