# Terraform: Installation and Setup

| | |
|---|---|
| **Cluster** | 05 Infrastructure as Code |
| **Role** | Anchor |
| **Prerequisites** | An AWS account you are allowed to create resources in, AWS CLI v2, Git |
| **Target version** | Terraform 1.9+ (1.10+ recommended for S3 native locking and ephemeral values) |
| **Capstone phase** | Phase 2 (Months 5-8) |

> After this file you will have a pinned Terraform version managed by a version manager, a correctly authenticated AWS provider that uses short-lived credentials rather than static access keys, the supporting tools (tflint, Checkov, Trivy, tfsec successor, pre-commit), and a verified first `init`/`plan`/`apply`/`destroy` cycle against a real account.

---

## 1. Choosing a Version Strategy

Terraform follows semantic versioning and the CLI is a single static binary. Three rules:

1. **Pin the CLI per repository** with a `.terraform-version` file (read by `tfenv` and `tenv`) and a `required_version` constraint in configuration. The two must agree.
2. **Everyone on a state file uses the same minor version.** State written by 1.10 can be read by 1.9 in most cases, but a newer CLI may upgrade the state format and lock out older clients. CI and laptops must match.
3. **Upgrade deliberately.** Read the upgrade guide for each minor version; run `terraform plan` after upgrading and expect "no changes".

### 1.1 Terraform versus OpenTofu

| | Terraform | OpenTofu |
|---|---|---|
| Binary | `terraform` | `tofu` |
| Licence | BSL 1.1 (source-available; cannot be used to build a competing product) | MPL-2.0 (Linux Foundation) |
| Config compatibility | Reference | Compatible with Terraform 1.5-era language; diverges with its own features from 1.7+ (state encryption, `for_each` on providers, early variable evaluation in backends and module sources) |
| Registry | registry.terraform.io | registry.opentofu.org (mirrors public providers and modules) |
| Version file | `.terraform-version` | `.opentofu-version` |

