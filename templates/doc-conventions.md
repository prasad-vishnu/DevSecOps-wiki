# Documentation Conventions

These conventions apply to every Markdown file in this wiki. They exist so that a reader who learns how one tool's folder is laid out can navigate every other tool's folder without thinking.

---

## 1. Repository Layout

```
DevSecOps-wiki/
├── README.md                      # Master index and how to use the wiki
├── roadmap/                       # Source PDF + the roadmap rewritten as Markdown
├── capstone/                      # The unified capstone repository architecture
├── templates/                     # This file, ADR / runbook / postmortem templates
├── 00-foundations/                # Linux, networking, Git, Bash
├── 01-cloud/                      # AWS (anchor), Azure, GCP
├── 02-managed-ai-platforms/       # Bedrock (anchor), Vertex AI, Azure OpenAI
├── 03-agentic-frameworks/         # LangChain, LangGraph (anchor), Strands Agents (anchor)
├── 04-containers-and-idp/         # Docker, Kubernetes (anchor), Helm, Argo CD, Backstage
├── 05-infrastructure-as-code/     # Terraform (anchor), CloudFormation, Pulumi
├── 06-cicd/                       # GitHub Actions (anchor), Jenkins, Azure DevOps
├── 07-backend/                    # Python, FastAPI (anchor), Java, Spring Boot (anchor)
├── 08-devsecops/                  # 20 tools organised by the 7 pipeline security gates
├── 09-observability/              # OpenTelemetry (anchor), Prometheus, Grafana, Loki, Tempo, ELK
├── 10-ai-infra-rag-mcp/           # GPU on K8s, vLLM, RAG, vector DBs, MCP, evals
├── 11-sre/                        # SLI/SLO, burn-rate alerting, on-call, postmortems
└── 12-enterprise-ops/             # Production support, DR, FinOps, cybersecurity governance
```

Cluster numbers match the twelve lines of the roadmap. `00-foundations` is a prerequisite layer added for completeness.

---

## 2. Per-Tool Folder: The Standard Kit

Every tool folder contains exactly these files, in this order. Use the same names everywhere.

| File | Purpose | Target length |
|---|---|---|
| `README.md` | What it is, why it exists, architecture diagram, core mental model, anchor-and-bridge mapping, learning path, index of the other files | 150-300 lines |
| `01-installation-and-setup.md` | Install on Linux/macOS/Windows/containers, first-run configuration, authentication, verification commands | 150-300 lines |
| `02-core-concepts.md` | The vocabulary and internals. Explain *how it works*, not just what the buttons do | 250-500 lines |
| `03-commands-and-cheatsheet.md` | Exhaustive CLI / API / config reference organised by task. Every command has a one-line explanation | 250-500 lines |
| `04-hands-on-labs.md` | Progressive labs (Lab 1 beginner to Lab N advanced). Each lab has: objective, prerequisites, steps with full commands, expected output, verification, cleanup | 300-600 lines |
| `05-production-patterns.md` | Security hardening, high availability, scaling, cost, CI/CD integration, GitOps, multi-tenant, anti-patterns, decision checklist | 200-400 lines |
| `06-troubleshooting.md` | Symptom to cause to fix tables, diagnostic commands, common error messages verbatim, debugging workflow | 150-300 lines |
| `07-interview-questions.md` | 25+ questions spanning junior to principal level, each with a model answer | 150-300 lines |

Cluster-level `README.md` files (for example `01-cloud/README.md`) contain:

1. Cluster purpose and the core unifying pattern from the roadmap.
2. A comparison matrix of the sibling tools.
3. The anchor-and-bridge delta mapping table (vocabulary translation between tools).
4. Recommended learning order and time budget.
5. Links to each tool folder.

---

## 3. Writing Rules

1. **Lead with the mental model.** Before listing commands, explain the one or two ideas that make the tool make sense (for example "Kubernetes is a reconciliation loop over desired state stored in etcd").
2. **Every command is runnable.** Use real flags. Show expected output where it helps. Prefer long-form flags in docs (`--namespace` not `-n`) except in cheatsheets where both are shown.
3. **Placeholders are obvious.** Use `<ACCOUNT_ID>`, `<REGION>`, `<BUCKET_NAME>`, `example.com`, `123456789012`. Never use anything that looks like a real credential, key, token, or personal data.
4. **Code blocks declare a language.** ` ```bash `, ` ```yaml `, ` ```hcl `, ` ```python `, ` ```java `, ` ```json `, ` ```promql `, ` ```rego `, ` ```sql `, ` ```dockerfile `, ` ```mermaid `.
5. **Diagrams are Mermaid or ASCII.** GitHub renders Mermaid natively. Use `flowchart`, `sequenceDiagram`, and `stateDiagram-v2`.
6. **Tables for comparisons.** Prose for explanation. Bullets for parallel lists. Numbered lists for ordered steps.
7. **Tie every tool back to the capstone.** Each `README.md` ends with a "Role in the Capstone" section describing how the tool plugs into the unified capstone repository.
8. **Anchor-and-bridge.** Anchor tools are documented first and most deeply. Bridge tools include a "Coming from <anchor>" section that maps concepts one-to-one.
9. **Version awareness.** Mention the major version the docs target (for example "Kubernetes 1.31+", "Terraform 1.9+", "Python 3.12+", "Java 21 LTS", "Spring Boot 3.3+"). Avoid pinning patch versions in prose; pin them in reproducible lab manifests.
10. **Security by default.** Every example uses least privilege, non-root containers, OIDC rather than static keys, encryption at rest and in transit, and short-lived credentials. Call out insecure shortcuts explicitly when they appear in a lab for simplicity.
11. **No marketing.** Describe trade-offs honestly, including where the tool is weak.
12. **Relative links.** Link between docs with relative paths so the wiki works on GitHub, in Backstage TechDocs, and in a local Markdown viewer.

---

## 4. Front-Matter Block

Every file starts with a title and a short metadata table:

```markdown
# <Tool>: <File Topic>

| | |
|---|---|
| **Cluster** | 04 Containers & IDP |
| **Role** | Anchor |
| **Prerequisites** | Docker fundamentals, Linux networking |
| **Target version** | Kubernetes 1.31+ |
| **Capstone phase** | Phase 3 (Months 9-12) |

> One-paragraph summary of what the reader will be able to do after this file.
```

---

## 5. Lab Format

```markdown
## Lab 3: Zero-Downtime Rolling Update

**Objective.** Deploy v2 of the FastAPI service with no failed requests.

**Prerequisites.** Lab 2 complete; `kubectl` context pointing at the kind cluster.

**Time.** 25 minutes.

### Steps

1. ...

   ```bash
   kubectl set image deployment/api api=ghcr.io/example/api:v2 --record
   ```

   Expected output:

   ```text
   deployment.apps/api image updated
   ```

### Verify

...

### Cleanup

...

### What you learned

...
```

---

## 6. Interview Question Format

```markdown
### Q12. How does Kubernetes decide which node runs a Pod? (Mid-level)

**Answer.** The scheduler runs two phases: filtering (predicates) ...
```

---

## 7. Troubleshooting Table Format

| Symptom | Likely cause | Diagnose | Fix |
|---|---|---|---|
| `ImagePullBackOff` | Wrong image tag or missing registry secret | `kubectl describe pod <pod>` and read Events | Fix tag, create `imagePullSecret`, attach to ServiceAccount |

---

## 8. Architecture Decision Records

Use [adr-template.md](adr-template.md). Store ADRs under `capstone/adrs/`. One decision per file, named `NNNN-short-title.md`.
