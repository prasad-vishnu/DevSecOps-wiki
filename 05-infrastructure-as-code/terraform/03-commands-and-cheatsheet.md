# Terraform: Commands and Cheatsheet

| | |
|---|---|
| **Cluster** | 05 Infrastructure as Code |
| **Role** | Anchor |
| **Prerequisites** | [02-core-concepts.md](02-core-concepts.md) |
| **Target version** | Terraform 1.9+ (OpenTofu: substitute `tofu`) |
| **Capstone phase** | Phase 2 (Months 5-8) |

> An exhaustive, task-organised reference for the Terraform CLI, the configuration blocks you reach for daily, backend configuration snippets, and the surrounding toolchain (tflint, Checkov, Trivy, Conftest, Terragrunt). Every command has a one-line explanation. Long-form flags are shown first; short forms where they exist.

---

## 1. Global Flags and Environment

| Command / variable | Explanation |
|---|---|
| `terraform -help` / `terraform <cmd> -help` | Help for any command |
| `terraform version` / `terraform version -json` | CLI and provider versions in use |
| `terraform -chdir=live/prod/network plan` | Run as if in that directory (preferred over `cd` in scripts) |
| `TF_LOG=TRACE TF_LOG_PATH=tf.log terraform plan` | Full debug log including provider RPCs |
| `TF_LOG_PROVIDER=DEBUG` / `TF_LOG_CORE=DEBUG` | Log only providers or only core |
| `TF_INPUT=0` | Never prompt (CI) |
| `TF_IN_AUTOMATION=1` | Hide "next steps" guidance |
| `TF_VAR_name=value` | Set root input variable `name` |
| `TF_CLI_ARGS="-no-color"` / `TF_CLI_ARGS_apply="-auto-approve"` | Default args globally or per subcommand |
| `TF_PLUGIN_CACHE_DIR=~/.terraform.d/plugin-cache` | Share provider binaries across directories |
| `TF_DATA_DIR=.terraform-prod` | Alternative `.terraform` directory |
| `TF_WORKSPACE=prod` | Select workspace non-interactively |
| `TF_REGISTRY_DISCOVERY_RETRY=3` / `TF_REGISTRY_CLIENT_TIMEOUT=30` | Registry resilience behind proxies |

---

## 2. Initialise

| Command | Explanation |
|---|---|
| `terraform init` | Download providers and modules, configure backend, write lock file |
| `terraform init -upgrade` | Allow newer provider/module versions within constraints; rewrites lock file |
| `terraform init -backend-config=env/prod.s3.tfbackend` | Supply backend settings from a file (`key = "..."` lines) |
| `terraform init -backend-config="key=live/prod/network.tfstate"` | Supply one backend setting inline |
| `terraform init -migrate-state` | Copy existing state to a newly configured backend |
| `terraform init -reconfigure` | Reinitialise backend, ignoring any saved configuration (no migration) |
| `terraform init -backend=false` | Skip backend init (for `validate` in CI without credentials) |
| `terraform init -input=false -no-color` | CI-friendly |
| `terraform init -lockfile=readonly` | Fail if the lock file would need changes (CI integrity check) |
| `terraform init -plugin-dir=/opt/providers` | Use only locally mirrored providers (air-gapped) |
| `terraform init -from-module=git::https://...//examples/complete` | Copy a module into an empty directory as a starting point |
| `terraform get` / `terraform get -update` | (Re)download modules only |
| `terraform providers` | Tree of providers required by the configuration and modules |
| `terraform providers lock -platform=linux_amd64 -platform=darwin_arm64 -platform=windows_amd64` | Add checksums for all team/CI platforms to the lock file |
| `terraform providers mirror ./mirror` | Download providers into a local filesystem mirror |
| `terraform providers schema -json > schema.json` | Dump every resource/data source schema (feeds tooling and `console`) |

Backend config file example:

```hcl
# env/prod.s3.tfbackend
bucket       = "capstone-tfstate-123456789012"
key          = "live/prod/network/terraform.tfstate"
region       = "eu-west-1"
encrypt      = true
kms_key_id   = "arn:aws:kms:eu-west-1:123456789012:key/<KEY_ID>"
use_lockfile = true
```

---

## 3. Write and Validate