This folder uses the `terraform` binary. Every command works with `tofu` substituted unless noted. See [05-production-patterns.md](05-production-patterns.md#12-opentofu-compatibility) for a migration checklist.

---

## 2. Installing Terraform

### 2.1 Version manager (recommended)

`tenv` manages Terraform, OpenTofu, Terragrunt, and Atmos from one tool and respects `.terraform-version`.

```bash
# macOS
brew install tenv

# Linux (generic)
TENV_VERSION="$(curl -s https://api.github.com/repos/tofuutils/tenv/releases/latest | grep -oP '"tag_name": "\K[^"]+')"
curl -LO "https://github.com/tofuutils/tenv/releases/download/${TENV_VERSION}/tenv_${TENV_VERSION}_amd64.deb"
sudo dpkg -i "tenv_${TENV_VERSION}_amd64.deb"

# Install and select a Terraform version
tenv tf install 1.10.5
tenv tf use 1.10.5
terraform version
```

Expected output:

```text
Terraform v1.10.5
on linux_amd64
```

Alternatively `tfenv` (Terraform only):

```bash
git clone --depth=1 https://github.com/tfutils/tfenv.git ~/.tfenv
echo 'export PATH="$HOME/.tfenv/bin:$PATH"' >> ~/.bashrc && source ~/.bashrc
tfenv install 1.10.5 && tfenv use 1.10.5
```

### 2.2 Package manager

```bash
# Debian / Ubuntu (HashiCorp apt repository)
sudo apt-get update && sudo apt-get install -y gnupg software-properties-common curl
curl -fsSL https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" \
  | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt-get update && sudo apt-get install -y terraform

# RHEL / Fedora / Amazon Linux 2023
sudo dnf install -y dnf-plugins-core
sudo dnf config-manager --add-repo https://rpm.releases.hashicorp.com/fedora/hashicorp.repo
sudo dnf install -y terraform

# macOS
brew tap hashicorp/tap && brew install hashicorp/tap/terraform
```

### 2.3 Windows

```powershell
# winget
winget install HashiCorp.Terraform

# or Chocolatey
choco install terraform --version=1.10.5

# or scoop
scoop install terraform
```

On Windows prefer running Terraform inside WSL2 for consistent path handling and because most module READMEs assume a POSIX shell. If you must use native Windows, use PowerShell 7 and set `TF_PLUGIN_CACHE_DIR` to a short path such as `C:\tfcache` to avoid `MAX_PATH` issues.

### 2.4 Manual binary with checksum verification

```bash
TF_VERSION="1.10.5"
cd "$(mktemp -d)"
curl -sLO "https://releases.hashicorp.com/terraform/${TF_VERSION}/terraform_${TF_VERSION}_linux_amd64.zip"
curl -sLO "https://releases.hashicorp.com/terraform/${TF_VERSION}/terraform_${TF_VERSION}_SHA256SUMS"
curl -sLO "https://releases.hashicorp.com/terraform/${TF_VERSION}/terraform_${TF_VERSION}_SHA256SUMS.sig"
# Verify the signature with the HashiCorp public key, then the checksum
gpg --verify "terraform_${TF_VERSION}_SHA256SUMS.sig" "terraform_${TF_VERSION}_SHA256SUMS"
sha256sum --check --ignore-missing "terraform_${TF_VERSION}_SHA256SUMS"
unzip "terraform_${TF_VERSION}_linux_amd64.zip"
sudo install -m 0755 terraform /usr/local/bin/terraform
```

### 2.5 Container image (for CI parity)

```bash
docker run --rm -it \
  -v "$PWD":/workspace -w /workspace \
  -v "$HOME/.aws":/root/.aws:ro \
  -e AWS_PROFILE \
  hashicorp/terraform:1.10.5 version
```

In CI prefer `hashicorp/setup-terraform` (GitHub Actions) or a pinned image digest rather than `:latest`.

---

## 3. Installing the Supporting Toolchain

| Tool | Purpose | Install |
|---|---|---|
| AWS CLI v2 | Provider auth, inspecting resources, SSO login | `curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o awscliv2.zip && unzip awscliv2.zip && sudo ./aws/install` |
| tflint + AWS ruleset | Linter: deprecated syntax, invalid instance types, unused declarations | `curl -s https://raw.githubusercontent.com/terraform-linters/tflint/master/install_linux.sh \| bash` |
| Checkov | Policy scanner for HCL and plan JSON | `pipx install checkov` |
| Trivy | Misconfiguration and secret scanner (absorbed tfsec) | `brew install trivy` or apt repo |
| terraform-docs | Generates module README input/output tables | `brew install terraform-docs` |
| pre-commit | Runs fmt/validate/lint on commit | `pipx install pre-commit` |
| Terragrunt | Thin wrapper for DRY backends and dependencies (optional) | `tenv tg install latest` |
| jq | Reading `terraform show -json` | package manager |
| Graphviz (`dot`) | Rendering `terraform graph` | package manager |

Create a `.tflint.hcl` in the repository root:

```hcl
plugin "aws" {
  enabled = true
  version = "0.37.0"
  source  = "github.com/terraform-linters/tflint-ruleset-aws"
}

rule "terraform_required_version" { enabled = true }
rule "terraform_required_providers" { enabled = true }
rule "terraform_naming_convention" { enabled = true }
rule "terraform_unused_declarations" { enabled = true }
rule "terraform_documented_variables" { enabled = true }
```

```bash
tflint --init && tflint --version
```

A `.pre-commit-config.yaml` that enforces hygiene before code reaches CI:

```yaml
repos:
  - repo: https://github.com/antonbabenko/pre-commit-terraform
    rev: v1.96.2
    hooks:
      - id: terraform_fmt
      - id: terraform_validate
      - id: terraform_tflint
        args: ["--args=--config=__GIT_WORKING_DIR__/.tflint.hcl"]
      - id: terraform_docs
      - id: terraform_checkov
        args: ["--args=--quiet", "--args=--framework terraform"]
```

```bash
pre-commit install && pre-commit run --all-files
```

---

## 4. Authenticating the AWS Provider

The provider follows the standard AWS SDK credential chain. Order of precedence:

1. Explicit `access_key`/`secret_key` in the provider block (never do this).
2. Environment variables `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_SESSION_TOKEN`.
3. Web identity token (`AWS_WEB_IDENTITY_TOKEN_FILE` + `AWS_ROLE_ARN`), which is what OIDC in CI uses.
4. Shared config and credentials files (`~/.aws/config`), including SSO profiles.
5. Container and EC2 instance metadata credentials.

### 4.1 Workstation: IAM Identity Center (SSO) profile

```ini
# ~/.aws/config
[sso-session corp]
sso_start_url = https://<ORG_ALIAS>.awsapps.com/start
sso_region = <REGION>
sso_registration_scopes = sso:account:access

[profile capstone-dev]
sso_session = corp
sso_account_id = 123456789012
sso_role_name = PowerUserAccess
region = <REGION>
output = json
```

```bash
aws sso login --profile capstone-dev
export AWS_PROFILE=capstone-dev
aws sts get-caller-identity
```

Expected output:

```json
{
    "UserId": "AROAEXAMPLEID:you",
    "Account": "123456789012",
    "Arn": "arn:aws:sts::123456789012:assumed-role/AWSReservedSSO_PowerUserAccess_abc123/you"
}
```

### 4.2 Provider block that assumes a role

Keep the provider block free of credentials and let the chain do its work. Assume a dedicated role so the permissions Terraform runs with are explicit and auditable:

```hcl
provider "aws" {
  region = var.region

  assume_role {
    role_arn     = "arn:aws:iam::123456789012:role/TerraformExecution"
    session_name = "terraform-${var.environment}"
  }

  default_tags {
    tags = {
      ManagedBy   = "terraform"
      Environment = var.environment
      Repository  = "github.com/<ORG>/capstone-infra"
    }
  }
}
```

### 4.3 CI: OIDC, never static keys

In GitHub Actions the `aws-actions/configure-aws-credentials` action exchanges the job's OIDC token for temporary credentials; the provider picks them up from environment variables. The full pattern including the IAM trust policy is in [05-production-patterns.md](05-production-patterns.md#5-cicd-pipeline-plan-on-pr-apply-on-merge).

### 4.4 Provider authentication for other clouds

| Provider | Local | CI |
|---|---|---|
| `azurerm` | `az login`, `ARM_SUBSCRIPTION_ID` | Workload identity federation: `ARM_USE_OIDC=true`, `ARM_CLIENT_ID`, `ARM_TENANT_ID` |
| `google` | `gcloud auth application-default login` | Workload Identity Federation via `google-github-actions/auth` |
| `kubernetes` / `helm` | Kubeconfig context or `exec` plugin for EKS tokens | Same, using the OIDC-obtained AWS credentials |

---

## 5. First-Run Configuration

### 5.1 CLI configuration file

`~/.terraformrc` (Linux/macOS) or `%APPDATA%\terraform.rc` (Windows):

```hcl
plugin_cache_dir   = "$HOME/.terraform.d/plugin-cache"
disable_checkpoint = true
```

```bash
mkdir -p "$HOME/.terraform.d/plugin-cache"
```

The plugin cache stops every working directory from downloading its own copy of the 600 MB AWS provider. `disable_checkpoint` stops the CLI phoning home for version checks (required in many corporate networks).

### 5.2 Useful environment variables

| Variable | Effect |
|---|---|
| `TF_LOG=DEBUG` / `TRACE` | Verbose logging (TRACE includes provider gRPC payloads) |
| `TF_LOG_PATH=./tf.log` | Write logs to a file |
| `TF_INPUT=0` | Never prompt (CI) |
| `TF_IN_AUTOMATION=1` | Suppresses "next steps" hints |
| `TF_VAR_<name>` | Sets a root variable, e.g. `TF_VAR_environment=dev` |
| `TF_CLI_ARGS_plan="-lock-timeout=5m"` | Default arguments for a subcommand |
| `TF_PLUGIN_CACHE_DIR` | Same as `plugin_cache_dir` |
| `TF_DATA_DIR` | Relocates `.terraform/` (useful for running multiple backends from one directory) |
| `TF_WORKSPACE` | Selects a workspace without `terraform workspace select` |

### 5.3 Git ignore

```gitignore
# .gitignore
.terraform/
*.tfstate
*.tfstate.*
*.tfplan
crash.log
override.tf
override.tf.json
*_override.tf
.terraform.tfstate.lock.info
# Do NOT ignore .terraform.lock.hcl - commit it
```

Variable files containing secrets (`*.auto.tfvars` with credentials) must never be committed; prefer environment variables or a secrets manager, see [05-production-patterns.md](05-production-patterns.md#4-secrets-handling).

---

## 6. Verification: A Complete First Cycle

Create a scratch directory and a configuration that touches a real API but costs nothing: an S3 bucket with security defaults.

```bash
mkdir -p ~/tf-hello && cd ~/tf-hello
```

```hcl
# versions.tf
terraform {
  required_version = ">= 1.9.0, < 2.0.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.80"
    }
    random = {
      source  = "hashicorp/random"
      version = "~> 3.6"
    }
  }
}

provider "aws" {
  region = var.region
}
```

```hcl
# main.tf
variable "region" {
  type    = string
  default = "eu-west-1"
}

resource "random_pet" "suffix" {
  length = 2
}

resource "aws_s3_bucket" "hello" {
  bucket        = "tf-hello-${random_pet.suffix.id}"
  force_destroy = true
}

resource "aws_s3_bucket_versioning" "hello" {
  bucket = aws_s3_bucket.hello.id
  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "hello" {
  bucket = aws_s3_bucket.hello.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}

resource "aws_s3_bucket_public_access_block" "hello" {
  bucket                  = aws_s3_bucket.hello.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

output "bucket_name" {
  value = aws_s3_bucket.hello.bucket
}
```

Run the cycle:

```bash
terraform init
```

Expected output (abridged):

```text
Initializing the backend...
Initializing provider plugins...
- Finding hashicorp/aws versions matching "~> 5.80"...
- Finding hashicorp/random versions matching "~> 3.6"...
- Installing hashicorp/aws v5.84.0...
- Installed hashicorp/aws v5.84.0 (signed by HashiCorp)
- Installing hashicorp/random v3.6.3...
- Installed hashicorp/random v3.6.3 (signed by HashiCorp)
Terraform has created a lock file .terraform.lock.hcl to record the provider
selections it made above. Include this file in your version control repository
so that Terraform can guarantee to make the same selections by default when
you run "terraform init" in the future.

Terraform has been successfully initialized!
```

```bash
terraform fmt -check -recursive && terraform validate
terraform plan -out=tfplan
```

Expected output (abridged):

```text
Terraform used the selected providers to generate the following execution
plan. Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # aws_s3_bucket.hello will be created
  + resource "aws_s3_bucket" "hello" {
      + bucket        = (known after apply)
      + force_destroy = true
      ...
    }

  # random_pet.suffix will be created
  + resource "random_pet" "suffix" {
      + id        = (known after apply)
      + length    = 2
      + separator = "-"
    }

Plan: 5 to add, 0 to change, 0 to destroy.

Changes to Outputs:
  + bucket_name = (known after apply)

Saved the plan to: tfplan
```

```bash
terraform apply tfplan
terraform output bucket_name
aws s3api get-bucket-versioning --bucket "$(terraform output -raw bucket_name)"
```

Expected output:

```text
Apply complete! Resources: 5 added, 0 changed, 0 destroyed.

Outputs:

bucket_name = "tf-hello-sincere-mole"
```

```json
{
    "Status": "Enabled"
}
```

Prove idempotency, then tear down:

```bash
terraform plan        # expect: No changes. Your infrastructure matches the configuration.
terraform destroy -auto-approve
```

Expected output:

```text
Destroy complete! Resources: 5 destroyed.
```

Inspect what was created locally:

```bash
ls -la
cat .terraform.lock.hcl
```

You will see `terraform.tfstate` (now containing only an empty resources list), `terraform.tfstate.backup`, `.terraform/` with provider binaries, and the lock file recording provider versions and `h1:`/`zh:` hashes.

---

## 7. Editor Setup

- **VS Code**: HashiCorp Terraform extension (language server, format on save, hover docs). Settings: `"[terraform]": { "editor.formatOnSave": true, "editor.defaultFormatter": "hashicorp.terraform" }`.
- **JetBrains**: the bundled Terraform and HCL plugin.
- **Neovim**: `terraform-ls` via `mason.nvim` plus `nvim-treesitter` HCL grammar.

Enable `terraform-ls` experimental features `prefillRequiredFields` and `validateOnSave` for fast feedback.

---

## 8. Setup Checklist

- [ ] Terraform pinned via `.terraform-version` and `required_version`; CI uses the same version
- [ ] Plugin cache directory configured
- [ ] AWS authentication uses SSO or assumed roles locally and OIDC in CI; no access keys in files or provider blocks
- [ ] `.gitignore` excludes state and plan files; `.terraform.lock.hcl` is committed
- [ ] tflint, Checkov, Trivy, terraform-docs, and pre-commit installed and passing on the hello configuration
- [ ] First `init`/`plan`/`apply`/`plan` (no changes)/`destroy` cycle completed

Next: [02-core-concepts.md](02-core-concepts.md).
