# ADR-0001: Terraform as the IaC anchor; Pulumi and CloudFormation as bridges

| | |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-10-27 |
| **Deciders** | Platform lead |
| **Cluster** | 05 Infrastructure as Code |
| **Capstone phase** | Phase 2 |

## Context

The capstone needs to provision a multi-AZ VPC, an EKS cluster, an RDS Postgres instance, S3 buckets with KMS, ECR repositories and later a Bedrock Knowledge Base and a GPU node pool. The same engineer must also be able to read infrastructure code in enterprises that standardised on CloudFormation (AWS-only shops) or Pulumi (platform teams embedding IaC in Python/TypeScript). The roadmap requires one anchor to be learned deeply so the "declarative DAG plus state" pattern is internalised, after which sibling tools are mapped by delta.

Forces:

- **Breadth of providers.** The lab touches AWS primarily but also Azure, GCP, Vault, Grafana, Kubernetes and GitHub. A single tool that drives all of them reduces context switching.
- **State semantics.** Learners must understand remote state, locking, drift and import because those concepts recur in Argo CD, Kubernetes controllers and CloudFormation stacks.
- **Hiring market.** Terraform remains the most requested IaC skill in platform and SRE job descriptions; CloudFormation and Pulumi are frequent secondary requirements.
- **Licensing.** Terraform moved to the BSL in 2023; OpenTofu is the Linux Foundation fork with HCL compatibility. The decision must not strand the learner if an employer mandates OpenTofu.
- **Testing and policy.** Checkov, tflint and `terraform test` provide a complete shift-left story that the DevSecOps phase reuses.

## Decision

We will use **Terraform (HCL) as the anchor IaC tool** for all capstone infrastructure, with remote state in S3 using native state locking, modules per concern, and GitHub Actions OIDC for authentication. We will implement **one service's infrastructure twice more**, once in **Pulumi (Python)** and once in **CloudFormation**, purely as bridge exercises, and keep those stacks out of the production path.

Terraform's provider ecosystem covers every system in the capstone (AWS, Azure, GCP, Kubernetes, Helm, Vault, Grafana, GitHub), its plan/apply graph makes the DAG pattern visible via `terraform graph`, and its state model forces the learner to confront drift, import and locking explicitly. Everything learned transfers: Pulumi uses the same provider schemas through its bridge, and CloudFormation resources map one-to-one to Terraform resources.

## Alternatives Considered

| Option | Pros | Cons | Why rejected (or deferred) |
|---|---|---|---|
| Pulumi (Python) as anchor | Real language: loops, classes, unit tests; same Python as FastAPI; strong Backstage plugin | Smaller job market; adds Pulumi Cloud or self-managed backend; abstractions can hide the DAG/state pattern behind code | Chosen as bridge. Excellent second tool once state semantics are understood |
| CloudFormation / CDK as anchor | AWS-managed state and rollbacks; no backend to run; CDK gives constructs | AWS-only; cannot manage Vault, Grafana, Kubernetes resources, Azure or GCP; slow stack operations; drift detection is partial | Chosen as bridge for AWS-only fluency |
| OpenTofu as anchor | MIT-licensed, HCL compatible, state encryption built in | Smaller ecosystem of tutorials and CI actions; provider parity lags occasionally | Deferred. All HCL in this capstone must stay OpenTofu-compatible so switching is a one-line change in CI |
| Crossplane (Kubernetes-native IaC) | Everything in Kubernetes; GitOps-native | Requires a cluster before you have a cluster; steep learning curve; conflates platform and app lifecycles for beginners | Rejected for the anchor; revisit for platform APIs in Phase 3 if time permits |

## Consequences

### Positive

- One mental model (resources, state, plan diff, DAG) learned once and mapped three times.
- Policy-as-code (Checkov) and testing (`terraform test`) reuse directly in Phase 5.
- OIDC-based CI auth eliminates static cloud keys from day one.

### Negative / Risks

- HCL's limited expressiveness pushes complex logic into `for` expressions and `dynamic` blocks that are hard to read. Mitigation: keep modules small, use Pulumi where real logic is needed.
- BSL licensing could matter to some employers. Mitigation: OpenTofu compatibility is a standing constraint; test the pipeline with `tofu` once per phase.
- State contains secrets (RDS passwords if generated). Mitigation: state bucket encrypted with KMS, access only via the CI role and a break-glass admin role; prefer `ephemeral` values and Secrets Manager data sources.

### Neutral / Follow-ups

- Write ADR for Terragrunt vs plain Terraform if environment count exceeds three.
- Evaluate HCP Terraform remote runs if team size grows past one.

## Compliance and Security Impact

All infrastructure changes are reviewed via PR with a visible plan, scanned by Checkov against CIS benchmarks, and applied by a short-lived OIDC identity. Audit evidence is the PR, the plan output, and CloudTrail entries attributed to the CI role session.

## Cost Impact

Terraform itself is free. The S3 state bucket costs cents. The real cost lever is the nightly destroy workflow, which Terraform makes trivial (`terraform destroy -auto-approve` on the lab workspace).

## Validation

Success looks like: a new engineer can read the Pulumi and CloudFormation versions of the service stack and explain each resource by reference to its Terraform equivalent within one hour. Review at the end of Phase 2.

## References

- [../../05-infrastructure-as-code/README.md](../../05-infrastructure-as-code/README.md)
- [../../05-infrastructure-as-code/terraform/](../../05-infrastructure-as-code/terraform/)
- Roadmap Cluster 5 table: Terraform vs CloudFormation vs Pulumi