| Command | Explanation |
|---|---|
| `terraform fmt` | Rewrite files in the current directory to canonical style |
| `terraform fmt -recursive` | Include subdirectories |
| `terraform fmt -check -diff` | Exit non-zero and show diffs if anything would change (CI) |
| `terraform validate` | Check syntax, types, and references; needs `init` (backend may be disabled) |
| `terraform validate -json` | Machine-readable diagnostics |
| `terraform console` | Interactive expression evaluator with access to state and config |
| `echo 'cidrsubnet("10.0.0.0/16", 4, 2)' \| terraform console` | One-shot evaluation |
| `terraform console -plan` | (1.9+) Evaluate against a fresh plan so unknowns are resolved where possible |
| `tflint --init && tflint --recursive` | Lint with provider rules |
| `tflint --format=sarif > tflint.sarif` | SARIF for GitHub code scanning |
| `terraform-docs markdown table --output-file README.md .` | Generate module documentation |

---

## 4. Plan

| Command | Explanation |
|---|---|
| `terraform plan` | Compute the diff and print it |
| `terraform plan -out=tfplan` | Save the plan to a binary file for an exact later apply |
| `terraform plan -var-file=env/prod.tfvars` | Load variables from a file |
| `terraform plan -var='environment=prod'` | Set one variable |
| `terraform plan -var='azs=["eu-west-1a","eu-west-1b"]'` | Set a list variable (HCL syntax inside quotes) |
| `terraform plan -target='module.vpc'` | Limit scope to a resource/module and its dependencies (emergency only) |
| `terraform plan -replace='aws_instance.bastion'` | Plan a forced replacement |
| `terraform plan -refresh-only` | Only detect drift; propose state updates, no infrastructure changes |
| `terraform plan -refresh=false` | Skip the refresh step (faster, hides drift) |
| `terraform plan -destroy` | Plan a complete teardown |
| `terraform plan -detailed-exitcode` | Exit 0 no changes, 1 error, 2 changes present |
| `terraform plan -lock=false` | Do not take a state lock (read-only investigations only) |
| `terraform plan -lock-timeout=5m` | Wait up to 5 minutes for a lock |
| `terraform plan -parallelism=5` | Limit concurrent graph walkers |
| `terraform plan -json` | Streaming JSON log (one object per line) |
| `terraform plan -compact-warnings -no-color -input=false` | CI output |
| `terraform plan -generate-config-out=generated.tf` | Write HCL for `import` blocks lacking configuration |
| `terraform show tfplan` | Human-readable rendering of a saved plan |
| `terraform show -json tfplan > plan.json` | JSON plan for Checkov/Conftest/OPA |
| `terraform show -json tfplan \| jq -r '.resource_changes[] \| select(.change.actions \| index("delete")) \| .address'` | List everything a plan would destroy |

Plan JSON structure you will query most:

```bash
jq '.resource_changes[] | {address, actions: .change.actions}' plan.json
jq '[.resource_changes[] | select(.change.actions == ["delete","create"] or .change.actions == ["create","delete"])] | length' plan.json   # replacements
jq '.planned_values.root_module.resources[] | select(.type=="aws_db_instance") | .values.multi_az' plan.json
```

---

## 5. Apply and Destroy

| Command | Explanation |
|---|---|
| `terraform apply` | Plan, prompt, apply |
| `terraform apply tfplan` | Apply a saved plan exactly; no prompt, no variables accepted |
| `terraform apply -auto-approve` | Skip the prompt (scripts, labs; in CI prefer applying a saved plan) |
| `terraform apply -target='aws_security_group.db'` | Apply a subset (follow with a full plan) |
| `terraform apply -replace='aws_db_instance.postgres'` | Replace one resource now |
| `terraform apply -refresh-only` | Accept detected drift into state without changing infrastructure |
| `terraform apply -lock-timeout=10m -input=false -no-color` | CI apply flags |
| `terraform apply -parallelism=20` | More concurrency for wide graphs |
| `terraform destroy` | Plan and prompt for a full teardown |
| `terraform destroy -target='module.compute'` | Destroy a subset |
| `terraform destroy -auto-approve` | Lab teardown |
| `terraform apply -destroy` | Same as `destroy` |

Reading an apply:

```text
aws_vpc.main: Creating...
aws_vpc.main: Creation complete after 2s [id=vpc-0abc123def456]
aws_subnet.private["a"]: Creating...
aws_subnet.private["b"]: Creating...
aws_subnet.private["c"]: Creating...
aws_subnet.private["b"]: Creation complete after 1s [id=subnet-0b...]
aws_db_instance.postgres: Still creating... [5m10s elapsed]
aws_db_instance.postgres: Creation complete after 7m42s [id=capstone-dev-postgres]

Apply complete! Resources: 14 added, 0 changed, 0 destroyed.
```

---

## 6. State

