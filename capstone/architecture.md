# Capstone Architecture

> The end state after Phase 6, described layer by layer with the data flows, trust boundaries and the specific tools from each cluster. Each section links to the wiki folder that teaches the tool.

---

## 1. Layered View

```mermaid
flowchart LR
    L0[00 Foundations<br/>Linux, networking, Git, Bash]
    L1[01 Cloud<br/>AWS anchor]
    L5[05 IaC<br/>Terraform]
    L4[04 Containers & IDP<br/>Docker, K8s, Helm, Argo CD, Backstage]
    L6[06 CI/CD<br/>GitHub Actions]
    L7[07 Backend<br/>FastAPI, Spring Boot]
    L8[08 DevSecOps<br/>7 gates, 20 tools]
    L9[09 Observability<br/>OTel + LGTM]
    L11[11 SRE<br/>SLO, burn rate, on-call]
    L12[12 Enterprise Ops<br/>DR, FinOps, NIST CSF]
    L2[02 Managed AI<br/>Bedrock]
    L3[03 Agents<br/>LangGraph, Strands]
    L10[10 AI Infra<br/>vLLM, Qdrant, MCP, evals]

    L0 --> L1 --> L5 --> L4
    L7 --> L6 --> L4
    L8 -.gates.-> L6
    L8 -.admission/runtime.-> L4
    L4 --> L9 --> L11 --> L12
    L2 & L10 --> L3 --> L7
```

---

## 2. Network and Trust Boundaries (Cloud)

```mermaid
flowchart TB
    Internet((Internet))
    subgraph AWS["AWS account 123456789012 (lab OU, SCP-guarded)"]
        subgraph VPC["VPC 10.0.0.0/16, 3 AZs"]
            subgraph Pub["Public subnets"]
                ALB[ALB / NLB<br/>TLS 1.3, ACM cert]
                NAT[NAT Gateway]
            end
            subgraph Priv["Private subnets"]
                EKS[EKS worker nodes<br/>Karpenter, Graviton, Spot]
                GPU[GPU node pool<br/>g5, taint nvidia.com/gpu]
            end
            subgraph Data["Isolated data subnets"]
                RDS[(RDS Postgres Multi-AZ<br/>KMS CMEK, IAM auth)]
            end
            VPCE[VPC endpoints:<br/>S3, ECR, STS, Secrets Manager,<br/>bedrock-runtime, CloudWatch]
        end
        S3[(S3: state, SBOMs, backups<br/>Object Lock, KMS)]
        ECR[ECR: signed images]
        BR[Amazon Bedrock<br/>Converse API, Guardrails]
        KMS[KMS CMKs]
    end
    Internet --> ALB --> EKS
    EKS --> NAT --> Internet
    EKS --> VPCE --> S3 & ECR & BR
    EKS --> RDS
    GPU --> VPCE
```

Rules enforced by Terraform and Checkov:

- No `0.0.0.0/0` ingress except on the ALB security group ports 443.
- Every S3 bucket: Block Public Access on, default SSE-KMS, versioning, lifecycle to Glacier after 30 days.
- RDS: `publicly_accessible = false`, `storage_encrypted = true`, deletion protection in `prod` workspace only.
- Every resource tagged `owner`, `cost-center`, `environment`, `ttl`.
- All CI access via GitHub Actions OIDC; no IAM user access keys exist in the account.

See [../01-cloud/aws/](../01-cloud/aws/), [../05-infrastructure-as-code/terraform/](../05-infrastructure-as-code/terraform/), [../08-devsecops/gate-5-policy-as-code/checkov/](../08-devsecops/gate-5-policy-as-code/checkov/).

---

## 3. Build and Supply Chain

