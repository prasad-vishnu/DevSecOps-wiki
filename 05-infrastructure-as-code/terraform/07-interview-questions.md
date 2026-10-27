# Terraform: Interview Questions

| | |
|---|---|
| **Cluster** | 05 Infrastructure as Code |
| **Role** | Anchor |
| **Prerequisites** | All previous Terraform files |
| **Target version** | Terraform 1.9+ |
| **Capstone phase** | Phase 2 (Months 5-8) |

> Thirty questions from junior to principal level, each with a model answer that an interviewer would consider complete. Use them to check your understanding after the labs; the principal-level answers are the ones that distinguish someone who has operated Terraform from someone who has used it.

---

## Junior

### Q1. What problem does Terraform solve, and what are its three main components? (Junior)

**Answer.** Terraform makes infrastructure declarative, versioned, and repeatable: you describe the desired end state and Terraform computes and executes the API calls to reach it. Its components are the **core** (parses HCL, builds the dependency graph, plans, manages state), **providers** (plugins that map resource schemas to a specific API such as AWS or Kubernetes), and **state** (a record of the resources Terraform manages, mapping configuration addresses to real IDs and attributes).

### Q2. What is the difference between `terraform plan` and `terraform apply`? (Junior)

**Answer.** `plan` computes the diff between configuration, state, and the refreshed real world and prints the proposed create/update/replace/destroy actions without changing anything. `apply` executes those actions. Saving the plan with `-out` and applying that file guarantees apply does exactly what was reviewed.

### Q3. Why is the state file important and why should it be remote? (Junior)

**Answer.** State is how Terraform knows which real resources correspond to which configuration blocks; without it Terraform would try to create everything again. Remote state (S3, GCS, Azure Blob, HCP Terraform) lets a team and CI share one source of truth, adds locking so two applies cannot run concurrently, and provides encryption and versioning. Local state is only for experiments.

### Q4. What does `terraform init` do? (Junior)

**Answer.** It initialises the working directory: configures the backend, downloads the providers that satisfy `required_providers` constraints (recording exact versions and checksums in `.terraform.lock.hcl`), and downloads modules. It is safe to run repeatedly.

### Q5. What is the difference between a resource and a data source? (Junior)

**Answer.** A `resource` is something Terraform creates, updates, and destroys. A `data` source is a read-only lookup of something that already exists (an AMI, an account ID, a parameter) that is refreshed on every plan and whose result can be referenced by resources.

### Q6. What is a module? (Junior)

**Answer.** A module is a directory of Terraform configuration that can be called from another configuration with inputs (variables) and returns outputs. Every working directory is a root module; child modules are called with `module` blocks and sourced from local paths, Git, or a registry. They are the unit of reuse and composition.

### Q7. Explain the plan symbols `+`, `~`, `-`, and `-/+`. (Junior)

**Answer.** `+` create, `~` update in place, `-` destroy, `-/+` replace by destroying then creating (shown as `+/-` when `create_before_destroy` is set). An attribute annotated `# forces replacement` is the cause of a replace.

### Q8. How do you pass variables to Terraform? (Junior)

**Answer.** Defaults in the `variable` block, `TF_VAR_<name>` environment variables, `terraform.tfvars` or `*.auto.tfvars` files loaded automatically, `-var-file=` and `-var` flags. Later sources override earlier ones; command-line flags win.

---

## Mid-level

### Q9. When would you use `count` versus `for_each`, and why does the choice matter? (Mid-level)

**Answer.** `count` is appropriate for a boolean toggle (`count = var.enabled ? 1 : 0`) because instances are indexed by position. `for_each` is appropriate for collections because instances are keyed by stable strings. With `count` over a list, removing an element shifts every later index and Terraform plans to destroy and recreate those resources. With `for_each`, removing key `"b"` destroys only `["b"]`. `for_each` keys must be known at plan time, so they should derive from variables and locals rather than resource attributes.

### Q10. What is drift, how do you detect it, and what are your remediation options? (Mid-level)

**Answer.** Drift is any difference between recorded state and real resources caused outside Terraform. Detect it with `terraform plan -refresh-only -detailed-exitcode` (exit code 2 means drift), scheduled nightly in CI. Remediate by re-applying configuration to revert the change, updating configuration to match reality, or `apply -refresh-only` to accept it into state (usually with `ignore_changes` for attributes owned by another system). Prevent it with IAM that denies console writes to managed resources.

### Q11. Explain the `lifecycle` meta-argument's main settings. (Mid-level)