| Command | Explanation |
|---|---|
| `terraform state list` | Every resource address in state |
| `terraform state list 'module.vpc'` | Filter by address prefix |
| `terraform state list -id=vpc-0abc123` | Find the address for a real ID |
| `terraform state show 'aws_subnet.private["a"]'` | Attributes of one instance |
| `terraform state pull > state.json` | Download raw state (contains secrets) |
| `terraform state push state.json` | Upload state (dangerous; refuses lineage/serial mismatch without `-force`) |
| `terraform state mv 'aws_subnet.a' 'aws_subnet.private["a"]'` | Rename/move in state only (prefer `moved` blocks) |
| `terraform state mv -state-out=../other/terraform.tfstate 'module.rds' 'module.rds'` | Move a resource to another local state file (split a state) |
| `terraform state rm 'aws_instance.legacy'` | Forget a resource; it is not destroyed (prefer `removed` blocks) |
| `terraform state replace-provider hashicorp/aws registry.opentofu.org/hashicorp/aws` | Rewrite provider addresses (OpenTofu migration) |
| `terraform import 'aws_s3_bucket.logs' capstone-prod-logs` | Legacy immediate import (prefer `import` blocks) |
| `terraform force-unlock <LOCK_ID>` | Remove a stale lock after confirming no run is active |
| `terraform refresh` | Deprecated alias for `apply -refresh-only -auto-approve`; avoid |
| `terraform taint` / `untaint` | Deprecated; use `-replace` |

Declarative state operations (reviewable in PRs):

```hcl
moved   { from = aws_subnet.a            to = aws_subnet.private["a"] }
removed { from = aws_instance.legacy     lifecycle { destroy = false } }
import  { to = aws_s3_bucket.logs        id = "capstone-prod-logs" }
import  { for_each = var.bucket_names    to = aws_s3_bucket.many[each.key]  id = each.value }   # 1.7+
```

Splitting one state into two (moving the data layer out of a monolith):

```bash
# in the monolith directory
terraform state pull > monolith.tfstate
# in the new data-layer directory (backend configured, init done)
terraform state pull > data.tfstate            # empty new state
terraform state mv -state=../monolith/monolith.tfstate -state-out=data.tfstate 'module.rds' 'module.rds'
terraform state push data.tfstate
# back in the monolith
terraform state push monolith.tfstate
# verify both: plan shows "No changes"
```

---

## 7. Workspaces

| Command | Explanation |
|---|---|
| `terraform workspace list` | Workspaces in the current backend (`*` marks current) |
| `terraform workspace show` | Current workspace name |
| `terraform workspace new pr-123` | Create and switch |
| `terraform workspace select prod` | Switch |
| `terraform workspace select -or-create staging` | Switch, creating if missing (1.4+) |
| `terraform workspace delete pr-123` | Delete an empty workspace (`-force` if state is non-empty) |

In S3 the non-default workspace state lives at `env:/<workspace>/<key>` (configurable with `workspace_key_prefix`).

---

## 8. Outputs, Graph, Test

| Command | Explanation |
|---|---|
| `terraform output` | All root outputs (sensitive ones redacted) |
| `terraform output vpc_id` | One output, quoted |
| `terraform output -raw vpc_id` | Unquoted string for shell use |
| `terraform output -json` | All outputs including sensitive values as JSON |
| `terraform output -json \| jq -r '.private_subnet_ids.value[]'` | Extract a list |
| `terraform graph` | DOT graph of the configuration |
| `terraform graph -type=plan \| dot -Tsvg > graph.svg` | Render the plan graph |
| `terraform graph -draw-cycles` | Highlight dependency cycles |
| `terraform test` | Run `*.tftest.hcl` under `tests/` and the current directory |
| `terraform test -filter=tests/vpc.tftest.hcl` | Run one file |
| `terraform test -verbose` | Print plans/applies for each run block |
| `terraform test -junit-xml=results.xml` | (1.11+) JUnit report for CI |
| `terraform test -test-directory=integration` | Alternative test directory |

---

## 9. Login, Cloud, Metadata

| Command | Explanation |
|---|---|
| `terraform login` | Obtain an API token for HCP Terraform or a private registry |
| `terraform logout` | Remove the token |
| `terraform metadata functions -json` | List all built-in functions with signatures |
| `terraform modules` | (1.10+) List modules in the configuration with sources and versions |

---

## 10. Configuration Snippets by Task

### 10.1 Root module skeleton

```hcl
# versions.tf
terraform {
  required_version = ">= 1.9.0, < 2.0.0"
  required_providers {
    aws = { source = "hashicorp/aws", version = "~> 5.80" }
  }
  backend "s3" {}                       # settings via -backend-config
}

# providers.tf
provider "aws" {
  region = var.region
  default_tags { tags = local.tags }
}

# variables.tf / locals.tf / main.tf / outputs.tf
```

