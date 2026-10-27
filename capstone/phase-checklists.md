# Phase Checklists: Definition of Done

> Tick each item in the capstone repository before calling a phase complete. Each item links to the wiki folder that teaches it. Items marked **(bridge)** are the sibling tools mapped by pattern transfer after the anchor is fluent.

---

## Phase 1: Months 1-4 (160 h) — Foundations, Backend, Docker, GitHub Actions

Anchors: Linux, Python, FastAPI, Docker, GitHub Actions. Bridges: Java 21 & Spring Boot, Docker Compose.

- [ ] Linux lab (WSL2/multipass) with `systemd`, `ss`, `strace`, `tcpdump` exercises completed — [00-foundations/linux](../00-foundations/linux/)
- [ ] Can subnet a /16 into /24s by hand, explain NAT and a TLS 1.3 handshake — [00-foundations/networking](../00-foundations/networking/)
- [ ] Git: trunk-based flow, signed commits, pre-commit framework installed — [00-foundations/git](../00-foundations/git/)
- [ ] Bash scripts use strict mode, pass shellcheck, have bats tests — [00-foundations/bash-scripting](../00-foundations/bash-scripting/)
- [ ] FastAPI incident API: CRUD, Pydantic v2, async SQLAlchemy + Postgres, `/healthz`, `/readyz`, `/metrics`, OpenAPI — [07-backend/fastapi](../07-backend/fastapi/)
- [ ] pytest suite with Testcontainers Postgres, coverage ≥ 80% — [07-backend/python](../07-backend/python/)
- [ ] **(bridge)** Spring Boot incident API with same contract, Actuator liveness/readiness/prometheus, virtual threads on — [07-backend/spring-boot](../07-backend/spring-boot/)
- [ ] **(bridge)** JUnit 5 + Testcontainers `@ServiceConnection`, JaCoCo ≥ 80% — [07-backend/java](../07-backend/java/)
- [ ] Both Dockerfiles: multi-stage, distroless/JRE, non-root, HEALTHCHECK, `.dockerignore`, image < 200 MB (Python) — [04-containers-and-idp/docker](../04-containers-and-idp/docker/)
- [ ] **(bridge)** Docker Compose runs both services + Postgres locally with healthcheck-gated `depends_on`
- [ ] GitHub Actions: CI per service (lint, type-check, test, coverage), matrix on Python 3.12/3.13 and Java 21 — [06-cicd/github-actions](../06-cicd/github-actions/)
- [ ] Images built with buildx, multi-arch (amd64/arm64), pushed to GHCR by digest, actions pinned to SHA
- [ ] `permissions:` block least-privilege in every workflow; no long-lived cloud keys anywhere

**Milestone:** polyglot microservice pair in multi-stage distroless images with automated tests and OCI publishing.

---

## Phase 2: Months 5-8 (160 h) — AWS and Terraform

Anchors: AWS (VPC, IAM, EKS, RDS, S3, KMS), Terraform. Bridges: Azure, GCP, CloudFormation, Pulumi (Python).

- [ ] AWS Organizations lab OU with SCP denying regions outside `us-east-1` and denying IAM user key creation — [01-cloud/aws](../01-cloud/aws/)
- [ ] GitHub OIDC provider + role with trust policy scoped to `repo:<org>/<repo>:ref:refs/heads/main`
- [ ] Terraform bootstrap: S3 state bucket (versioned, KMS, Block Public Access) with state locking — [05-infrastructure-as-code/terraform](../05-infrastructure-as-code/terraform/)
- [ ] Modules: `vpc` (3 AZ, public/private/data subnets, NAT, flow logs, endpoints), `eks`, `rds-postgres`, `s3-secure-bucket`, `ecr`
- [ ] `terraform test` suite passes; `tflint` and `checkov` clean; `terraform plan` posted as PR comment; apply on merge via OIDC
- [ ] Nightly `terraform destroy` cron workflow proven to run; AWS Budget at $15 with alert
- [ ] Mandatory tags enforced by a custom Checkov policy — [08-devsecops/gate-5-policy-as-code/checkov](../08-devsecops/gate-5-policy-as-code/checkov/)
- [ ] **(bridge)** Pulumi Python stack provisions the same RDS + S3 for one service; unit tests with mocks — [05-infrastructure-as-code/pulumi](../05-infrastructure-as-code/pulumi/)
- [ ] **(bridge)** CloudFormation template for the same service deployed via change set — [05-infrastructure-as-code/cloudformation](../05-infrastructure-as-code/cloudformation/)
- [ ] **(bridge)** Azure: Resource Group + VNet + AKS or Container Apps + Key Vault via `az` and a Managed Identity — [01-cloud/azure](../01-cloud/azure/)
- [ ] **(bridge)** GCP: project + VPC + GKE Autopilot or Cloud Run + Workload Identity Federation — [01-cloud/gcp](../01-cloud/gcp/)
- [ ] ADR written: "Terraform as IaC anchor; when Pulumi/CloudFormation instead" — [adrs/](adrs/)