```mermaid
sequenceDiagram
    autonumber
    participant Dev as Developer
    participant GH as GitHub (PR)
    participant GA as GitHub Actions
    participant REG as ECR / GHCR
    participant REK as Rekor (Sigstore)
    participant DD as DefectDojo
    participant ARGO as Argo CD
    participant KYV as Kyverno

    Dev->>GH: push branch (pre-commit: gitleaks, ruff, semgrep)
    GH->>GA: pull_request event
    GA->>GA: Gate 1 gitleaks + trufflehog --only-verified
    GA->>GA: Gate 2 semgrep ci + sonar-scanner (quality gate wait)
    GA->>GA: Gate 3 snyk test, cyclonedx SBOM -> Dependency-Track
    GA->>GA: build image with buildx (SBOM+provenance attestations)
    GA->>GA: Gate 4 syft sbom, trivy image --exit-code 1 --severity HIGH,CRITICAL, grype sbom
    GA->>REG: push image by digest
    GA->>REK: cosign sign --yes (keyless OIDC), cosign attest (SBOM, SLSA provenance)
    GA->>GA: Gate 5 checkov -d infra/ and manifests, kyverno test
    GA->>GA: deploy review env on kind-in-CI, Gate 6 zap-api-scan on OpenAPI
    GA->>DD: reimport-scan for every gate (SARIF/JSON)
    GA->>GH: PR status checks + SARIF to code scanning
    Dev->>GH: merge
    GA->>GH: update image digest in gitops/ overlays (commit)
    ARGO->>GH: detect change, sync
    ARGO->>KYV: apply manifests
    KYV->>REK: verifyImages: signature + attestation match issuer/subject
    KYV-->>ARGO: admit or deny
```

See [../06-cicd/github-actions/](../06-cicd/github-actions/) and the whole of [../08-devsecops/](../08-devsecops/).

---

## 4. Runtime Platform (Kubernetes)

| Namespace | Workloads | Policies |
|---|---|---|
| `platform-system` | Argo CD, Argo Rollouts, Kyverno, External Secrets Operator, cert-manager | PSS `restricted` (except Kyverno webhooks as documented) |
| `observability` | OTel Collector (gateway), Prometheus Operator stack, Loki, Tempo, Grafana, Alertmanager | NetworkPolicy allows only OTLP 4317/4318 ingress from app namespaces |
| `security` | Falco + falcosidekick, Trivy Operator, Vault (Raft HA) | Falco needs privileged DaemonSet; documented exception |
| `incident-api` | FastAPI incident API/agent (Rollout, canary), Spring Boot incident API (Rollout), Vault Agent sidecars | PSS `restricted`, default-deny NetworkPolicy, verifyImages, requests/limits required |
| `ai-infra` | vLLM (GPU), Qdrant StatefulSet, MCP servers (Prometheus, Kubernetes, runbooks) | MCP K8s server ServiceAccount has read-only ClusterRole plus a single `rollout restart` verb gated by human approval |
| `backstage` | Backstage frontend/backend, Postgres | OIDC login via GitHub |

Admission flow:

```mermaid
flowchart LR
    A[kubectl / Argo CD apply] --> B[API server authn/authz RBAC]
    B --> C[Mutating: Kyverno add default labels, imagePullPolicy]
    C --> D[Validating: Pod Security Admission restricted]
    D --> E[Validating: Kyverno policies<br/>non-root, no :latest, limits, probes, registry allow-list]
    E --> F[Kyverno verifyImages<br/>Cosign keyless identity + SBOM attestation]
    F --> G[etcd]
```

See [../04-containers-and-idp/kubernetes/](../04-containers-and-idp/kubernetes/), [../08-devsecops/gate-5-policy-as-code/kyverno/](../08-devsecops/gate-5-policy-as-code/kyverno/), [../08-devsecops/gate-7-runtime-and-secrets/](../08-devsecops/gate-7-runtime-and-secrets/).

---

## 5. Observability and Reliability

```mermaid
flowchart LR
    subgraph Apps
        FA[FastAPI<br/>OTel SDK auto-instr]
        SB[Spring Boot<br/>OTel Java agent]
        VL[vLLM /metrics]
        GPU[DCGM exporter]
    end
    FA & SB -->|OTLP gRPC| COL[OTel Collector gateway<br/>batch, memory_limiter, k8sattributes,<br/>redaction, tail_sampling, spanmetrics]
    COL -->|remote_write| PROM[(Prometheus)]
    COL -->|otlp| LOKI[(Loki)]
    COL -->|otlp| TEMPO[(Tempo)]
    VL & GPU -->|scrape| PROM
    PROM --> RULES[Recording rules:<br/>SLI ratios at 5m,30m,1h,6h,1d,3d]
    RULES --> ALERTS[Burn-rate alerts:<br/>14.4x page, 6x page, 3x ticket, 1x ticket]
    ALERTS --> AM[Alertmanager] --> OC[Grafana OnCall / PagerDuty]
    PROM & LOKI & TEMPO --> GRAF[Grafana<br/>RED dashboards, SLO dashboard,<br/>trace<->log<->metric correlation]
    OC -->|runbook link| RB[Runbooks in TechDocs]
    RB -.RAG index.-> QD[(Qdrant)]
```