### 10.2 Conditional resource

```hcl
resource "aws_cloudwatch_log_group" "flow" {
  count             = var.enable_flow_logs ? 1 : 0
  name              = "/vpc/${var.name}/flow"
  retention_in_days = 90
  kms_key_id        = var.logs_kms_key_arn
}
```

### 10.3 `for_each` over a map of objects

```hcl
resource "aws_iam_role" "irsa" {
  for_each           = var.service_accounts   # { "api" = { namespace = "apps", policy_arns = [...] } }
  name               = "${var.cluster_name}-${each.key}"
  assume_role_policy = data.aws_iam_policy_document.irsa[each.key].json
}

resource "aws_iam_role_policy_attachment" "irsa" {
  for_each = { for pair in flatten([
    for sa, cfg in var.service_accounts : [
      for arn in cfg.policy_arns : { key = "${sa}:${arn}", sa = sa, arn = arn }
    ]
  ]) : pair.key => pair }

  role       = aws_iam_role.irsa[each.value.sa].name
  policy_arn = each.value.arn
}
```

### 10.4 Reading another layer's outputs

```hcl
# Option A: remote state (couples layers to the backend)
data "terraform_remote_state" "network" {
  backend = "s3"
  config = {
    bucket = "capstone-tfstate-123456789012"
    key    = "live/${var.environment}/network/terraform.tfstate"
    region = "eu-west-1"
  }
}
# data.terraform_remote_state.network.outputs.private_subnet_ids

# Option B: SSM parameter contract (decoupled; also readable by CloudFormation and Pulumi)
data "aws_ssm_parameter" "private_subnet_ids" {
  name = "/capstone/${var.environment}/network/private_subnet_ids"
}
locals {
  private_subnet_ids = split(",", data.aws_ssm_parameter.private_subnet_ids.value)
}
```

### 10.5 Writing the contract from the network layer

```hcl
resource "aws_ssm_parameter" "exports" {
  for_each = {
    vpc_id             = module.vpc.vpc_id
    private_subnet_ids = join(",", module.vpc.private_subnets)
    public_subnet_ids  = join(",", module.vpc.public_subnets)
  }
  name  = "/capstone/${var.environment}/network/${each.key}"
  type  = "String"
  value = each.value
}
```

### 10.6 Provider-level assume role and default tags

```hcl
provider "aws" {
  region = var.region
  assume_role {
    role_arn     = "arn:aws:iam::123456789012:role/TerraformExecution"
    session_name = "tf-${var.environment}-${var.layer}"
  }
  default_tags {
    tags = { ManagedBy = "terraform", Environment = var.environment, Layer = var.layer }
  }
}
```

### 10.7 Kubernetes/Helm providers authenticated to EKS without static kubeconfig

```hcl
data "aws_eks_cluster_auth" "this" {
  name = module.eks.cluster_name
}

provider "kubernetes" {
  host                   = module.eks.cluster_endpoint
  cluster_ca_certificate = base64decode(module.eks.cluster_certificate_authority_data)
  exec {
    api_version = "client.authentication.k8s.io/v1beta1"
    command     = "aws"
    args        = ["eks", "get-token", "--cluster-name", module.eks.cluster_name]
  }
}

provider "helm" {
  kubernetes {
    host                   = module.eks.cluster_endpoint
    cluster_ca_certificate = base64decode(module.eks.cluster_certificate_authority_data)
    exec {
      api_version = "client.authentication.k8s.io/v1beta1"
      command     = "aws"
      args        = ["eks", "get-token", "--cluster-name", module.eks.cluster_name]
    }
  }
}
```

### 10.8 Lifecycle guardrails for the state bucket

```hcl
resource "aws_s3_bucket" "state" {
  bucket = "capstone-tfstate-${data.aws_caller_identity.current.account_id}"
  lifecycle { prevent_destroy = true }
}

resource "aws_s3_bucket_versioning" "state" {
  bucket = aws_s3_bucket.state.id
  versioning_configuration { status = "Enabled" }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "state" {
  bucket = aws_s3_bucket.state.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = "aws:kms"
      kms_master_key_id = aws_kms_key.state.arn
    }
    bucket_key_enabled = true
  }
}

resource "aws_s3_bucket_policy" "state_tls_only" {
  bucket = aws_s3_bucket.state.id
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Sid       = "DenyInsecureTransport"
      Effect    = "Deny"
      Principal = "*"
      Action    = "s3:*"
      Resource  = [aws_s3_bucket.state.arn, "${aws_s3_bucket.state.arn}/*"]
      Condition = { Bool = { "aws:SecureTransport" = "false" } }
    }]
  })
}
```