**Milestone:** multi-AZ VPC, managed Postgres and container cluster via modular Terraform with remote state locking; cross-IaC fluency verified.

---

## Phase 3: Months 9-12 (180 h) — Kubernetes, GitOps, Backstage

Anchors: Kubernetes (EKS/kind, Helm, Argo CD, HPA, NetworkPolicy), Backstage. Bridges: Jenkins, Azure DevOps.

- [ ] `kind-up.sh` creates a 3-node cluster with Gateway API CRDs and a local registry — [04-containers-and-idp/kubernetes](../04-containers-and-idp/kubernetes/)
- [ ] Both services deployed with Deployment, Service, HTTPRoute, HPA, PDB, NetworkPolicy (default deny), probes, requests/limits, PSS `restricted`
- [ ] Zero-downtime rolling update demonstrated under load (k6) with 0 failed requests
- [ ] Production Helm chart with `values.schema.json`, HPA/PDB/NetworkPolicy/ServiceMonitor toggles, `helm lint` + `ct` in CI — [04-containers-and-idp/helm](../04-containers-and-idp/helm/)
- [ ] Argo CD installed via app-of-apps; automated sync with prune + self-heal; dev/stage/prod overlays; image digest bumped by CI commit — [04-containers-and-idp/argocd](../04-containers-and-idp/argocd/)
- [ ] EKS cluster from Phase 2 joined as a second Argo CD destination (ephemeral)
- [ ] Backstage running on kind with Postgres; GitHub OIDC login — [04-containers-and-idp/backstage](../04-containers-and-idp/backstage/)
- [ ] Software Catalog: both services, their APIs, the Postgres resource, owning Group; Kubernetes plugin shows pods
- [ ] Golden-path template scaffolds a new FastAPI or Spring Boot repo with Dockerfile, CI, chart, Argo Application and `catalog-info.yaml` in under 60 seconds
- [ ] TechDocs renders this wiki and each service's docs
- [ ] **(bridge)** Jenkins on kind via Helm with JCasC; Jenkinsfile with Kubernetes pod agents and a shared library running the same CI — [06-cicd/jenkins](../06-cicd/jenkins/)
- [ ] **(bridge)** Azure Pipelines YAML with stage/job/step templates and an environment approval for the same CI — [06-cicd/azure-devops](../06-cicd/azure-devops/)
- [ ] ADR written: "Argo CD vs Flux", "Helm vs Kustomize"

**Milestone:** both services on Kubernetes via GitOps; Backstage portal with catalog and one-click golden path.

---

## Phase 4: Months 13-16 (160 h) — Observability and SRE

Anchors: OpenTelemetry, Prometheus, Grafana, Loki, SLI/SLO/error budgets. Bridges: ELK, OnCall routing.

- [ ] Local `docker-compose.lgtm.yaml` and Kubernetes `observability` namespace (kube-prometheus-stack, Loki, Tempo, OTel Collector, Grafana) — [09-observability](../09-observability/)
- [ ] FastAPI instrumented with `opentelemetry-instrument`; Spring Boot with the OTel Java agent; both emit traces, metrics, logs with `trace_id` — [09-observability/opentelemetry](../09-observability/opentelemetry/)
- [ ] OTel Collector gateway: `k8sattributes`, `memory_limiter`, `batch`, PII `redaction`, `tail_sampling`, `spanmetrics` connector
- [ ] Grafana dashboards: RED per service, Kubernetes cluster, SLO/burn-rate; trace-to-logs and exemplars configured — [09-observability/grafana](../09-observability/grafana/)
- [ ] SLO document for each service: 99.9% availability + latency < 300 ms over 30 days — [11-sre/sli-slo-error-budgets](../11-sre/sli-slo-error-budgets/)
- [ ] `PrometheusRule` with recording rules and multi-window multi-burn-rate alerts; `promtool test rules` passes — [11-sre/burn-rate-alerting](../11-sre/burn-rate-alerting/)
- [ ] Alertmanager routes pages to Grafana OnCall (or PagerDuty free tier) and tickets to Slack/Jira; every alert has a runbook URL — [11-sre/oncall-and-incident-management](../11-sre/oncall-and-incident-management/)
- [ ] Error injected (fault flag) → 14.4× alert fires within 5 minutes → page received → runbook followed → postmortem written — [11-sre/postmortems-and-runbooks](../11-sre/postmortems-and-runbooks/)
- [ ] Argo Rollouts canary for FastAPI with Prometheus AnalysisTemplate auto-rolling back if 5xx > 0.5% — [11-sre/progressive-delivery](../11-sre/progressive-delivery/)
- [ ] **(bridge)** ELK via ECK on kind ingesting the same logs; KQL queries equivalent to the LogQL ones — [09-observability/elk-stack](../09-observability/elk-stack/)
- [ ] ADR written: "Loki for operational logs vs ELK for SIEM"