**Answer.** `prevent_destroy` makes any plan that would destroy the resource fail (though deleting the block itself bypasses it). `create_before_destroy` inverts replacement order so the new resource exists before the old is removed. `ignore_changes` lists attributes whose drift is tolerated after creation. `replace_triggered_by` forces replacement when referenced resources or attributes change. `precondition` and `postcondition` assert invariants before planning and after applying, with `self` available in postconditions.

### Q12. How does Terraform determine the order in which resources are created? (Mid-level)

**Answer.** It builds a directed acyclic graph where every expression reference (`aws_vpc.main.id`) is an edge, plus explicit `depends_on` edges. It walks the graph with up to ten parallel workers (`-parallelism`), starting any node whose dependencies are satisfied. Destroy walks the graph in reverse using dependency information stored in state. Cycles are errors (`terraform graph -draw-cycles` finds them).

### Q13. A colleague renamed a resource from `aws_subnet.a` to `aws_subnet.private["a"]` and the plan wants to destroy and recreate it. What do you do? (Mid-level)

**Answer.** Add a `moved { from = aws_subnet.a  to = aws_subnet.private["a"] }` block. The next plan shows the address move with zero resource changes. Apply, and once every environment has applied, delete the `moved` block. Before `moved` blocks existed you would use `terraform state mv`, which is not reviewable in a PR.

### Q14. How do you bring an existing manually created resource under Terraform management? (Mid-level)

**Answer.** Write an `import { to = <address>  id = <real id> }` block, optionally run `terraform plan -generate-config-out=generated.tf` to produce a starting resource block, trim it to intended arguments, run plan expecting "1 to import" with no other changes, then apply. The legacy `terraform import` command does the same without a plan.

### Q15. Does Terraform store secrets in state? How do you mitigate it? (Mid-level)

**Answer.** Yes, any secret that is an attribute of a resource or data source is stored in plaintext in state; `sensitive = true` only redacts CLI output. Mitigations in order: do not give Terraform the secret (RDS `manage_master_user_password`, IAM roles instead of keys), use ephemeral resources and write-only arguments (1.10/1.11+) so the value never persists, encrypt the backend with KMS and restrict access, never commit state or plan files, and let applications read secrets at runtime via IAM rather than through Terraform outputs.

### Q16. Compare workspaces with a directory-per-environment layout. (Mid-level)

**Answer.** Workspaces give one configuration multiple state files under the same backend and provider configuration, differentiated only by `terraform.workspace`, which makes the active environment invisible in code review and forces every environment to share versions. Directory per environment makes each environment a thin root module with its own backend, provider, and module versions; a PR touching `live/prod/` is obviously a production change. Workspaces suit ephemeral per-PR copies; directories suit long-lived environments.

### Q17. What is the dependency lock file and why commit it? (Mid-level)

**Answer.** `.terraform.lock.hcl` records the exact provider versions `init` selected and their checksums for each platform. Committing it makes every developer and CI run use identical provider builds, like `package-lock.json`. `init -upgrade` updates it deliberately; `providers lock -platform=...` adds checksums for other OS/architectures to avoid CI failures.

### Q18. How do you structure Terraform for multiple layers that depend on each other? (Mid-level)

**Answer.** Separate state per layer (network, data, compute) under `live/<env>/`, each consuming versioned modules. Share outputs through a neutral contract such as SSM Parameter Store paths (`/capstone/prod/network/vpc_id`) or, less preferably, `terraform_remote_state`. Apply layers in dependency order in CI. This bounds blast radius and lets layers have different cadences and permissions.

---

## Senior

### Q19. Design the CI/CD workflow for Terraform in a regulated environment. (Senior)

**Answer.** On pull request: fmt, validate, tflint, Checkov/Trivy on HCL, `terraform plan -out=tfplan -lock-timeout=5m` using an OIDC-federated read-mostly plan role, policy evaluation on `terraform show -json` output (Conftest/Checkov), Infracost diff, and a truncated plan comment with full artefacts uploaded. Merge requires CODEOWNERS review. On merge to `main`: a job gated by a protected environment with required reviewers assumes an apply role whose trust policy only accepts `sub` claims for `main`/the environment, re-plans (or downloads the reviewed plan) and applies with `-lock-timeout`. Concurrency groups serialise applies per environment. A nightly refresh-only job raises drift issues. Everything is pinned: CLI version, lock file read-only, module tags, action SHAs.

### Q20. Explain how S3 state locking works with the native lock file versus DynamoDB, and what happens when a run crashes. (Senior)