### 10.9 Ephemeral secret into a provider (1.10+)

```hcl
ephemeral "aws_secretsmanager_secret_version" "grafana" {
  secret_id = "capstone/prod/grafana-admin"
}

provider "grafana" {
  url  = "https://grafana.example.com"
  auth = ephemeral.aws_secretsmanager_secret_version.grafana.secret_string
}
```

---

## 11. Toolchain Cheatsheet

| Tool | Command | Explanation |
|---|---|---|
| Checkov | `checkov -d . --framework terraform --quiet --compact` | Scan HCL |
| Checkov | `checkov -f plan.json --framework terraform_plan --repo-root-for-plan-enrichment .` | Scan a plan with module context |
| Checkov | `checkov -d . --skip-check CKV_AWS_144 --soft-fail-on LOW` | Suppress and soft-fail |
| Checkov inline | `# checkov:skip=CKV_AWS_18: Access logging handled by org-level bucket` | Justified skip above a resource |
| Trivy | `trivy config --severity HIGH,CRITICAL --exit-code 1 .` | Misconfiguration scan |
| Trivy | `trivy fs --scanners secret .` | Secret scan of the repo |
| Conftest | `conftest test plan.json --policy policy/ --all-namespaces` | OPA policies on plan JSON |
| OPA | `opa eval --data policy/ --input plan.json 'data.terraform.deny'` | Evaluate directly |
| Infracost | `infracost breakdown --path . --format table` | Cost estimate per plan |
| Terragrunt | `terragrunt run-all plan` | Plan every unit under the current directory in dependency order |
| Terragrunt | `terragrunt run-all apply --terragrunt-non-interactive` | Apply all units |
| Terragrunt | `terragrunt hclfmt` | Format `terragrunt.hcl` |
| terraform-docs | `terraform-docs markdown table . > README.md` | Module docs |
| tfenv / tenv | `tenv tf install 1.10.5 && tenv tf use 1.10.5` | Manage CLI versions |
| OpenTofu | `tofu init && tofu plan` | Drop-in substitution |
| OpenTofu | `tofu init -upgrade` after `state replace-provider` | Migration step |

Terragrunt `terragrunt.hcl` that removes backend duplication:

```hcl
# live/terragrunt.hcl (root)
remote_state {
  backend = "s3"
  generate = { path = "backend.tf", if_exists = "overwrite_terragrunt" }
  config = {
    bucket       = "capstone-tfstate-${get_aws_account_id()}"
    key          = "${path_relative_to_include()}/terraform.tfstate"
    region       = "eu-west-1"
    encrypt      = true
    use_lockfile = true
  }
}

# live/prod/data/terragrunt.hcl
include "root" { path = find_in_parent_folders() }
terraform { source = "git::https://github.com/<ORG>/terraform-modules.git//rds-postgres?ref=v2.3.1" }
dependency "network" { config_path = "../network" }
inputs = {
  subnet_ids = dependency.network.outputs.private_subnet_ids
  vpc_id     = dependency.network.outputs.vpc_id
}
```

---

## 12. Exit Codes and Automation Idioms

| Command | Exit codes |
|---|---|
| `terraform plan` | 0 success (changes or not), 1 error |
| `terraform plan -detailed-exitcode` | 0 no changes, 1 error, 2 changes |
| `terraform apply` | 0 success, 1 error (state saved for completed resources) |
| `terraform validate` / `fmt -check` | 0 ok, 1 problems (fmt: 3 when files differ) |
| `terraform test` | 0 all pass, 1 failures |

```bash
# Drift check job
set +e
terraform plan -refresh-only -detailed-exitcode -no-color -input=false -lock=false
rc=$?
set -e
case $rc in
  0) echo "No drift" ;;
  2) echo "DRIFT DETECTED"; exit 2 ;;
  *) echo "Plan error"; exit 1 ;;
esac
```

```bash
# Plan and fail if anything would be destroyed (guard for prod pipelines)
terraform plan -out=tfplan -input=false -lock-timeout=5m
terraform show -json tfplan > plan.json
destroys=$(jq '[.resource_changes[] | select(.change.actions | index("delete"))] | length' plan.json)
if [ "$destroys" -gt 0 ] && [ "${ALLOW_DESTROY:-false}" != "true" ]; then
  echo "Plan would destroy $destroys resource(s); set ALLOW_DESTROY=true to proceed"; exit 1
fi
```

Next: [04-hands-on-labs.md](04-hands-on-labs.md).