SLO definition for both services:

| Field | Value |
|---|---|
| SLI (availability) | `sum(rate(http_requests_total{code!~"5.."}[w])) / sum(rate(http_requests_total[w]))` measured at the ingress |
| SLI (latency) | `sum(rate(http_request_duration_seconds_bucket{le="0.3"}[w])) / sum(rate(http_request_duration_seconds_count[w]))` |
| SLO | 99.9% over rolling 30 days |
| Error budget | 0.1% = 43.2 minutes or 1,000 bad requests per million |
| Paging | Burn rate 14.4× over 1 h and 5 m; 6× over 6 h and 30 m |
| Ticketing | 3× over 1 d and 2 h; 1× over 3 d and 6 h |

For the agent and vLLM: TTFT under 800 ms at p95, grounded-answer rate above 95% (from RAGAS faithfulness on sampled production traces), tool-call success rate above 99%.

See [../09-observability/](../09-observability/) and [../11-sre/](../11-sre/).

---

## 6. The AI Layer (Phase 6)

```mermaid
flowchart TB
    PAGE[Alert fires: FastAPI 5xx burn rate 14.4x] --> AG

    subgraph AG["Incident & Platform Agent (FastAPI service, LangGraph or Strands)"]
        direction TB
        S1[Node: gather context] --> S2[Node: retrieve runbooks<br/>hybrid dense+BM25, RRF, rerank]
        S2 --> S3[Node: reason with Bedrock Claude<br/>or self-hosted vLLM model]
        S3 --> S4{Tool call?}
        S4 -->|read-only| T1[MCP Prometheus: query_range, list_alerts]
        S4 -->|read-only| T2[MCP Kubernetes: list_pods, get_logs, get_events]
        S4 -->|destructive| HITL[interrupt: human approval in Slack]
        HITL -->|approved| T3[MCP Kubernetes: rollout_restart]
        T1 & T2 & T3 --> S3
        S3 --> S5[Node: write incident timeline, propose postmortem draft]
    end

    S2 --> QD[(Qdrant: runbooks collection<br/>dense + sparse vectors, payload ACL)]
    S3 --> BR[Bedrock Converse API + Guardrails]
    S3 --> VL[vLLM: Qwen/Llama on GPU pool]
    AG -->|OTel spans per LLM & tool call,<br/>gen_ai.* attributes, token counts| COL[OTel Collector]
    CKPT[(Postgres checkpointer<br/>thread_id = incident id)] <--> AG
```

Quality gates in CI for the agent:

| Layer | Tool | Gate |
|---|---|---|
| RAG triad | RAGAS | faithfulness ≥ 0.85, context precision ≥ 0.8 on the golden runbook dataset |
| Agent behaviour | DeepEval | `ToolCorrectnessMetric` ≥ 0.9, `TaskCompletionMetric` ≥ 0.8, G-Eval rubric for "safe remediation proposal" |
| Prompt regression + red team | Promptfoo | 0 failures on prompt-injection, PII leakage and excessive-agency plugins against the staging agent |
| Production grading | LangSmith (or self-hosted alternative) | Sample 5% of traces, online faithfulness evaluator, feed failures to the dataset |

See [../03-agentic-frameworks/](../03-agentic-frameworks/), [../10-ai-infra-rag-mcp/](../10-ai-infra-rag-mcp/), [../02-managed-ai-platforms/amazon-bedrock/](../02-managed-ai-platforms/amazon-bedrock/).

---

## 7. Disaster Recovery and FinOps Posture

| Area | Decision |
|---|---|
| DR tier | Pilot light for the lab (RDS cross-region read replica, Terraform-recreatable cluster, Velero backups of stateful namespaces to S3 with Object Lock) |
| RPO / RTO targets | RPO 5 minutes (RDS replication lag), RTO 30 minutes (Terraform + Argo CD bootstrap) |
| Drill | Quarterly: drain one AZ's nodes, verify SLO holds; annual: full region failover in a lab account |
| FinOps | OpenCost showback per namespace; Infracost on PRs; Karpenter consolidation; Graviton images; token spend dashboard; semantic cache in front of Bedrock |
| Governance | NIST CSF 2.0 profile mapping every control in [../12-enterprise-ops/cybersecurity-governance/](../12-enterprise-ops/cybersecurity-governance/) |

See [../12-enterprise-ops/](../12-enterprise-ops/).
