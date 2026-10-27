# Terraform: Hands-On Labs

| | |
|---|---|
| **Cluster** | 05 Infrastructure as Code |
| **Role** | Anchor |
| **Prerequisites** | [01-installation-and-setup.md](01-installation-and-setup.md), [02-core-concepts.md](02-core-concepts.md); an AWS account with permission to create VPC, RDS, EKS, IAM, S3, KMS, SSM resources |
| **Target version** | Terraform 1.10+, AWS provider 5.80+ |
| **Capstone phase** | Phase 2 (Months 5-8) |

> Eight progressive labs that build the capstone's Phase 2 infrastructure: a locked, encrypted state backend; a hand-written VPC module; a Multi-AZ RDS PostgreSQL layer; an EKS cluster; importing an existing resource; a `terraform test` suite; a GitHub Actions OIDC pipeline; and a controlled teardown. Every lab ends with verification and cleanup. Labs 3 and 4 create billable resources (roughly USD 0.10/hour for the NAT gateway and RDS, about USD 0.10/hour for the EKS control plane plus nodes); run the teardown the same day.

**Cost and safety notes.** Use a sandbox account. The labs use `eu-west-1`; change `region` consistently if you prefer another. Every lab uses `environment = "dev"` sizing (single NAT gateway, `db.t4g.micro`, two small nodes). Nothing in these labs stores a long-lived credential.

**Repository layout the labs build:**

```text
capstone-infra/
├── .github/workflows/terraform.yml          # Lab 7
├── .terraform-version                       # 1.10.5
├── global/
│   ├── state-backend/                       # Lab 1
│   └── iam/                                 # Lab 7
├── modules/
│   ├── vpc/                                 # Lab 2 (+ tests/ in Lab 6)
│   └── rds-postgres/                        # Lab 3
└── live/dev/
    ├── network/                             # Lab 2
    ├── data/                                # Lab 3
    └── compute/                             # Lab 4
```

```bash
mkdir -p capstone-infra/{global/{state-backend,iam},modules/{vpc,rds-postgres},live/dev/{network,data,compute}}
cd capstone-infra && git init && echo "1.10.5" > .terraform-version
```

