# Glossary

Terms used across the wiki, grouped by cluster. Each entry is one or two sentences; follow the link for depth.

## Cross-cutting patterns

| Term | Meaning | Where |
|---|---|---|
| Anchor-and-Bridge | Learn one tool per layer deeply (anchor), then map siblings (bridges) by vocabulary delta | [roadmap](roadmap/README.md) |
| Reconciliation loop | A controller continuously compares desired state with observed state and acts to close the gap | Kubernetes, Argo CD, Terraform |
| Desired state | The declared target configuration, stored in Git, etcd or a state file | IaC, GitOps |
| Drift | Divergence between desired state and the real world, detected by plan/diff or audit | Terraform, CloudFormation, Argo CD |
| DAG | Directed acyclic graph; the execution order of resources, jobs or agent nodes | Terraform, CI/CD, LangGraph |
| OIDC workload identity | Short-lived tokens issued by an identity provider (GitHub, Kubernetes) exchanged for cloud credentials, replacing static keys | GitHub Actions, IRSA, Workload Identity, Cosign |
| Golden path | A paved, supported, self-service way to build and ship a service, usually via a Backstage template | Backstage |
| Shift left | Running security and quality checks as early as possible in the delivery flow | DevSecOps |
| Policy as code | Rules expressed in a machine-checkable language (Rego, YAML, Python, HCL) and enforced automatically | Checkov, Kyverno, OPA |
| Zero trust | Never trust by network location; authenticate and authorise every request with identity and least privilege | NIST 800-207 |

## Cloud (01, 02)

| Term | Meaning |
|---|---|
| Region / Availability Zone | Geographic area / isolated data-centre group inside it; multi-AZ is the baseline for HA |
| SCP | Service Control Policy; organisation-level guardrail limiting what accounts may do |
| IRSA / Pod Identity | Mechanisms binding a Kubernetes ServiceAccount to an AWS IAM role |
| Managed Identity | Azure identity attached to a resource so it can authenticate without secrets |
| Workload Identity Federation | GCP (and Azure) mechanism to trust external OIDC tokens |
| VPC endpoint / PrivateLink / Private Endpoint / Private Service Connect | Private connectivity to a cloud service without traversing the public internet |
| CMEK | Customer-managed encryption key held in the provider's KMS |
| PTU | Provisioned Throughput Unit; reserved model capacity in Azure OpenAI |
| Inference profile | Bedrock routing construct enabling cross-region inference |
| Guardrail | Bedrock feature that filters topics, content and PII in prompts and responses |
| Knowledge Base | Bedrock managed RAG over S3 documents with a vector store |

## Containers, IaC, CI/CD (04, 05, 06)

| Term | Meaning |
|---|---|
| OCI image | Standard container image format: layers plus a manifest and config, addressed by digest |
| Distroless | Minimal base image with no shell or package manager |
| BuildKit | Docker's build engine supporting cache mounts, secret mounts, multi-arch and attestations |
| Pod / Deployment / StatefulSet / DaemonSet | Kubernetes workload primitives for stateless replicas, stable identity, one-per-node |
| Gateway API | Successor to Ingress: GatewayClass, Gateway, HTTPRoute resources |
| NetworkPolicy | Kubernetes L3/L4 firewall rules selecting pods |
| HPA / VPA / Karpenter | Horizontal pod autoscaler / vertical recommender / node autoprovisioner |
| Pod Security Admission | Built-in admission enforcing privileged, baseline or restricted profiles |
| Admission controller | Webhook that validates or mutates API requests before persistence |
| CRD / Operator | Custom resource type / controller that manages it |
| Helm chart / release | Templated Kubernetes package / an installed instance of it |
| GitOps | Git as the source of truth, with an agent pulling and reconciling cluster state |
| App of Apps / ApplicationSet | Argo CD patterns for managing many Applications |
| Terraform state | JSON mapping of resource addresses to real IDs; must be remote and locked |
| Module | Reusable Terraform unit with inputs and outputs |
| Change set | CloudFormation preview of a stack update |
| Stack (Pulumi) | An isolated instance of a program's resources with its own config and state |
| Runner / agent | Ephemeral machine or container executing CI jobs |
| Reusable workflow / composite action / shared library / template | Reuse mechanisms in GitHub Actions / Jenkins / Azure Pipelines |
| SHA pinning | Referencing an action by commit hash to prevent tag hijacking |

## Backend (07)

| Term | Meaning |
|---|---|
| ASGI | Async Python server interface used by FastAPI via uvicorn |
| Dependency injection | Supplying collaborators from outside (FastAPI `Depends`, Spring IoC) |
| Pydantic | Python data validation via type hints |
| Virtual threads | JDK 21 lightweight threads making blocking I/O cheap |
| Actuator | Spring Boot operational endpoints: health, metrics, info |
| Liveness / readiness | Probes answering "restart me?" and "send me traffic?" |
| ProblemDetail | RFC 9457 standard error body |
| Testcontainers | Library spinning up real dependencies in Docker for tests |

## DevSecOps (08)