**Milestone:** unified correlation dashboards and a 99.9% SLO paging an on-call schedule via burn-rate alerts.

---

## Phase 5: Months 17-20 (180 h) — DevSecOps, DR, FinOps, Governance

Anchors: Trivy, SonarQube, Semgrep, Checkov, OWASP ZAP, Falco, Vault. Bridges: Checkmarx, Snyk, Kyverno, DefectDojo, FinOps, NIST CSF 2.0, DR drills.

- [ ] Gate 1: gitleaks pre-commit + PR check; TruffleHog `--only-verified` on history — [gate-1-secrets](../08-devsecops/gate-1-secrets/)
- [ ] Gate 2: SonarQube quality gate (0 new blockers, 80% new-code coverage) blocks PR; Semgrep with 6+ custom org rules — [gate-2-sast](../08-devsecops/gate-2-sast/)
- [ ] **(bridge)** Checkmarx One CLI scan documented; Snyk test with `--severity-threshold=high` — [gate-3-sca](../08-devsecops/gate-3-sca/)
- [ ] Gate 3: CycloneDX SBOMs for both services uploaded to Dependency-Track with a license + CVSS policy
- [ ] Gate 4: Syft SBOM, Trivy (`--exit-code 1 --severity HIGH,CRITICAL --ignore-unfixed`), Grype on SBOM, Cosign keyless sign + SBOM/SLSA attestations — [gate-4-supply-chain](../08-devsecops/gate-4-supply-chain/)
- [ ] Gate 5: Checkov on Terraform plan JSON, manifests and Dockerfiles; Kyverno `verifyImages` rejects unsigned images; PSS restricted policies in Enforce; `kyverno test` in CI — [gate-5-policy-as-code](../08-devsecops/gate-5-policy-as-code/)
- [ ] **(bridge)** One OPA Gatekeeper ConstraintTemplate with `gator test` for comparison
- [ ] Gate 6: ZAP API scan against the review environment's OpenAPI; Nuclei custom templates for exposed actuator/debug endpoints — [gate-6-dast](../08-devsecops/gate-6-dast/)
- [ ] Gate 7: Falco modern eBPF with 6+ custom rules; falcosidekick to Slack and Loki; `kubectl exec` shell detected — [gate-7-runtime-and-secrets/falco](../08-devsecops/gate-7-runtime-and-secrets/falco/)
- [ ] Vault Raft HA on kind; Kubernetes auth; dynamic Postgres credentials via Agent sidecar for FastAPI and Spring Cloud Vault for Java; PKI issuing short-lived certs; no static DB password in any manifest — [gate-7-runtime-and-secrets/vault](../08-devsecops/gate-7-runtime-and-secrets/vault/)
- [ ] DefectDojo receives every gate's output via reimport; SLA by severity; Jira (or GitHub Issues) integration — [aspm-and-siem/defectdojo](../08-devsecops/aspm-and-siem/defectdojo/)
- [ ] **(bridge)** Wazuh single-node ingesting Falco alerts and Kubernetes audit logs — [aspm-and-siem/wazuh](../08-devsecops/aspm-and-siem/wazuh/)
- [ ] Multi-AZ failover drill executed with `dr-drill-az-failure.sh`; measured RPO/RTO recorded against targets — [12-enterprise-ops/disaster-recovery](../12-enterprise-ops/disaster-recovery/)
- [ ] Velero scheduled backups of stateful namespaces to S3 with Object Lock; restore tested
- [ ] OpenCost showback dashboard; Infracost PR comments; Karpenter Spot + consolidation; Graviton images — [12-enterprise-ops/finops](../12-enterprise-ops/finops/)
- [ ] NIST CSF 2.0 profile mapping every implemented control; production readiness checklist signed off — [12-enterprise-ops/cybersecurity-governance](../12-enterprise-ops/cybersecurity-governance/)
- [ ] ADRs written: "Trivy as unified scanner vs Grype+Syft", "Kyverno vs Gatekeeper", "Vault vs cloud secret managers"