Create the `.gitignore` from [01-installation-and-setup.md](01-installation-and-setup.md#53-git-ignore).

---

## Lab 1: S3 State Backend with Native Locking

**Objective.** Create the S3 bucket and KMS key that will hold every other layer's state, with versioning, encryption, TLS-only access, and S3 native lock files; then migrate this layer's own state into it.

**Prerequisites.** AWS credentials via SSO profile (`aws sts get-caller-identity` works).

**Time.** 30 minutes.

### Steps

1. Write the configuration in `global/state-backend/`.

   ```hcl
   # global/state-backend/versions.tf
   terraform {
     required_version = ">= 1.10.0, < 2.0.0"
     required_providers {
       aws = { source = "hashicorp/aws", version = "~> 5.80" }
     }
   }

   provider "aws" {
     region = var.region
     default_tags {
       tags = { ManagedBy = "terraform", Layer = "global/state-backend" }
     }
   }
   ```

   ```hcl
   # global/state-backend/main.tf
   variable "region" {
     type    = string
     default = "eu-west-1"
   }

   data "aws_caller_identity" "current" {}

   locals {
     bucket_name = "capstone-tfstate-${data.aws_caller_identity.current.account_id}"
   }

   resource "aws_kms_key" "state" {
     description             = "Terraform state encryption"
     deletion_window_in_days = 30
     enable_key_rotation     = true
   }

   resource "aws_kms_alias" "state" {
     name          = "alias/terraform-state"
     target_key_id = aws_kms_key.state.key_id
   }

   resource "aws_s3_bucket" "state" {
     bucket = local.bucket_name

     lifecycle {
       prevent_destroy = true
     }
   }

   resource "aws_s3_bucket_versioning" "state" {
     bucket = aws_s3_bucket.state.id
     versioning_configuration {
       status = "Enabled"
     }
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

   resource "aws_s3_bucket_public_access_block" "state" {
     bucket                  = aws_s3_bucket.state.id
     block_public_acls       = true
     block_public_policy     = true
     ignore_public_acls      = true
     restrict_public_buckets = true
   }

   resource "aws_s3_bucket_lifecycle_configuration" "state" {
     bucket = aws_s3_bucket.state.id
     rule {
       id     = "expire-old-versions"
       status = "Enabled"
       filter {}
       noncurrent_version_expiration {
         noncurrent_days = 90
       }
       abort_incomplete_multipart_upload {
         days_after_initiation = 7
       }
     }
   }

   resource "aws_s3_bucket_policy" "state" {
     bucket = aws_s3_bucket.state.id
     policy = jsonencode({
       Version = "2012-10-17"
       Statement = [
         {
           Sid       = "DenyInsecureTransport"
           Effect    = "Deny"
           Principal = "*"
           Action    = "s3:*"
           Resource  = [aws_s3_bucket.state.arn, "${aws_s3_bucket.state.arn}/*"]
           Condition = { Bool = { "aws:SecureTransport" = "false" } }
         },
         {
           Sid       = "DenyUnencryptedObjectUploads"
           Effect    = "Deny"
           Principal = "*"
           Action    = "s3:PutObject"
           Resource  = "${aws_s3_bucket.state.arn}/*"
           Condition = { StringNotEquals = { "s3:x-amz-server-side-encryption" = "aws:kms" } }
         }
       ]
     })
   }

   output "state_bucket" {
     value = aws_s3_bucket.state.bucket
   }

   output "state_kms_key_arn" {
     value = aws_kms_key.state.arn
   }
   ```

2. Apply with local state (the bucket does not exist yet, so it cannot hold its own state).

   ```bash
   cd global/state-backend
   terraform init && terraform apply -auto-approve
   ```

   Expected output:

   ```text
   Apply complete! Resources: 8 added, 0 changed, 0 destroyed.

   Outputs:

   state_bucket = "capstone-tfstate-123456789012"
   state_kms_key_arn = "arn:aws:kms:eu-west-1:123456789012:key/1a2b3c4d-..."
   ```

3. Add a backend block pointing at the new bucket and migrate.

   ```hcl
   # global/state-backend/backend.tf
   terraform {
     backend "s3" {
       bucket       = "capstone-tfstate-123456789012"     # replace with your output
       key          = "global/state-backend/terraform.tfstate"
       region       = "eu-west-1"
       encrypt      = true
       kms_key_id   = "alias/terraform-state"
       use_lockfile = true
     }
   }
   ```

   ```bash
   terraform init -migrate-state
   ```

   Expected output (answer `yes`):

   ```text
   Initializing the backend...
   Do you want to copy existing state to the new backend?
     Pre-existing state was found while migrating the previous "local" backend to the
     newly configured "s3" backend. ...
     Enter a value: yes

   Successfully configured the backend "s3"! Terraform will automatically
   use this backend unless the backend configuration changes.
   ```

4. Remove the now-redundant local state and confirm.

   ```bash
   rm -f terraform.tfstate terraform.tfstate.backup
   terraform plan
   ```

   Expected: `No changes. Your infrastructure matches the configuration.`

### Verify

Observe the lock file appear during a plan. In a second terminal:

```bash
aws s3 ls s3://capstone-tfstate-123456789012/global/state-backend/
```

While a `terraform plan` is running you will see `terraform.tfstate.tflock`; afterwards only `terraform.tfstate`. Simulate contention:

```bash
# terminal 1
terraform plan -lock-timeout=0s &
# terminal 2 immediately
terraform plan -lock-timeout=0s
```

Expected output in terminal 2:

```text
Error: Error acquiring the state lock

Error message: operation error S3: PutObject, https response error StatusCode: 412,
PreconditionFailed: At least one of the pre-conditions you specified did not hold
Lock Info:
  ID:        3b7e1c4a-...
  Path:      capstone-tfstate-123456789012/global/state-backend/terraform.tfstate
  Operation: OperationTypePlan
  Who:       you@workstation
  Created:   2026-10-27 09:14:02.1 +0000 UTC
```

### Cleanup

None. This layer stays for the rest of the labs (the Lab 8 teardown covers it).

### What you learned

State must exist somewhere before it can be remote; bootstrapping is a two-step dance. S3 native locking needs no DynamoDB table. `prevent_destroy`, versioning, KMS, and a TLS-only policy are the minimum protections for a file that will contain secrets.

---

## Lab 2: A Hand-Written VPC Module and the Network Layer

**Objective.** Write a reusable multi-AZ VPC module using `for_each`, `cidrsubnet`, and `dynamic` blocks, then instantiate it from `live/dev/network/` with remote state and publish its IDs to SSM Parameter Store as the cross-layer contract.

**Prerequisites.** Lab 1.

**Time.** 60 minutes.

### Steps

1. Create the module.

   ```hcl
   # modules/vpc/versions.tf
   terraform {
     required_version = ">= 1.9.0"
     required_providers {
       aws = { source = "hashicorp/aws", version = ">= 5.40" }
     }
   }
   ```

   ```hcl
   # modules/vpc/variables.tf
   variable "name" {
     description = "Name prefix for all resources."
     type        = string
   }

   variable "cidr" {
     description = "VPC CIDR block (/16 to /20)."
     type        = string
     validation {
       condition     = can(cidrnetmask(var.cidr)) && tonumber(split("/", var.cidr)[1]) <= 20 && tonumber(split("/", var.cidr)[1]) >= 16
       error_message = "cidr must be a valid IPv4 block between /16 and /20."
     }
   }

   variable "azs" {
     description = "Availability zones; one public and one private subnet are created per AZ."
     type        = list(string)
     validation {
       condition     = length(var.azs) >= 2 && length(var.azs) <= 4
       error_message = "Provide between 2 and 4 availability zones."
     }
   }

   variable "single_nat_gateway" {
     description = "Use one NAT gateway for all AZs (cheaper, non-HA)."
     type        = bool
     default     = true
   }

   variable "enable_flow_logs" {
     type    = bool
     default = true
   }

   variable "interface_endpoints" {
     description = "Interface VPC endpoints to create (service short names)."
     type        = set(string)
     default     = ["ecr.api", "ecr.dkr", "sts", "secretsmanager", "logs"]
   }

   variable "tags" {
     type    = map(string)
     default = {}
   }
   ```

   ```hcl
   # modules/vpc/main.tf
   locals {
     # Key subnets by AZ suffix letter so for_each keys are stable identities
     az_map = { for i, az in var.azs : substr(az, -1, 1) => { index = i, az = az } }

     private_subnets = { for k, v in local.az_map : k => merge(v, { cidr = cidrsubnet(var.cidr, 4, v.index) }) }        # /20s
     public_subnets  = { for k, v in local.az_map : k => merge(v, { cidr = cidrsubnet(var.cidr, 8, v.index + 48) }) }   # /24s

     nat_keys = var.single_nat_gateway ? [keys(local.az_map)[0]] : keys(local.az_map)
   }

   resource "aws_vpc" "this" {
     cidr_block           = var.cidr
     enable_dns_support   = true
     enable_dns_hostnames = true
     tags                 = merge(var.tags, { Name = var.name })
   }

   resource "aws_internet_gateway" "this" {
     vpc_id = aws_vpc.this.id
     tags   = merge(var.tags, { Name = var.name })
   }

   resource "aws_subnet" "public" {
     for_each                = local.public_subnets
     vpc_id                  = aws_vpc.this.id
     cidr_block              = each.value.cidr
     availability_zone       = each.value.az
     map_public_ip_on_launch = false
     tags = merge(var.tags, {
       Name                     = "${var.name}-public-${each.key}"
       "kubernetes.io/role/elb" = "1"
     })
   }

   resource "aws_subnet" "private" {
     for_each          = local.private_subnets
     vpc_id            = aws_vpc.this.id
     cidr_block        = each.value.cidr
     availability_zone = each.value.az
     tags = merge(var.tags, {
       Name                              = "${var.name}-private-${each.key}"
       "kubernetes.io/role/internal-elb" = "1"
     })
   }

   resource "aws_eip" "nat" {
     for_each = toset(local.nat_keys)
     domain   = "vpc"
     tags     = merge(var.tags, { Name = "${var.name}-nat-${each.key}" })
   }

   resource "aws_nat_gateway" "this" {
     for_each      = toset(local.nat_keys)
     allocation_id = aws_eip.nat[each.key].id
     subnet_id     = aws_subnet.public[each.key].id
     tags          = merge(var.tags, { Name = "${var.name}-nat-${each.key}" })
     depends_on    = [aws_internet_gateway.this]
   }

   resource "aws_route_table" "public" {
     vpc_id = aws_vpc.this.id
     tags   = merge(var.tags, { Name = "${var.name}-public" })
   }

   resource "aws_route" "public_default" {
     route_table_id         = aws_route_table.public.id
     destination_cidr_block = "0.0.0.0/0"
     gateway_id             = aws_internet_gateway.this.id
   }

   resource "aws_route_table_association" "public" {
     for_each       = aws_subnet.public
     subnet_id      = each.value.id
     route_table_id = aws_route_table.public.id
   }

   resource "aws_route_table" "private" {
     for_each = local.private_subnets
     vpc_id   = aws_vpc.this.id
     tags     = merge(var.tags, { Name = "${var.name}-private-${each.key}" })
   }

   resource "aws_route" "private_default" {
     for_each               = local.private_subnets
     route_table_id         = aws_route_table.private[each.key].id
     destination_cidr_block = "0.0.0.0/0"
     nat_gateway_id         = aws_nat_gateway.this[var.single_nat_gateway ? local.nat_keys[0] : each.key].id
   }

   resource "aws_route_table_association" "private" {
     for_each       = aws_subnet.private
     subnet_id      = each.value.id
     route_table_id = aws_route_table.private[each.key].id
   }

   # Gateway endpoint for S3 (free) attached to every private route table
   resource "aws_vpc_endpoint" "s3" {
     vpc_id            = aws_vpc.this.id
     service_name      = "com.amazonaws.${data.aws_region.current.name}.s3"
     vpc_endpoint_type = "Gateway"
     route_table_ids   = [for rt in aws_route_table.private : rt.id]
     tags              = merge(var.tags, { Name = "${var.name}-s3" })
   }

   data "aws_region" "current" {}

   resource "aws_security_group" "endpoints" {
     name        = "${var.name}-vpc-endpoints"
     description = "HTTPS from inside the VPC to interface endpoints"
     vpc_id      = aws_vpc.this.id
     tags        = merge(var.tags, { Name = "${var.name}-vpc-endpoints" })
   }

   resource "aws_vpc_security_group_ingress_rule" "endpoints_https" {
     security_group_id = aws_security_group.endpoints.id
     cidr_ipv4         = var.cidr
     from_port         = 443
     to_port           = 443
     ip_protocol       = "tcp"
   }

   resource "aws_vpc_endpoint" "interface" {
     for_each            = var.interface_endpoints
     vpc_id              = aws_vpc.this.id
     service_name        = "com.amazonaws.${data.aws_region.current.name}.${each.key}"
     vpc_endpoint_type   = "Interface"
     subnet_ids          = [for s in aws_subnet.private : s.id]
     security_group_ids  = [aws_security_group.endpoints.id]
     private_dns_enabled = true
     tags                = merge(var.tags, { Name = "${var.name}-${each.key}" })
   }

   resource "aws_cloudwatch_log_group" "flow" {
     count             = var.enable_flow_logs ? 1 : 0
     name              = "/vpc/${var.name}/flow-logs"
     retention_in_days = 30
     tags              = var.tags
   }

   data "aws_iam_policy_document" "flow_assume" {
     count = var.enable_flow_logs ? 1 : 0
     statement {
       actions = ["sts:AssumeRole"]
       principals {
         type        = "Service"
         identifiers = ["vpc-flow-logs.amazonaws.com"]
       }
     }
   }

   resource "aws_iam_role" "flow" {
     count              = var.enable_flow_logs ? 1 : 0
     name               = "${var.name}-vpc-flow-logs"
     assume_role_policy = data.aws_iam_policy_document.flow_assume[0].json
   }

   resource "aws_iam_role_policy" "flow" {
     count = var.enable_flow_logs ? 1 : 0
     role  = aws_iam_role.flow[0].id
     policy = jsonencode({
       Version = "2012-10-17"
       Statement = [{
         Effect   = "Allow"
         Action   = ["logs:CreateLogStream", "logs:PutLogEvents", "logs:DescribeLogStreams"]
         Resource = "${aws_cloudwatch_log_group.flow[0].arn}:*"
       }]
     })
   }

   resource "aws_flow_log" "this" {
     count                = var.enable_flow_logs ? 1 : 0
     vpc_id               = aws_vpc.this.id
     traffic_type         = "ALL"
     log_destination_type = "cloud-watch-logs"
     log_destination      = aws_cloudwatch_log_group.flow[0].arn
     iam_role_arn         = aws_iam_role.flow[0].arn
   }
   ```

   ```hcl
   # modules/vpc/outputs.tf
   output "vpc_id" { value = aws_vpc.this.id }
   output "vpc_cidr" { value = aws_vpc.this.cidr_block }
   output "public_subnet_ids" { value = [for k in sort(keys(aws_subnet.public)) : aws_subnet.public[k].id] }
   output "private_subnet_ids" { value = [for k in sort(keys(aws_subnet.private)) : aws_subnet.private[k].id] }
   output "private_subnets_by_az" { value = { for k, s in aws_subnet.private : s.availability_zone => s.id } }
   output "nat_gateway_ids" { value = [for n in aws_nat_gateway.this : n.id] }
   output "endpoint_security_group_id" { value = aws_security_group.endpoints.id }
   ```

2. Create the network layer root module.

   ```hcl
   # live/dev/network/versions.tf
   terraform {
     required_version = ">= 1.10.0, < 2.0.0"
     required_providers {
       aws = { source = "hashicorp/aws", version = "~> 5.80" }
     }
     backend "s3" {
       bucket       = "capstone-tfstate-123456789012"
       key          = "live/dev/network/terraform.tfstate"
       region       = "eu-west-1"
       encrypt      = true
       kms_key_id   = "alias/terraform-state"
       use_lockfile = true
     }
   }

   provider "aws" {
     region = var.region
     default_tags {
       tags = { ManagedBy = "terraform", Environment = var.environment, Layer = "network" }
     }
   }
   ```

   ```hcl
   # live/dev/network/main.tf
   variable "region" { default = "eu-west-1" }
   variable "environment" { default = "dev" }

   data "aws_availability_zones" "available" {
     state = "available"
   }

   module "vpc" {
     source = "../../../modules/vpc"

     name               = "capstone-${var.environment}"
     cidr               = "10.10.0.0/16"
     azs                = slice(data.aws_availability_zones.available.names, 0, 3)
     single_nat_gateway = true
   }

   resource "aws_ssm_parameter" "exports" {
     for_each = {
       vpc_id             = module.vpc.vpc_id
       vpc_cidr           = module.vpc.vpc_cidr
       private_subnet_ids = join(",", module.vpc.private_subnet_ids)
       public_subnet_ids  = join(",", module.vpc.public_subnet_ids)
     }
     name  = "/capstone/${var.environment}/network/${each.key}"
     type  = "String"
     value = each.value
   }

   output "vpc_id" { value = module.vpc.vpc_id }
   output "private_subnet_ids" { value = module.vpc.private_subnet_ids }
   output "public_subnet_ids" { value = module.vpc.public_subnet_ids }
   ```

3. Initialise, plan, and read the plan carefully.

   ```bash
   cd live/dev/network
   terraform init
   terraform plan -out=tfplan
   ```

   Expected output (tail):

   ```text
   Plan: 41 to add, 0 to change, 0 to destroy.

   Changes to Outputs:
     + private_subnet_ids = [
         + (known after apply),
         + (known after apply),
         + (known after apply),
       ]
     + public_subnet_ids  = [ ... ]
     + vpc_id             = (known after apply)
   ```

   Count what the plan contains to check your mental model: 1 VPC, 1 IGW, 3 public + 3 private subnets, 1 EIP, 1 NAT, 1 public RT + 1 route + 3 associations, 3 private RTs + 3 routes + 3 associations, 1 S3 endpoint, 1 SG + 1 rule, 5 interface endpoints, 1 log group, 1 role, 1 role policy, 1 flow log, 4 SSM parameters = 41.

4. Apply.

   ```bash
   terraform apply tfplan
   ```

   Expected: `Apply complete! Resources: 41 added, 0 changed, 0 destroyed.` (3-4 minutes; interface endpoints are the slow part).

5. Render the graph and look at the parallelism structure.

   ```bash
   terraform graph -type=plan | dot -Tsvg > network-graph.svg
   ```

### Verify

```bash
terraform output -json | jq '.private_subnet_ids.value'
aws ssm get-parameters-by-path --path /capstone/dev/network --query 'Parameters[].{Name:Name,Value:Value}' --output table
aws ec2 describe-nat-gateways --filter Name=vpc-id,Values="$(terraform output -raw vpc_id)" --query 'NatGateways[].State'
terraform plan   # No changes.
```

Test the `for_each` stability claim: change `azs` to `slice(..., 0, 2)` and plan (do not apply). The plan should destroy only the `c` subnets and their route tables, not touch `a` and `b`. Revert the change.

### Cleanup

Keep the network layer for Labs 3 and 4. Delete `network-graph.svg`.

### What you learned

Building subnets as a map keyed by AZ letter gives stable addresses; `cidrsubnet` derives non-overlapping ranges deterministically; gateway endpoints are free and interface endpoints are not; SSM parameters make a contract other layers and other tools can consume.

---

## Lab 3: Multi-AZ RDS PostgreSQL with Secrets Manager

**Objective.** Build an `rds-postgres` module with a managed master password (never in state), KMS encryption, parameter group, and `precondition`/`postcondition` guardrails, consumed by the `data` layer via the SSM contract.

**Prerequisites.** Lab 2 applied.

**Time.** 60 minutes (RDS creation takes 8-12 minutes).

### Steps

1. Create the module.

   ```hcl
   # modules/rds-postgres/variables.tf
   variable "name" { type = string }
   variable "vpc_id" { type = string }
   variable "subnet_ids" { type = list(string) }
   variable "allowed_cidrs" {
     description = "CIDRs allowed to reach port 5432."
     type        = list(string)
   }
   variable "environment" { type = string }
   variable "instance_class" {
     type    = string
     default = "db.t4g.micro"
   }
   variable "engine_version" {
     type    = string
     default = "16.4"
   }
   variable "multi_az" {
     type    = bool
     default = false
   }
   variable "allocated_storage" {
     type    = number
     default = 20
   }
   ```

   ```hcl
   # modules/rds-postgres/main.tf
   terraform {
     required_providers {
       aws = { source = "hashicorp/aws", version = ">= 5.40" }
     }
   }

   resource "aws_kms_key" "db" {
     description             = "${var.name} RDS encryption"
     enable_key_rotation     = true
     deletion_window_in_days = 7
   }

   resource "aws_db_subnet_group" "this" {
     name       = var.name
     subnet_ids = var.subnet_ids
   }

   resource "aws_security_group" "db" {
     name        = "${var.name}-db"
     description = "PostgreSQL access"
     vpc_id      = var.vpc_id
   }

   resource "aws_vpc_security_group_ingress_rule" "db" {
     for_each          = toset(var.allowed_cidrs)
     security_group_id = aws_security_group.db.id
     cidr_ipv4         = each.value
     from_port         = 5432
     to_port           = 5432
     ip_protocol       = "tcp"
   }

   resource "aws_db_parameter_group" "this" {
     name   = "${var.name}-pg16"
     family = "postgres16"

     parameter {
       name  = "log_min_duration_statement"
       value = "1000"
     }
     parameter {
       name  = "rds.force_ssl"
       value = "1"
     }
   }

   resource "aws_db_instance" "this" {
     identifier                    = var.name
     engine                        = "postgres"
     engine_version                = var.engine_version
     instance_class                = var.instance_class
     allocated_storage             = var.allocated_storage
     max_allocated_storage         = var.allocated_storage * 5
     storage_type                  = "gp3"
     storage_encrypted             = true
     kms_key_id                    = aws_kms_key.db.arn
     db_name                       = "app"
     username                      = "app_admin"
     manage_master_user_password   = true
     master_user_secret_kms_key_id = aws_kms_key.db.arn
     db_subnet_group_name          = aws_db_subnet_group.this.name
     vpc_security_group_ids        = [aws_security_group.db.id]
     parameter_group_name          = aws_db_parameter_group.this.name
     multi_az                      = var.multi_az
     publicly_accessible           = false
     backup_retention_period       = var.environment == "prod" ? 14 : 1
     backup_window                 = "02:00-03:00"
     maintenance_window            = "sun:03:30-sun:04:30"
     auto_minor_version_upgrade    = true
     deletion_protection           = var.environment == "prod"
     skip_final_snapshot           = var.environment != "prod"
     final_snapshot_identifier     = var.environment == "prod" ? "${var.name}-final" : null
     performance_insights_enabled  = true
     enabled_cloudwatch_logs_exports = ["postgresql", "upgrade"]
     apply_immediately             = var.environment != "prod"

     lifecycle {
       precondition {
         condition     = var.environment != "prod" || var.multi_az
         error_message = "Production databases must be Multi-AZ."
       }
       postcondition {
         condition     = self.storage_encrypted && !self.publicly_accessible
         error_message = "Database must be encrypted and private."
       }
       ignore_changes = [engine_version]   # minor upgrades applied by AWS during maintenance
     }

     timeouts {
       create = "40m"
       update = "80m"
       delete = "40m"
     }
   }
   ```

   ```hcl
   # modules/rds-postgres/outputs.tf
   output "endpoint" { value = aws_db_instance.this.endpoint }
   output "address" { value = aws_db_instance.this.address }
   output "port" { value = aws_db_instance.this.port }
   output "db_name" { value = aws_db_instance.this.db_name }
   output "master_user_secret_arn" {
     value       = aws_db_instance.this.master_user_secret[0].secret_arn
     description = "Secrets Manager ARN holding username/password managed by RDS."
   }
   output "security_group_id" { value = aws_security_group.db.id }
   ```

2. Create the data layer, reading the network contract from SSM.

   ```hcl
   # live/dev/data/main.tf
   terraform {
     required_version = ">= 1.10.0, < 2.0.0"
     required_providers {
       aws = { source = "hashicorp/aws", version = "~> 5.80" }
     }
     backend "s3" {
       bucket       = "capstone-tfstate-123456789012"
       key          = "live/dev/data/terraform.tfstate"
       region       = "eu-west-1"
       encrypt      = true
       kms_key_id   = "alias/terraform-state"
       use_lockfile = true
     }
   }

   provider "aws" {
     region = "eu-west-1"
     default_tags {
       tags = { ManagedBy = "terraform", Environment = "dev", Layer = "data" }
     }
   }

   data "aws_ssm_parameter" "vpc_id" { name = "/capstone/dev/network/vpc_id" }
   data "aws_ssm_parameter" "vpc_cidr" { name = "/capstone/dev/network/vpc_cidr" }
   data "aws_ssm_parameter" "private_subnet_ids" { name = "/capstone/dev/network/private_subnet_ids" }

   module "postgres" {
     source = "../../../modules/rds-postgres"

     name          = "capstone-dev-postgres"
     environment   = "dev"
     vpc_id        = data.aws_ssm_parameter.vpc_id.value
     subnet_ids    = split(",", data.aws_ssm_parameter.private_subnet_ids.value)
     allowed_cidrs = [data.aws_ssm_parameter.vpc_cidr.value]
     multi_az      = false
   }

   resource "aws_ssm_parameter" "exports" {
     for_each = {
       db_endpoint   = module.postgres.endpoint
       db_secret_arn = module.postgres.master_user_secret_arn
       db_sg_id      = module.postgres.security_group_id
     }
     name  = "/capstone/dev/data/${each.key}"
     type  = "String"
     value = each.value
   }

   output "endpoint" { value = module.postgres.endpoint }
   output "master_user_secret_arn" { value = module.postgres.master_user_secret_arn }
   ```

3. Plan, apply, and inspect state for the password.

   ```bash
   cd live/dev/data
   terraform init && terraform apply -auto-approve
   terraform state pull | jq '.resources[] | select(.type=="aws_db_instance") | .instances[0].attributes | {password, master_user_secret}'
   ```

   Expected output:

   ```json
   {
     "password": null,
     "master_user_secret": [
       {
         "kms_key_id": "arn:aws:kms:eu-west-1:123456789012:key/...",
         "secret_arn": "arn:aws:secretsmanager:eu-west-1:123456789012:secret:rds!db-...",
         "secret_status": "active"
       }
     ]
   }
   ```

   The password is not in state; only the ARN of the secret is.

4. Exercise the precondition: temporarily set `environment = "prod"` with `multi_az = false` and run `terraform plan`.

   Expected output:

   ```text
   Error: Resource precondition failed

     on ../../../modules/rds-postgres/main.tf line 61, in resource "aws_db_instance" "this":
     61:       condition     = var.environment != "prod" || var.multi_az

   Production databases must be Multi-AZ.
   ```

   Revert.

### Verify

```bash
aws rds describe-db-instances --db-instance-identifier capstone-dev-postgres \
  --query 'DBInstances[0].{Status:DBInstanceStatus,Encrypted:StorageEncrypted,Public:PubliclyAccessible,MultiAZ:MultiAZ}'
aws secretsmanager describe-secret --secret-id "$(terraform output -raw master_user_secret_arn)" --query 'Name'
```

### Cleanup

Keep for Lab 4 if you continue today; otherwise `terraform destroy -auto-approve` here (about 5 minutes) and re-apply later.

### What you learned

`manage_master_user_password` is the single most effective way to keep a database credential out of state. Preconditions fail fast at plan time; postconditions verify what the provider actually produced. SSM parameters decouple layers so the data layer never needs access to the network layer's state file.

---

## Lab 4: EKS Cluster with `terraform-aws-modules`

**Objective.** Use the community EKS module to provision a cluster with a managed node group, EKS Pod Identity, and the core add-ons, in the compute layer.

**Prerequisites.** Lab 2 applied. `kubectl` installed.

**Time.** 60 minutes (cluster creation 12-15 minutes).

### Steps

1. Write the compute layer.

   ```hcl
   # live/dev/compute/main.tf
   terraform {
     required_version = ">= 1.10.0, < 2.0.0"
     required_providers {
       aws = { source = "hashicorp/aws", version = "~> 5.80" }
     }
     backend "s3" {
       bucket       = "capstone-tfstate-123456789012"
       key          = "live/dev/compute/terraform.tfstate"
       region       = "eu-west-1"
       encrypt      = true
       kms_key_id   = "alias/terraform-state"
       use_lockfile = true
     }
   }

   provider "aws" {
     region = "eu-west-1"
     default_tags {
       tags = { ManagedBy = "terraform", Environment = "dev", Layer = "compute" }
     }
   }

   data "aws_ssm_parameter" "vpc_id" { name = "/capstone/dev/network/vpc_id" }
   data "aws_ssm_parameter" "private_subnet_ids" { name = "/capstone/dev/network/private_subnet_ids" }
   data "aws_caller_identity" "current" {}

   module "eks" {
     source  = "terraform-aws-modules/eks/aws"
     version = "~> 20.31"

     cluster_name    = "capstone-dev"
     cluster_version = "1.31"

     vpc_id     = data.aws_ssm_parameter.vpc_id.value
     subnet_ids = split(",", data.aws_ssm_parameter.private_subnet_ids.value)

     cluster_endpoint_public_access       = true                 # lab convenience; restrict CIDRs or go private in prod
     cluster_endpoint_public_access_cidrs = ["0.0.0.0/0"]        # INSECURE SHORTCUT: set to your IP /32 if possible
     cluster_endpoint_private_access      = true

     enable_cluster_creator_admin_permissions = true
     authentication_mode                      = "API"

     cluster_encryption_config = {
       resources = ["secrets"]
     }

     cluster_enabled_log_types = ["api", "audit", "authenticator"]

     cluster_addons = {
       coredns                = {}
       kube-proxy             = {}
       vpc-cni                = { before_compute = true }
       eks-pod-identity-agent = {}
       aws-ebs-csi-driver     = {}
     }

     eks_managed_node_groups = {
       default = {
         ami_type       = "AL2023_ARM_64_STANDARD"
         instance_types = ["t4g.medium"]
         min_size       = 2
         max_size       = 4
         desired_size   = 2
         labels         = { workload = "general" }
         metadata_options = {
           http_tokens                 = "required"
           http_put_response_hop_limit = 1
         }
       }
     }

     tags = { "karpenter.sh/discovery" = "capstone-dev" }
   }

   resource "aws_ssm_parameter" "exports" {
     for_each = {
       cluster_name     = module.eks.cluster_name
       cluster_endpoint = module.eks.cluster_endpoint
       oidc_provider    = module.eks.oidc_provider
     }
     name  = "/capstone/dev/compute/${each.key}"
     type  = "String"
     value = each.value
   }

   output "cluster_name" { value = module.eks.cluster_name }
   output "cluster_endpoint" { value = module.eks.cluster_endpoint }
   output "configure_kubectl" {
     value = "aws eks update-kubeconfig --region eu-west-1 --name ${module.eks.cluster_name}"
   }
   ```

2. Plan and inspect the module tree.

   ```bash
   cd live/dev/compute
   terraform init
   terraform providers
   terraform plan -out=tfplan | tail -5
   ```

   Expected output (tail):

   ```text
   Plan: 59 to add, 0 to change, 0 to destroy.
   ```

   ```bash
   terraform show -json tfplan | jq -r '.resource_changes[].address' | sed 's/\[.*//' | sort | uniq -c | sort -rn | head
   ```

3. Apply and connect.

   ```bash
   terraform apply tfplan
   $(terraform output -raw configure_kubectl)
   kubectl get nodes -o wide
   ```

   Expected output:

   ```text
   NAME                                       STATUS   ROLES    AGE   VERSION
   ip-10-10-0-123.eu-west-1.compute.internal  Ready    <none>   3m    v1.31.x-eks-...
   ip-10-10-16-45.eu-west-1.compute.internal  Ready    <none>   3m    v1.31.x-eks-...
   ```

4. Observe `ignore_changes` in action: scale the node group manually and plan.

   ```bash
   aws eks update-nodegroup-config --cluster-name capstone-dev --nodegroup-name "$(aws eks list-nodegroups --cluster-name capstone-dev --query 'nodegroups[0]' --output text)" --scaling-config desiredSize=3
   terraform plan
   ```

   The module sets `ignore_changes = [scaling_config[0].desired_size]` on the node group, so the plan shows no changes even though desired size drifted. Scale back to 2.

### Verify

```bash
kubectl get pods -n kube-system
aws eks describe-cluster --name capstone-dev --query 'cluster.{status:status,encryption:encryptionConfig[0].resources,logging:logging.clusterLogging[0].enabled}'
```

### Cleanup

Keep for Lab 7 if continuing today; otherwise destroy (`terraform destroy -auto-approve`, about 10 minutes). Always destroy any Kubernetes `LoadBalancer` Services first or the VPC teardown will fail on orphaned ELBs.

### What you learned

Community modules encode hundreds of provider details (IAM roles, security group rules, add-on ordering). Reading `terraform providers` and the plan's address histogram tells you what a module really creates. The compute layer consumed the network layer without touching its state.

---

## Lab 5: Importing an Existing Resource

**Objective.** Adopt a bucket created outside Terraform using an `import` block and `-generate-config-out`, then refactor with a `moved` block without destroying anything.

**Prerequisites.** Lab 1.

**Time.** 30 minutes.

### Steps

1. Create a "legacy" bucket by hand to simulate pre-Terraform infrastructure.

   ```bash
   aws s3api create-bucket --bucket capstone-legacy-logs-123456789012 --region eu-west-1 \
     --create-bucket-configuration LocationConstraint=eu-west-1
   aws s3api put-bucket-versioning --bucket capstone-legacy-logs-123456789012 --versioning-configuration Status=Enabled
   ```

2. In `live/dev/network/`, declare the import.

   ```hcl
   # live/dev/network/import.tf
   import {
     to = aws_s3_bucket.legacy_logs
     id = "capstone-legacy-logs-123456789012"
   }

   import {
     to = aws_s3_bucket_versioning.legacy_logs
     id = "capstone-legacy-logs-123456789012"
   }
   ```

3. Generate configuration and review.

   ```bash
   terraform plan -generate-config-out=generated.tf
   ```

   Expected output (abridged):

   ```text
   Terraform will perform the following actions:

     # aws_s3_bucket.legacy_logs will be imported
       resource "aws_s3_bucket" "legacy_logs" {
           bucket = "capstone-legacy-logs-123456789012"
           ...
       }

     # aws_s3_bucket_versioning.legacy_logs will be imported
       ...

   Plan: 2 to import, 0 to add, 0 to change, 0 to destroy.

   Terraform has generated configuration and written it to generated.tf. Please review
   the configuration and edit it as necessary before adding it to version control.
   ```

4. Trim `generated.tf` to the essentials and move it into `main.tf`.

   ```hcl
   resource "aws_s3_bucket" "legacy_logs" {
     bucket = "capstone-legacy-logs-123456789012"
   }

   resource "aws_s3_bucket_versioning" "legacy_logs" {
     bucket = aws_s3_bucket.legacy_logs.id
     versioning_configuration {
       status = "Enabled"
     }
   }
   ```

   ```bash
   rm generated.tf
   terraform plan      # still "2 to import", nothing else
   terraform apply -auto-approve
   ```

   Expected: `Apply complete! Resources: 2 imported, 0 added, 0 changed, 0 destroyed.`

5. Refactor: rename to `alb_logs` with a `moved` block.

   ```hcl
   moved {
     from = aws_s3_bucket.legacy_logs
     to   = aws_s3_bucket.alb_logs
   }
   moved {
     from = aws_s3_bucket_versioning.legacy_logs
     to   = aws_s3_bucket_versioning.alb_logs
   }
   ```

   Rename the resource blocks accordingly, then:

   ```bash
   terraform plan
   ```

   Expected output:

   ```text
     # aws_s3_bucket.legacy_logs has moved to aws_s3_bucket.alb_logs
       resource "aws_s3_bucket" "alb_logs" { ... }

   Plan: 0 to add, 0 to change, 0 to destroy.
   ```

   ```bash
   terraform apply -auto-approve
   ```

6. Remove the `import` and `moved` blocks (they have done their job) and confirm `terraform plan` is clean.

### Verify

```bash
terraform state list | grep alb_logs
terraform state show aws_s3_bucket.alb_logs | grep bucket
```

### Cleanup

Remove the two resource blocks and apply: Terraform destroys the bucket (it is empty). Alternatively practise `removed { from = aws_s3_bucket.alb_logs  lifecycle { destroy = false } }` and delete the bucket with the CLI afterwards.

### What you learned

Import blocks make adoption a reviewed plan, not a side effect. Generated configuration is a starting point, never the final code. `moved` blocks let you refactor addresses with zero infrastructure change.

---

## Lab 6: Writing `terraform test` for the VPC Module

**Objective.** Add unit tests (mocked provider, `command = plan`) and one integration test (`command = apply`) for `modules/vpc`.

**Prerequisites.** Lab 2 module written.

**Time.** 40 minutes.

### Steps

1. Create the test file.

   ```hcl
   # modules/vpc/tests/unit.tftest.hcl
   mock_provider "aws" {
     mock_data "aws_region" {
       defaults = { name = "eu-west-1" }
     }
   }

   variables {
     name = "test"
     cidr = "10.99.0.0/16"
     azs  = ["eu-west-1a", "eu-west-1b", "eu-west-1c"]
   }

   run "subnet_layout" {
     command = plan

     assert {
       condition     = length(aws_subnet.private) == 3 && length(aws_subnet.public) == 3
       error_message = "Expected 3 private and 3 public subnets."
     }

     assert {
       condition     = aws_subnet.private["a"].cidr_block == "10.99.0.0/20"
       error_message = "First private subnet should be the first /20."
     }

     assert {
       condition     = alltrue([for s in aws_subnet.public : !s.map_public_ip_on_launch])
       error_message = "Public subnets must not auto-assign public IPs."
     }
   }

   run "single_nat_by_default" {
     command = plan
     assert {
       condition     = length(aws_nat_gateway.this) == 1
       error_message = "Default should be one NAT gateway."
     }
   }

   run "nat_per_az_when_requested" {
     command = plan
     variables {
       single_nat_gateway = false
     }
     assert {
       condition     = length(aws_nat_gateway.this) == 3
       error_message = "Expected one NAT gateway per AZ."
     }
   }

   run "rejects_single_az" {
     command = plan
     variables {
       azs = ["eu-west-1a"]
     }
     expect_failures = [var.azs]
   }

   run "rejects_tiny_cidr" {
     command = plan
     variables {
       cidr = "10.99.0.0/24"
     }
     expect_failures = [var.cidr]
   }

   run "flow_logs_optional" {
     command = plan
     variables {
       enable_flow_logs = false
     }
     assert {
       condition     = length(aws_flow_log.this) == 0
       error_message = "Flow logs should be absent when disabled."
     }
   }
   ```

2. Run the unit tests (no credentials needed).

   ```bash
   cd modules/vpc
   terraform init -backend=false
   terraform test
   ```

   Expected output:

   ```text
   tests/unit.tftest.hcl... in progress
     run "subnet_layout"... pass
     run "single_nat_by_default"... pass
     run "nat_per_az_when_requested"... pass
     run "rejects_single_az"... pass
     run "rejects_tiny_cidr"... pass
     run "flow_logs_optional"... pass
   tests/unit.tftest.hcl... tearing down
   tests/unit.tftest.hcl... pass

   Success! 6 passed, 0 failed.
   ```

3. Add an integration test that applies for real and cleans up automatically.

   ```hcl
   # modules/vpc/tests/integration.tftest.hcl
   provider "aws" {
     region = "eu-west-1"
   }

   variables {
     name                = "tftest"
     cidr                = "10.77.0.0/16"
     azs                 = ["eu-west-1a", "eu-west-1b"]
     interface_endpoints = []            # keep the test cheap and fast
     enable_flow_logs    = false
   }

   run "creates_real_vpc" {
     command = apply

     assert {
       condition     = startswith(output.vpc_id, "vpc-")
       error_message = "vpc_id output should be a real VPC ID."
     }
     assert {
       condition     = length(output.private_subnet_ids) == 2
       error_message = "Two private subnets expected."
     }
   }
   ```

   ```bash
   terraform test -filter=tests/integration.tftest.hcl -verbose
   ```

   Terraform applies, runs the assertions, then destroys everything it created (about 3 minutes because of the NAT gateway).

4. Break a test on purpose: change the first assertion to expect `/21` and rerun to see a failure report.

   Expected output (abridged):

   ```text
     run "subnet_layout"... fail
   ╷
   │ Error: Test assertion failed
   │   on tests/unit.tftest.hcl line 20, in run "subnet_layout":
   │ First private subnet should be the first /20.
   ╵
   Failure! 5 passed, 1 failed.
   ```

### Verify

Wire the unit tests into pre-commit or CI (`terraform test` in `modules/vpc`); they run in seconds with the mocked provider.

### Cleanup

Automatic for the integration run. Confirm nothing is left: `aws ec2 describe-vpcs --filters Name=tag:Name,Values=tftest --query 'Vpcs[].VpcId'` should return `[]`.

### What you learned

Mock providers make module unit tests free and fast; `expect_failures` tests your validation rules; `command = apply` runs are the integration layer and clean up after themselves.

---

## Lab 7: GitHub Actions OIDC Pipeline (Plan on PR, Apply on Merge)

**Objective.** Create the GitHub OIDC identity provider and a least-privilege role with Terraform, then a workflow that plans on pull requests (posting the plan as a comment) and applies on merge to `main`, with lock timeouts and no static credentials.

**Prerequisites.** Lab 1; a GitHub repository `<ORG>/capstone-infra` containing this code.

**Time.** 60 minutes.

### Steps

1. Create the OIDC provider and role in `global/iam/`.

   ```hcl
   # global/iam/main.tf
   terraform {
     required_version = ">= 1.10.0, < 2.0.0"
     required_providers {
       aws = { source = "hashicorp/aws", version = "~> 5.80" }
     }
     backend "s3" {
       bucket       = "capstone-tfstate-123456789012"
       key          = "global/iam/terraform.tfstate"
       region       = "eu-west-1"
       encrypt      = true
       kms_key_id   = "alias/terraform-state"
       use_lockfile = true
     }
   }

   provider "aws" { region = "eu-west-1" }

   variable "github_org" { default = "<ORG>" }
   variable "github_repo" { default = "capstone-infra" }

   data "aws_caller_identity" "current" {}

   resource "aws_iam_openid_connect_provider" "github" {
     url             = "https://token.actions.githubusercontent.com"
     client_id_list  = ["sts.amazonaws.com"]
     thumbprint_list = ["6938fd4d98bab03faadb97b34396831e3780aea1"]   # ignored by AWS for GitHub since 2023 but required by the API
   }

   # Plan role: read-only plus state access. Any branch / PR may assume it.
   data "aws_iam_policy_document" "plan_trust" {
     statement {
       actions = ["sts:AssumeRoleWithWebIdentity"]
       principals {
         type        = "Federated"
         identifiers = [aws_iam_openid_connect_provider.github.arn]
       }
       condition {
         test     = "StringEquals"
         variable = "token.actions.githubusercontent.com:aud"
         values   = ["sts.amazonaws.com"]
       }
       condition {
         test     = "StringLike"
         variable = "token.actions.githubusercontent.com:sub"
         values   = ["repo:${var.github_org}/${var.github_repo}:*"]
       }
     }
   }

   # Apply role: only the main branch (or a protected environment) may assume it.
   data "aws_iam_policy_document" "apply_trust" {
     statement {
       actions = ["sts:AssumeRoleWithWebIdentity"]
       principals {
         type        = "Federated"
         identifiers = [aws_iam_openid_connect_provider.github.arn]
       }
       condition {
         test     = "StringEquals"
         variable = "token.actions.githubusercontent.com:aud"
         values   = ["sts.amazonaws.com"]
       }
       condition {
         test     = "StringEquals"
         variable = "token.actions.githubusercontent.com:sub"
         values = [
           "repo:${var.github_org}/${var.github_repo}:ref:refs/heads/main",
           "repo:${var.github_org}/${var.github_repo}:environment:dev",
         ]
       }
     }
   }

   data "aws_iam_policy_document" "state" {
     statement {
       actions   = ["s3:ListBucket", "s3:GetBucketVersioning"]
       resources = ["arn:aws:s3:::capstone-tfstate-${data.aws_caller_identity.current.account_id}"]
     }
     statement {
       actions   = ["s3:GetObject", "s3:PutObject", "s3:DeleteObject"]
       resources = ["arn:aws:s3:::capstone-tfstate-${data.aws_caller_identity.current.account_id}/live/*"]
     }
     statement {
       actions   = ["kms:Decrypt", "kms:GenerateDataKey", "kms:DescribeKey"]
       resources = ["*"]
       condition {
         test     = "StringEquals"
         variable = "kms:ResourceAliases"
         values   = ["alias/terraform-state"]
       }
     }
   }

   resource "aws_iam_role" "plan" {
     name                 = "github-terraform-plan"
     assume_role_policy   = data.aws_iam_policy_document.plan_trust.json
     max_session_duration = 3600
   }

   resource "aws_iam_role_policy" "plan_state" {
     role   = aws_iam_role.plan.id
     policy = data.aws_iam_policy_document.state.json
   }

   resource "aws_iam_role_policy_attachment" "plan_readonly" {
     role       = aws_iam_role.plan.name
     policy_arn = "arn:aws:iam::aws:policy/ReadOnlyAccess"
   }

   resource "aws_iam_role" "apply" {
     name                 = "github-terraform-apply"
     assume_role_policy   = data.aws_iam_policy_document.apply_trust.json
     max_session_duration = 3600
   }

   resource "aws_iam_role_policy" "apply_state" {
     role   = aws_iam_role.apply.id
     policy = data.aws_iam_policy_document.state.json
   }

   # Lab shortcut: PowerUserAccess + IAM scoped by a permissions boundary would be the production pattern.
   resource "aws_iam_role_policy_attachment" "apply_power" {
     role       = aws_iam_role.apply.name
     policy_arn = "arn:aws:iam::aws:policy/AdministratorAccess"   # INSECURE SHORTCUT for the lab; narrow per layer in production
   }

   output "plan_role_arn" { value = aws_iam_role.plan.arn }
   output "apply_role_arn" { value = aws_iam_role.apply.arn }
   ```

   ```bash
   cd global/iam && terraform init && terraform apply -auto-approve
   ```

2. Create the workflow.

   ```yaml
   # .github/workflows/terraform.yml
   name: terraform

   on:
     pull_request:
       paths: ["live/dev/**", "modules/**"]
     push:
       branches: [main]
       paths: ["live/dev/**", "modules/**"]

   permissions:
     id-token: write        # OIDC token
     contents: read
     pull-requests: write   # plan comments

   concurrency:
     group: terraform-dev-${{ github.ref }}
     cancel-in-progress: false

   env:
     TF_IN_AUTOMATION: "1"
     TF_INPUT: "0"
     AWS_REGION: eu-west-1

   jobs:
     plan:
       if: github.event_name == 'pull_request'
       runs-on: ubuntu-latest
       strategy:
         fail-fast: false
         matrix:
           layer: [network, data, compute]
       steps:
         - uses: actions/checkout@v4

         - uses: aws-actions/configure-aws-credentials@v4
           with:
             role-to-assume: arn:aws:iam::123456789012:role/github-terraform-plan
             aws-region: ${{ env.AWS_REGION }}

         - uses: hashicorp/setup-terraform@v3
           with:
             terraform_version: 1.10.5
             terraform_wrapper: false

         - name: Init
           working-directory: live/dev/${{ matrix.layer }}
           run: terraform init -lockfile=readonly

         - name: Format and validate
           working-directory: live/dev/${{ matrix.layer }}
           run: |
             terraform fmt -check -recursive
             terraform validate

         - name: Lint
           uses: terraform-linters/setup-tflint@v4
         - run: tflint --init && tflint --recursive --format compact

         - name: Checkov
           uses: bridgecrewio/checkov-action@v12
           with:
             directory: live/dev/${{ matrix.layer }}
             framework: terraform
             quiet: true
             soft_fail: false

         - name: Plan
           id: plan
           working-directory: live/dev/${{ matrix.layer }}
           run: |
             set +e
             terraform plan -no-color -lock-timeout=5m -detailed-exitcode -out=tfplan > plan.txt 2>&1
             echo "exitcode=$?" >> "$GITHUB_OUTPUT"
             set -e
             terraform show -json tfplan > plan.json
             tail -c 60000 plan.txt > plan-trunc.txt

         - name: Fail on plan error
           if: steps.plan.outputs.exitcode == '1'
           run: cat live/dev/${{ matrix.layer }}/plan.txt && exit 1

         - name: Policy on plan JSON
           if: steps.plan.outputs.exitcode == '2'
           uses: bridgecrewio/checkov-action@v12
           with:
             file: live/dev/${{ matrix.layer }}/plan.json
             framework: terraform_plan

         - name: Comment plan on PR
           uses: actions/github-script@v7
           env:
             LAYER: ${{ matrix.layer }}
             EXIT: ${{ steps.plan.outputs.exitcode }}
           with:
             script: |
               const fs = require('fs');
               const plan = fs.readFileSync(`live/dev/${process.env.LAYER}/plan-trunc.txt`, 'utf8');
               const status = process.env.EXIT === '2' ? 'changes pending' : 'no changes';
               const body = `### Terraform plan: \`${process.env.LAYER}\` (${status})\n<details><summary>Show plan</summary>\n\n\`\`\`text\n${plan}\n\`\`\`\n</details>`;
               const { data: comments } = await github.rest.issues.listComments({ ...context.repo, issue_number: context.issue.number });
               const existing = comments.find(c => c.body.startsWith(`### Terraform plan: \`${process.env.LAYER}\``));
               if (existing) {
                 await github.rest.issues.updateComment({ ...context.repo, comment_id: existing.id, body });
               } else {
                 await github.rest.issues.createComment({ ...context.repo, issue_number: context.issue.number, body });
               }

     apply:
       if: github.event_name == 'push'
       runs-on: ubuntu-latest
       environment: dev                       # protected environment: reviewers, wait timer
       strategy:
         max-parallel: 1                      # layers in dependency order
         matrix:
           layer: [network, data, compute]
       steps:
         - uses: actions/checkout@v4
         - uses: aws-actions/configure-aws-credentials@v4
           with:
             role-to-assume: arn:aws:iam::123456789012:role/github-terraform-apply
             aws-region: ${{ env.AWS_REGION }}
         - uses: hashicorp/setup-terraform@v3
           with:
             terraform_version: 1.10.5
             terraform_wrapper: false
         - name: Init
           working-directory: live/dev/${{ matrix.layer }}
           run: terraform init -lockfile=readonly
         - name: Plan
           working-directory: live/dev/${{ matrix.layer }}
           run: terraform plan -no-color -lock-timeout=10m -out=tfplan
         - name: Apply saved plan
           working-directory: live/dev/${{ matrix.layer }}
           run: terraform apply -no-color -lock-timeout=10m tfplan
   ```

   The `strategy.matrix` with `max-parallel: 1` is a simple way to apply layers in order; the matrix order is network, data, compute. For stricter ordering use separate jobs with `needs:`.

3. Open a pull request that changes a tag on the VPC and watch the plan comment appear. Merge and watch the apply job assume the apply role.

### Verify

- In the plan job logs, `aws sts get-caller-identity` (add a step) shows `assumed-role/github-terraform-plan/GitHubActions`.
- In AWS CloudTrail, `AssumeRoleWithWebIdentity` events show the `sub` claim `repo:<ORG>/capstone-infra:pull_request` for plans and `...:ref:refs/heads/main` for applies.
- Try to assume the apply role from a feature branch (temporarily change the role in the plan job): expect `Not authorized to perform sts:AssumeRoleWithWebIdentity`.

### Cleanup

Keep for the capstone. Lab 8 destroys the IAM layer last.

### What you learned

OIDC trust conditions on `sub` are the authorisation boundary between "anyone can plan" and "only main can apply". Applying the saved plan file guarantees you apply exactly what was reviewed. `-lock-timeout` turns lock contention from a failure into a wait.

---

## Lab 8: Controlled Teardown

**Objective.** Destroy the estate in reverse dependency order, handle `prevent_destroy`, and leave the account clean.

**Prerequisites.** Any subset of Labs 1-7 applied.

**Time.** 30 minutes.

### Steps

1. Remove Kubernetes-created cloud resources first (load balancers, EBS volumes from PVCs).

   ```bash
   kubectl delete svc --all-namespaces --field-selector spec.type=LoadBalancer --all 2>/dev/null || true
   kubectl delete pvc --all --all-namespaces 2>/dev/null || true
   ```

2. Destroy layers in reverse order; plan the destroy first and read it.

   ```bash
   for layer in compute data network; do
     echo "== $layer =="
     terraform -chdir="live/dev/$layer" plan -destroy -out=destroy.tfplan -lock-timeout=5m
     terraform -chdir="live/dev/$layer" apply destroy.tfplan
   done
   ```

   Expected (compute): `Destroy complete! Resources: 59 destroyed.` (10-12 minutes). Data: `Resources: 11 destroyed.` Network: `Resources: 41 destroyed.`

3. Destroy the IAM layer.

   ```bash
   terraform -chdir=global/iam destroy -auto-approve
   ```

4. Destroy the state backend. `prevent_destroy` will refuse:

   ```bash
   terraform -chdir=global/state-backend destroy
   ```

   Expected output:

   ```text
   Error: Instance cannot be destroyed

     on main.tf line 25:
     25: resource "aws_s3_bucket" "state" {

   Resource aws_s3_bucket.state has lifecycle.prevent_destroy set, but the plan calls for
   this resource to be destroyed. To avoid this error and continue with the plan, either
   disable lifecycle.prevent_destroy or reduce the scope of the plan using the -target flag.
   ```

   This is the guardrail working. To proceed deliberately: migrate state back to local, remove the `prevent_destroy` line, empty the versioned bucket, and destroy.

   ```bash
   cd global/state-backend
   # 1. move state out of the bucket we are about to delete
   sed -i.bak '/backend "s3"/,/^  }/d' backend.tf && rm backend.tf.bak   # or delete backend.tf entirely
   terraform init -migrate-state
   # 2. relax the guardrail
   sed -i.bak 's/prevent_destroy = true/prevent_destroy = false/' main.tf
   # 3. empty all object versions (versioned buckets refuse deletion otherwise)
   BUCKET=capstone-tfstate-123456789012
   aws s3api list-object-versions --bucket "$BUCKET" --output json \
     --query '{Objects: [].{Key:Key,VersionId:VersionId}}' > versions.json
   aws s3api list-object-versions --bucket "$BUCKET" --output json \
     --query '{Objects: DeleteMarkers[].{Key:Key,VersionId:VersionId}}' > markers.json
   [ "$(jq '.Objects | length' versions.json)" -gt 0 ] && aws s3api delete-objects --bucket "$BUCKET" --delete file://versions.json
   [ "$(jq '.Objects | length' markers.json)" -gt 0 ] && aws s3api delete-objects --bucket "$BUCKET" --delete file://markers.json
   # 4. destroy
   terraform destroy -auto-approve
   ```

### Verify

```bash
aws ec2 describe-vpcs --filters Name=tag:ManagedBy,Values=terraform --query 'Vpcs[].VpcId'      # []
aws rds describe-db-instances --query 'DBInstances[].DBInstanceIdentifier'                     # []
aws eks list-clusters                                                                            # {"clusters": []}
aws s3 ls | grep capstone || echo "no capstone buckets"
aws kms list-aliases --query "Aliases[?starts_with(AliasName,'alias/terraform')]"               # [] (keys are pending deletion)
```

### Cleanup

Delete local `.terraform/` directories and any `*.tfplan` files. KMS keys remain in "pending deletion" for the window you configured.

### What you learned

Teardown order is the dependency graph reversed, across layers as well as within them. `prevent_destroy` does its job even against you. Versioned buckets need every version removed before they can be deleted, which is exactly why they protect your state.

---

## Where Next

- [05-production-patterns.md](05-production-patterns.md) turns these labs into an operable estate: layered state at scale, policy enforcement, secrets, drift schedules, Terragrunt, and OpenTofu.
- Rebuild Labs 2-4 in [../pulumi/04-hands-on-labs.md](../pulumi/04-hands-on-labs.md) and one service in [../cloudformation/04-hands-on-labs.md](../cloudformation/04-hands-on-labs.md) to complete the Phase 2 capstone.