| Term | Meaning |
|---|---|
| SAST / DAST / IAST | Static analysis of source / dynamic testing of a running app / instrumentation-based testing |
| SCA | Software composition analysis: known vulnerabilities in dependencies |
| SBOM | Software bill of materials (CycloneDX or SPDX) listing every component |
| VEX | Vulnerability Exploitability eXchange: statements that a CVE does or does not affect a product |
| Taint analysis | Tracking untrusted input (source) to a dangerous call (sink) through the code |
| Quality gate | SonarQube pass/fail conditions on new code |
| CVE / CVSS / CWE | Vulnerability identifier / severity score / weakness category |
| SLSA | Supply-chain Levels for Software Artifacts; provenance and build integrity levels |
| Keyless signing | Cosign signing with a short-lived certificate bound to an OIDC identity, logged in Rekor |
| Attestation | Signed statement (SBOM, provenance) attached to an image |
| Admission policy | Kyverno or Gatekeeper rule applied at `kubectl apply` time |
| eBPF | Kernel technology letting Falco observe syscalls safely without kernel modules |
| Dynamic secret | Vault-issued credential created on demand with a TTL |
| ASPM | Application security posture management; aggregation and triage hub (DefectDojo) |
| SIEM / XDR | Security event correlation / extended detection and response (Wazuh) |
| SARIF | Static analysis results interchange format consumed by GitHub code scanning and DefectDojo |

## Observability and SRE (09, 11)

| Term | Meaning |
|---|---|
| OTLP | OpenTelemetry protocol for traces, metrics and logs |
| Collector | OTel process with receivers, processors and exporters |
| Span / trace | A timed operation / a tree of spans sharing a trace_id |
| Exemplar | A trace_id attached to a metric sample enabling metric-to-trace jumps |
| Cardinality | Number of unique label combinations; the main cost driver |
| PromQL / LogQL / TraceQL / KQL | Query languages for Prometheus / Loki / Tempo / Kibana |
| Recording rule | Precomputed PromQL expression stored as a new series |
| RED / USE / Four Golden Signals | Rate, errors, duration / utilisation, saturation, errors / latency, traffic, errors, saturation |
| SLI / SLO / SLA | Measured indicator / internal objective / contractual agreement |
| Error budget | 1 minus the SLO, the allowed unreliability in a window |
| Burn rate | How fast the error budget is being consumed relative to a sustainable 1.0× |
| Multi-window alert | Burn-rate alert requiring both a long and a short window to exceed thresholds |
| Toil | Manual, repetitive, automatable operational work; capped at 50% |
| Runbook / postmortem | Step-by-step response doc for an alert / blameless analysis after an incident |
| Canary / blue-green | Progressive traffic shift to a new version / instant switch between two environments |
| DORA metrics | Deployment frequency, lead time, change failure rate, time to restore |

## AI platform (03, 10)

| Term | Meaning |
|---|---|
| Agent | LLM in a loop with tools and state, deciding its next action |
| ReAct | Reason-then-act prompting pattern underlying most agents |
| Checkpointer | LangGraph persistence layer saving graph state per step |
| Human-in-the-loop | Pausing an agent for approval before continuing |
| Tool calling | Model emits a structured call that the runtime executes |
| MCP | Model Context Protocol: JSON-RPC standard exposing Resources, Prompts and Tools to any client |
| RAG | Retrieval-augmented generation: ground answers in retrieved documents |
| Chunking | Splitting documents into retrievable pieces |
| Embedding | Dense vector representing semantic meaning |
| Hybrid search | Combining dense vector similarity with sparse lexical (BM25) scoring |
| RRF | Reciprocal Rank Fusion: merges ranked lists by summing 1/(k + rank) |
| Reranker | Cross-encoder scoring query-document pairs to reorder candidates |
| HNSW / IVF | Graph-based / clustering-based approximate nearest neighbour indexes |
| Quantisation | Compressing vectors (scalar, binary, product) to save memory |
| PagedAttention | vLLM's virtual-memory style KV-cache management |
| Continuous batching | Adding new requests to a running batch between decode steps |
| TTFT | Time to first token; the latency SLI for streaming LLM responses |
| MIG | Multi-Instance GPU partitioning on NVIDIA A100/H100 |
| DCGM | NVIDIA Data Center GPU Manager; source of GPU Prometheus metrics |
| Eval | Automated test of an AI system's quality or safety |
| LLM-as-judge / G-Eval | Using a model with a rubric to grade outputs |
| Faithfulness | Degree to which an answer is supported by retrieved context |
| Red teaming | Adversarial testing for prompt injection, jailbreaks and data leakage |
| OWASP Top 10 for LLM Applications | The standard list of LLM application risks, including prompt injection and excessive agency |

## Enterprise ops (12)

| Term | Meaning |
|---|---|
| RPO / RTO | Maximum tolerable data loss / maximum tolerable downtime |
| Pilot light / warm standby / active-active | DR tiers of increasing cost and decreasing RTO |
| DiRT | Disaster Recovery Testing: deliberate failure drills |
| FinOps | Cultural and technical practice of cloud financial management: Inform, Optimize, Operate |
| Unit economics | Cost per request, customer, token or task |
| Showback / chargeback | Reporting cost to teams / billing teams for it |
| FOCUS | FinOps Open Cost and Usage Specification for billing data |
| NIST CSF 2.0 | Cybersecurity Framework functions: Govern, Identify, Protect, Detect, Respond, Recover |
| CIS Benchmark | Consensus hardening configuration baselines |
| SOC 2 / ISO 27001 | Common assurance and security management standards |
