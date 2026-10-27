# Architecture Decision Records

> Golden Rule 5 of the roadmap: document every phase with one-page ADRs explaining why each tool was chosen and how its alternatives compare. These become the portfolio for principal-level interviews.

Use [../../templates/adr-template.md](../../templates/adr-template.md). One decision per file, numbered sequentially, never deleted (supersede instead).

## Index

| ADR | Title | Status | Phase |
|---|---|---|---|
| [0001](0001-terraform-as-iac-anchor.md) | Terraform as the IaC anchor; Pulumi and CloudFormation as bridges | Accepted | 2 |
| [0002](0002-loki-for-operational-logs-elk-for-siem.md) | Loki for operational logs, ELK for SIEM and full-text analytics | Accepted | 4 |
| [0003](0003-pgvector-vs-qdrant.md) | Qdrant for the runbook RAG index, pgvector for transactional embeddings | Accepted | 6 |
| [0004](0004-strands-agents-vs-langgraph.md) | LangGraph for the stateful incident agent, Strands for lightweight tool agents | Accepted | 6 |

## Suggested future ADRs (write them as you reach each decision)

- Argo CD vs Flux for GitOps
- Helm vs Kustomize for application packaging
- Kyverno vs OPA Gatekeeper for admission policy
- Trivy unified scanning vs Syft + Grype pairing
- Vault vs cloud-native secret managers
- Amazon Bedrock vs self-hosted vLLM for the agent's reasoning model
- GitHub Actions hosted runners vs Actions Runner Controller on EKS
- Grafana OnCall vs PagerDuty for paging
- Pilot-light vs warm-standby DR tier for the capstone
