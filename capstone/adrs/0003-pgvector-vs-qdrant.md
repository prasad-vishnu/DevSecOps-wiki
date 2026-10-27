# ADR-0003: Qdrant for the runbook RAG index, pgvector for transactional embeddings

| | |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-10-27 |
| **Deciders** | Platform lead, AI lead |
| **Cluster** | 10 AI Infra & RAG/MCP |
| **Capstone phase** | Phase 6 |

## Context

The Incident & Platform Agent retrieves runbooks, postmortems and SLO documents to ground its reasoning. The corpus is small today (hundreds of Markdown files, tens of thousands of chunks) but the retrieval quality bar is high: a wrong runbook during an incident is worse than no runbook. The roadmap's production RAG pattern requires hybrid search (dense embeddings plus BM25 sparse vectors), Reciprocal Rank Fusion, metadata filtering (service, severity, environment, document ACL) and cross-encoder reranking.

Separately, the incident API stores incidents in Postgres and wants "similar past incidents" lookups that must stay transactionally consistent with the incident rows themselves.

Forces:

- **Scale.** Far below the 10 to 50 million vector threshold where pgvector stops being the obvious default.
- **Hybrid search.** Native sparse+dense vectors with fusion in one query simplify the retriever; in Postgres this requires `tsvector` plus SQL-side RRF.
- **Filtering during HNSW traversal.** Payload filters that apply inside graph traversal (not post-filter) keep recall stable when filters are selective (for example `service = "incident-api-python"`).
- **Operational footprint.** One engineer, kind cluster, under $15/month.
- **Consistency.** Embeddings of incident rows must not drift from the rows.
- **Learning goals.** The roadmap names Qdrant as the anchor vector database and pgvector as the ideal default below 10 to 50 million vectors.

## Decision

We will use **Qdrant as the vector store for the runbook/knowledge RAG index**, with named dense and sparse vectors per point, payload indexes on `service`, `severity`, `doc_type` and `acl_groups`, scalar quantisation with rescoring, and collection aliases for zero-downtime reindexing. We will use **pgvector inside the existing Postgres** for embeddings that belong to transactional rows (incident similarity), created in the same transaction as the row and indexed with HNSW using cosine distance.

Qdrant's rich payload filtering during HNSW traversal, built-in sparse vectors and `prefetch` plus RRF fusion match the production RAG pattern with the least application code. pgvector keeps ACID guarantees where they matter and avoids a dual-write problem for incident data.

## Alternatives Considered

| Option | Pros | Cons | Why rejected (or deferred) |
|---|---|---|---|
| pgvector for everything | One database; ACID; joins with business tables; cheapest; team already knows Postgres | Hybrid search is hand-rolled (`tsvector` + RRF SQL); filtering is post-filter unless partial indexes match the predicate; HNSW build memory and `ef_search` tuning fall on the app team | Kept for transactional embeddings only. Would be the full answer if hybrid and ACL filtering requirements were weaker |
| Qdrant for everything | One vector API; best filtering and quantisation | Dual-write between Postgres incidents and Qdrant points; eventual consistency bugs in incident similarity | Rejected for transactional data |
| Milvus | Billion-scale, GPU indexing, disaggregated | Many components (etcd, message queue, object store, coordinators); heavy for kind; overkill at this scale | Deferred; trialled as a bridge exercise in Phase 6 |
| Pinecone | Zero ops, serverless, integrated rerank | External SaaS and data egress for runbooks that may contain internal hostnames; lock-in; cost model (read/write units) hard to predict | Rejected for the lab; documented as the managed option |
| Amazon Bedrock Knowledge Bases (OpenSearch Serverless) | Fully managed ingestion, chunking, retrieval; tight Bedrock integration | OpenSearch Serverless minimum OCUs cost roughly $170+/month; less control over chunking and hybrid tuning | Used for a comparison lab only, destroyed same day |
| Elasticsearch kNN + BM25 | Already running for SIEM; mature hybrid with RRF | JVM memory; mixing security data and RAG data in one cluster blurs trust boundaries | Rejected for separation of concerns |

## Consequences

### Positive

- Hybrid retrieval with RRF and payload ACL filtering is a single Qdrant query.
- Reindexing with a new embedding model is a new collection plus an alias flip.
- Incident similarity is always consistent with incident rows.

### Negative / Risks

- Two retrieval code paths. Mitigation: one `Retriever` interface in the FastAPI service with Qdrant and pgvector implementations.
- Qdrant is another StatefulSet with a PVC and snapshot schedule. Mitigation: Helm chart in `gitops/platform/ai-infra`, snapshots to S3 nightly, Velero backup.
- Embedding model changes require re-embedding both stores. Mitigation: store `embedding_model` and `embedding_version` in payload/columns; the ingest job is idempotent.

### Neutral / Follow-ups

- Reassess if the corpus passes 10 million chunks or if multi-tenant isolation requires per-tenant collections.
- Evaluate pgvectorscale if pgvector recall/latency at scale becomes limiting.

## Compliance and Security Impact

Runbooks may include internal hostnames and architecture details. Qdrant runs inside the cluster with TLS, an API key from Vault, a default-deny NetworkPolicy allowing only the agent's namespace, and payload ACL filtering so the agent cannot retrieve documents the requesting user's groups cannot read. No vector data leaves the VPC.

## Cost Impact

Qdrant on kind: 1 CPU, 1 GB RAM. In cloud, a single `t4g.small` equivalent pod with a 10 GB gp3 PVC is roughly $5/month, destroyed nightly. pgvector adds no infrastructure cost.

## Validation

RAGAS context precision ≥ 0.8 and context recall ≥ 0.8 on the golden runbook dataset; p95 retrieval latency under 150 ms for hybrid queries with rerank over 50 candidates. Review at end of Phase 6.

## References

- [../../10-ai-infra-rag-mcp/vector-databases/README.md](../../10-ai-infra-rag-mcp/vector-databases/README.md)
- [../../10-ai-infra-rag-mcp/rag-architecture/](../../10-ai-infra-rag-mcp/rag-architecture/)
- Roadmap Cluster 10: vector database selection guidance
