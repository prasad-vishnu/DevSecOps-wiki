# Capstone Repository Layout

> The directory tree of the capstone **code** repository (the "Monorepo Lab"). This wiki documents the tools; the capstone repo is where they run. Keep the two in separate Git repositories so TechDocs can index this wiki while CI runs against the capstone.

```text
platform-capstone/
├── .github/
│   ├── CODEOWNERS
│   ├── dependabot.yml                     # actions, pip, maven, docker, terraform ecosystems
│   ├── workflows/
│   │   ├── ci-fastapi.yml                 # uv sync, ruff, mypy, pytest, coverage -> Sonar
│   │   ├── ci-springboot.yml              # mvn verify, JaCoCo, Testcontainers
│   │   ├── devsecops-gates.yml            # reusable: the 7 gates (called by both CI workflows)
│   │   ├── build-sign-publish.yml         # buildx multi-arch, SBOM, trivy, grype, cosign sign+attest
│   │   ├── terraform-plan.yml             # on PR: fmt, validate, tflint, checkov, plan -> PR comment, infracost
│   │   ├── terraform-apply.yml            # on merge to main: apply with OIDC role, env approval
│   │   ├── nightly-destroy.yml            # cron 0 2 * * *: terraform destroy lab workspaces
│   │   ├── evals.yml                      # deepeval test run, ragas, promptfoo eval; blocks merge
│   │   ├── zap-review-env.yml             # kind-in-CI, deploy, zap-api-scan, upload to DefectDojo
│   │   └── techdocs-publish.yml           # build mkdocs for Backstage TechDocs
│   └── actions/
│       └── defectdojo-upload/             # composite action: reimport-scan with auto-create-context
│
├── services/
│   ├── incident-api-python/               # FastAPI service (Phase 1) -> Incident Agent (Phase 6)
│   │   ├── app/
│   │   │   ├── main.py                    # lifespan, routers, middleware, OTel
│   │   │   ├── api/                       # routers: incidents, health, agent
│   │   │   ├── core/                      # settings (pydantic-settings), logging, security (OIDC JWKS)
│   │   │   ├── db/                        # SQLAlchemy 2.0 async models, session, migrations (alembic)
│   │   │   ├── agent/                     # LangGraph graph / Strands agent, tools, prompts
│   │   │   └── rag/                       # ingest, chunk, embed, Qdrant hybrid retriever, reranker
│   │   ├── tests/                         # pytest, httpx AsyncClient, Testcontainers Postgres
│   │   ├── evals/                         # deepeval test_*.py, ragas datasets, promptfooconfig.yaml
│   │   ├── Dockerfile                     # multi-stage uv -> distroless, non-root
│   │   ├── pyproject.toml
│   │   ├── uv.lock
│   │   ├── catalog-info.yaml              # Backstage Component + API entities
│   │   └── mkdocs.yml                     # TechDocs for this service
│   └── incident-api-java/                 # Spring Boot 3.3 / Java 21 service
│       ├── src/main/java/com/example/incident/
│       │   ├── IncidentApplication.java
│       │   ├── api/                       # @RestController, ProblemDetail advice
│       │   ├── domain/                    # entities, records
│       │   ├── repo/                      # Spring Data JPA
│       │   └── config/                    # security (OAuth2 resource server), OTel, virtual threads
│       ├── src/main/resources/
│       │   ├── application.yaml           # actuator probes, graceful shutdown, management port
│       │   ├── db/migration/              # Flyway
│       │   └── logback-spring.xml         # JSON logs with trace_id
│       ├── src/test/java/                 # JUnit 5, Testcontainers @ServiceConnection
│       ├── Dockerfile                     # multi-stage, layered jar, Temurin JRE, ZGC
│       ├── pom.xml
│       └── catalog-info.yaml
│
├── mcp-servers/
│   ├── prometheus-mcp/                    # FastMCP: query_instant, query_range, list_alerts, get_slo_burn_rate
│   ├── kubernetes-mcp/                    # FastMCP: list_pods, get_logs, get_events, rollout_restart (destructive)
│   └── runbooks-mcp/                      # FastMCP: search_runbooks over Qdrant
│
├── infra/
│   ├── terraform/
│   │   ├── bootstrap/                     # S3 state bucket + lock, GitHub OIDC provider + roles (run once)
│   │   ├── modules/
│   │   │   ├── vpc/
│   │   │   ├── eks/                       # cluster, access entries, Karpenter, add-ons, GPU NodePool
│   │   │   ├── rds-postgres/
│   │   │   ├── s3-secure-bucket/
│   │   │   ├── ecr/
│   │   │   └── bedrock-knowledge-base/
│   │   ├── envs/
│   │   │   ├── lab/                       # backend.tf, main.tf, terraform.tfvars (ttl tags)
│   │   │   └── prod-sim/
│   │   └── tests/                         # *.tftest.hcl
│   ├── pulumi/                            # Python stack mirroring envs/lab for one service (Phase 2 bridge)
│   ├── cloudformation/                    # YAML template mirroring one service (Phase 2 bridge)
│   └── policies/
│       ├── checkov/                       # custom YAML/Python checks (mandatory tags, no public S3)
│       └── opa/                           # conftest policies for Terraform plan JSON
│
├── gitops/                                # Argo CD watches this tree
│   ├── bootstrap/
│   │   ├── argocd/                        # Argo CD install values, app-of-apps root Application
│   │   └── projects/                      # AppProject per team
│   ├── platform/
│   │   ├── kyverno/                       # install + ClusterPolicies (PSS restricted, verifyImages)
│   │   ├── external-secrets/
│   │   ├── cert-manager/
│   │   ├── argo-rollouts/
│   │   ├── observability/                 # kube-prometheus-stack, loki, tempo, otel-collector, grafana
│   │   ├── security/                      # falco, trivy-operator, vault
│   │   ├── ai-infra/                      # gpu-operator, vllm, qdrant, mcp-servers
│   │   └── backstage/
│   └── apps/
│       ├── incident-api-python/
│       │   ├── base/                      # Helm chart reference or kustomize base
│       │   └── overlays/{dev,stage,prod}/ # image digest, replicas, Rollout canary steps
│       └── incident-api-java/
│
├── charts/
│   ├── incident-api/                      # shared production chart: Rollout/Deployment, Service, HTTPRoute,
│   │                                      # HPA, PDB, NetworkPolicy, ServiceMonitor, Vault annotations, values.schema.json
│   └── mcp-server/
│
├── observability/
│   ├── docker-compose.lgtm.yaml           # local Grafana, Prometheus, Loki, Tempo, OTel Collector
│   ├── otel-collector/config.yaml
│   ├── prometheus/
│   │   ├── rules/slo-incident-api.yaml    # recording + multi-window burn-rate alerts
│   │   └── tests/slo-incident-api.test.yaml  # promtool unit tests
│   ├── alertmanager/alertmanager.yaml
│   └── grafana/
│       ├── provisioning/                  # datasources, dashboards providers
│       └── dashboards/                    # RED, SLO, cluster, vLLM, token cost (JSON)
│
├── security/
│   ├── semgrep/rules/                     # custom org rules + tests
│   ├── gitleaks/.gitleaks.toml
│   ├── trivy/.trivyignore.yaml
│   ├── zap/zap.yaml                       # Automation Framework plan for the API scan
│   ├── nuclei/templates/                  # custom templates for actuator/debug endpoint exposure
│   ├── falco/rules/falco_rules.local.yaml
│   ├── kyverno/tests/                     # kyverno-test.yaml per policy
│   └── vault/
│       ├── policies/*.hcl
│       └── terraform/                     # Vault provider config-as-code (auth methods, roles, engines)
│
├── backstage/
│   ├── templates/
│   │   ├── fastapi-service/template.yaml  # golden path: scaffold service + CI + chart + Argo app + catalog
│   │   └── springboot-service/template.yaml
│   ├── catalog/                           # Domains, Systems, Groups, Users, Resources, Locations
│   └── app-config.production.yaml
│
├── docs/                                  # TechDocs root for the capstone (mkdocs.yml at repo root)
│   ├── adrs/                              # Architecture Decision Records (ADR-0001...)
│   ├── runbooks/                          # one file per alert; indexed into Qdrant by the agent
│   ├── postmortems/
│   └── slo/                               # SLO documents per service
│
├── scripts/
│   ├── kind-up.sh                         # kind cluster with Gateway API CRDs, local registry, ingress
│   ├── bootstrap-argocd.sh
│   ├── dr-drill-az-failure.sh             # drains one AZ's nodes, records RPO/RTO
│   └── cost-report.sh                     # OpenCost + CUR summary
│
├── .pre-commit-config.yaml                # gitleaks, ruff, semgrep, terraform fmt/validate, checkov, shellcheck
├── mkdocs.yml
├── Makefile                               # make up / test / scan / deploy / destroy
├── SECURITY.md
├── CONTRIBUTING.md
└── README.md
```

## Conventions Inside the Capstone

| Convention | Rule |
|---|---|
| Branching | Trunk-based; short-lived branches; squash merge; signed commits (gitsign or SSH) |
| Versioning | Conventional Commits; images tagged `sha-<short>` and referenced by digest in GitOps overlays |
| Secrets | None in Git. Vault for runtime, GitHub OIDC for CI, SOPS only for non-sensitive config if needed |
| Environments | `dev` on kind, `stage` and `prod-sim` on ephemeral EKS; promotion by PR to the overlay |
| Ownership | `CODEOWNERS` mirrors Backstage `spec.owner` |
| Docs | Every service has `mkdocs.yml`; every alert has a runbook; every tool choice has an ADR |