**Milestone:** all 7 gates wired; multi-AZ failover drill validates RPO/RTO.

---

## Phase 6: Months 21-24 (200 h) — AI Platform

Anchors: Amazon Bedrock, LangGraph, Strands Agents, Qdrant/pgvector, MCP, DeepEval. Bridges: Vertex AI, Azure OpenAI, vLLM GPU serving, Milvus, Promptfoo.

- [ ] Bedrock model access, Converse API with streaming and tool use via PrivateLink endpoint; Guardrail with PII redaction applied — [02-managed-ai-platforms/amazon-bedrock](../02-managed-ai-platforms/amazon-bedrock/)
- [ ] Bedrock Knowledge Base over runbooks (S3 → OpenSearch Serverless or Aurora pgvector) compared against self-managed Qdrant RAG
- [ ] Runbooks ingested with header-aware chunking into a Qdrant collection with dense + sparse vectors; hybrid query with RRF and cross-encoder rerank — [10-ai-infra-rag-mcp/rag-architecture](../10-ai-infra-rag-mcp/rag-architecture/), [vector-databases/qdrant](../10-ai-infra-rag-mcp/vector-databases/qdrant/)
- [ ] **(bridge)** Same corpus in pgvector (HNSW) with a recall/latency comparison; Milvus standalone trialled — [vector-databases](../10-ai-infra-rag-mcp/vector-databases/)
- [ ] MCP servers (Prometheus, Kubernetes, runbooks) built with FastMCP, deployed to `ai-infra` with least-privilege RBAC, tested with MCP Inspector — [10-ai-infra-rag-mcp/mcp](../10-ai-infra-rag-mcp/mcp/)
- [ ] LangGraph incident agent: typed state, Postgres checkpointer, `interrupt()` approval before `rollout_restart`, streaming to the SSE endpoint — [03-agentic-frameworks/langgraph](../03-agentic-frameworks/langgraph/)
- [ ] Strands Agents version of the same agent with MCPClient and OTel tracing; ADR comparing the two — [03-agentic-frameworks/strands-agents](../03-agentic-frameworks/strands-agents/)
- [ ] Every LLM and tool call traced with `gen_ai.*` attributes and token counts; token-cost dashboard in Grafana
- [ ] **(bridge)** vLLM serving a small open model on a Spot GPU node (or CPU locally) via the GPU Operator; TTFT SLO alert; HPA on queue depth — [10-ai-infra-rag-mcp/vllm](../10-ai-infra-rag-mcp/vllm/), [gpu-workloads-kubernetes](../10-ai-infra-rag-mcp/gpu-workloads-kubernetes/)
- [ ] DeepEval suite (`ToolCorrectnessMetric`, `TaskCompletionMetric`, G-Eval) fails the PR below thresholds — [llm-evals/deepeval](../10-ai-infra-rag-mcp/llm-evals/deepeval/)
- [ ] RAGAS faithfulness ≥ 0.85 and context precision ≥ 0.8 on the golden runbook dataset in CI — [llm-evals/ragas](../10-ai-infra-rag-mcp/llm-evals/ragas/)
- [ ] **(bridge)** Promptfoo matrix across Bedrock/vLLM models and a red-team run (prompt injection, PII leakage, excessive agency) with 0 failures — [llm-evals/promptfoo](../10-ai-infra-rag-mcp/llm-evals/promptfoo/)
- [ ] **(bridge)** Vertex AI (Gemini via `google-genai`) and Azure OpenAI (`AzureOpenAI` with Entra ID) provider swaps verified through LangChain `init_chat_model` — [02-managed-ai-platforms](../02-managed-ai-platforms/)
- [ ] Semantic cache and model routing cut measured token spend; batch inference used for nightly eval generation
- [ ] ADRs written: "Strands Agents vs LangGraph for stateful workflows", "pgvector vs Qdrant", "Bedrock vs self-hosted vLLM"

**Milestone:** the FastAPI service is an autonomous Incident & Platform AI Agent with hybrid RAG, MCP tools, GPU serving and CI-gated evals.