**Answer.** With `use_lockfile = true` (1.10+), Terraform writes `<key>.tflock` using an S3 conditional PUT (`If-None-Match: *`); a concurrent writer receives HTTP 412 and fails with lock info. With DynamoDB, it performs a conditional `PutItem` on `LockID`. If a run crashes without releasing, the lock persists; operators confirm no run is active (lock info shows who/when) and run `terraform force-unlock <ID>`. Native locking removes a service dependency and cost; both can run together during migration. State writes also carry a `serial` so a stale writer is rejected even without a lock.

### Q21. What is the OIDC trust model for GitHub Actions and how do you scope plan versus apply? (Senior)

**Answer.** GitHub's OIDC issuer signs a JWT per job with claims including `aud`, `repository`, `ref`, `environment`, and a composite `sub`. AWS IAM has an identity provider for `token.actions.githubusercontent.com`; a role's trust policy allows `sts:AssumeRoleWithWebIdentity` when `aud = sts.amazonaws.com` and `sub` matches a pattern. The plan role accepts `repo:ORG/REPO:*` and has read-only plus state access; the apply role accepts only `repo:ORG/REPO:ref:refs/heads/main` or `repo:ORG/REPO:environment:prod` and has write permissions bounded by a permissions boundary. No long-lived keys exist anywhere.

### Q22. How does `terraform test` work and how would you test a module at different levels? (Senior)

**Answer.** `.tftest.hcl` files contain `run` blocks that execute `plan` or `apply` against the module with specified `variables`, then evaluate `assert` conditions; `expect_failures` tests validation rules. `mock_provider` blocks (1.7+) return fabricated values so unit tests run without credentials in seconds. Integration runs use `command = apply` with a real provider and are automatically destroyed at the end of the file. Strategy: fast mocked unit tests on every PR for structure, counts, and validation; integration applies nightly or on module release; `examples/complete` as the fixture; `check` blocks for continuous validation after deploy.

### Q23. A plan shows `(known after apply)` for 40 resources after you added a `depends_on` to a module. Why, and what is the fix? (Senior)

**Answer.** `depends_on` on a module or data source tells Terraform the dependent cannot be evaluated until the dependency finishes applying, so every attribute read from it becomes unknown at plan time and cascades. The fix is to remove the explicit dependency and let data references create precise edges, or move the data source so it depends on a specific resource attribute. Reserve `depends_on` for hidden side-effect dependencies such as IAM policy attachment before use.

### Q24. How do you perform a zero-downtime replacement of a resource that others depend on, for example a launch template or an ACM certificate? (Senior)

**Answer.** Set `create_before_destroy = true` on the resource so Terraform creates the replacement, updates dependents to reference it, and only then destroys the old one. Avoid fixed `name` attributes that would collide (use `name_prefix`). Dependents inherit the setting. For databases, use a blue/green approach outside a single apply: create the new instance, migrate, switch the SSM contract, and remove the old one in a later change.

### Q25. Compare Terraform's licence situation and explain how you would keep OpenTofu as an option. (Senior)

**Answer.** Terraform moved to BSL 1.1 in 2023: free to use internally, prohibited for building a competing product. OpenTofu is the MPL-2.0 fork under the Linux Foundation, compatible with 1.5-era language and state and adding its own features (state encryption, variables in backends, provider `for_each`). To keep the option: avoid exclusive features on either side, keep `required_version` satisfiable by both, run `tofu plan` periodically in CI against a copy of state to confirm "no changes", and document the decision in an ADR.

---

## Principal

### Q26. Design the state architecture for a company with 30 teams, 4 environments, and 3 regions. (Principal)

**Answer.** Three tiers of state ownership. **Platform-global** (organisation, SCPs, identity, state backend, OIDC providers) owned by the platform team with two-person approval. **Per environment per region** platform layers (network, data, compute, platform add-ons) under `live/<env>/<region>/<layer>`, each its own state, applied by per-layer OIDC roles. **Per team** application layers (`live/<env>/<region>/apps/<team>/<service>`) created from a Backstage Golden Path template, consuming platform outputs via SSM contracts, applied by team-scoped roles with permission boundaries restricting them to their tagged resources. Backend is one versioned KMS-encrypted bucket per account with prefix-scoped policies, or one bucket per team for strict isolation. Terragrunt or HCP Terraform orchestrates dependency order; Renovate propagates module versions; a nightly drift matrix covers all layers. Blast radius: a team's mistake cannot reach another team's state or the platform; a platform change to `network` cannot touch `data`.

### Q27. A production apply failed halfway through creating an RDS instance; the pipeline shows a timeout, the console shows the instance "available", and state does not contain it. Walk through recovery. (Principal)

**Answer.** First, stop further pipeline runs (concurrency group or manual lock awareness). Diagnose: `terraform state list` confirms absence; `aws rds describe-db-instances` confirms existence; the cause was a provider timeout after AWS eventually succeeded. Recovery: add an `import` block for `module.postgres.aws_db_instance.this` with the identifier, run plan, confirm "1 to import" with no destroy, apply via the normal pipeline (fast-tracked review). Then raise the resource `timeouts` so it does not recur. Verify dependents (SSM exports, security groups) are consistent with a clean plan. Write a postmortem; consider a pipeline check that reconciles orphaned tagged resources against state weekly.

### Q28. How would you enforce "no public S3 buckets and no 0.0.0.0/0 ingress on port 22" across all Terraform in the company, and what are the trade-offs of each enforcement point? (Principal)

**Answer.** Layered enforcement. (1) **Pre-commit/PR static scan** with Checkov or Trivy: fastest feedback, but static analysis cannot resolve every expression. (2) **Plan-JSON policy** with Conftest/OPA or Checkov `terraform_plan`: sees resolved values and resource changes, blocks merges; requires a plan with real credentials and is per-repo unless centralised via reusable workflows. (3) **HCP Terraform Sentinel** with hard-mandatory level: centralised and unbypassable for runs through HCP, but only if all runs go through it. (4) **Cloud-native guardrails**: S3 account-level Block Public Access, SCPs denying `ec2:AuthorizeSecurityGroupIngress` with world CIDR on 22, AWS Config rules: catch everything including console changes and other IaC tools, but fail late (at apply) with worse developer experience. Use all four: static for speed, plan policy for precision, cloud guardrails as the backstop. Exceptions require a time-boxed, reviewed skip annotation.

### Q29. Explain how the Terraform plugin protocol influences what you see in a plan, and give an example of a provider-caused surprise. (Principal)

**Answer.** Core never decides whether a change is in-place or replacement; it sends prior state and proposed configuration to the provider's `PlanResourceChange` RPC and the provider returns the planned state plus `RequiresReplace` paths. Providers also normalise values during `ReadResource`, which is why perpetual diffs occur when your configuration's representation differs from the API's canonical form (JSON key order, case). Example: `aws_security_group` with inline `ingress` blocks is a single resource whose rule set is one attribute, so adding one rule shows as `~` on the whole set and can conflict with standalone rule resources; the provider's design, not core, dictates that. Knowing this directs debugging to provider documentation and changelogs rather than to Terraform itself.

### Q30. What are the limits of Terraform as a platform and when would you choose CloudFormation, Pulumi, or Crossplane instead? (Principal)

**Answer.** Terraform's weaknesses: state is an operational liability; no server-side rollback; HCL struggles with complex logic and abstraction; `apply` is a point-in-time operation with no continuous reconciliation; BSL licence. Choose **CloudFormation/CDK** when the estate is AWS-only and you want AWS-managed state, automatic rollback, StackSets across an organisation, and tight integration with AWS services like Service Catalog. Choose **Pulumi** when platform engineers are building infrastructure inside application code or an IDP, need real testing frameworks, or want to embed provisioning via the Automation API. Choose **Crossplane** (Kubernetes control plane) when you want continuous reconciliation, Kubernetes RBAC and GitOps as the operating model, and infrastructure consumed as custom resources by application teams. Most organisations run Terraform for the platform foundation and let these tools cover specific niches; the mental model (desired state, plan as diff, DAG) transfers to all of them.

---

## Rapid-Fire Checks

| Question | One-line answer |
|---|---|
| Default parallelism? | 10 |
| Where do non-default workspace states live in S3? | `env:/<workspace>/<key>` |
| Replace a resource without `taint`? | `terraform apply -replace='<address>'` |
| Exit code meaning changes present with `-detailed-exitcode`? | 2 |
| Block that forgets a resource without destroying it? | `removed { from = ... lifecycle { destroy = false } }` |
| Function to safely try an expression? | `try()` (returns fallback) / `can()` (returns bool) |
| Convert a list to a `for_each`-compatible value? | `toset()` or build a map |
| File that pins provider versions? | `.terraform.lock.hcl` |
| Reference the first element of a `count = 1` resource safely? | `one(aws_kms_key.db[*].arn)` |
| Terraform version that introduced `import` blocks? | 1.5 |
| Version that introduced `terraform test`? | 1.6 |
| Version that introduced ephemeral values and S3 `use_lockfile`? | 1.10 |

Back to [README.md](README.md).
