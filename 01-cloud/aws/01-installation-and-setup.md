# AWS: Installation and Setup

| | |
|---|---|
| **Cluster** | 01 Hyperscale Cloud |
| **Role** | Anchor |
| **Prerequisites** | A new AWS account you control, a password manager, a hardware or TOTP MFA device, Linux/macOS/WSL shell |
| **Target version** | AWS CLI v2.15+, Session Manager plugin 1.2+, `eksctl` 0.190+, `kubectl` 1.31+ |
| **Capstone phase** | Phase 2 (Months 5-8) |

> After this file you will have AWS CLI v2 installed and authenticating through IAM Identity Center (SSO) with no long-lived access keys, an account with its root user locked down, account-wide guardrails (S3 Block Public Access, default EBS encryption, no default VPCs in unused regions), a $10 budget alarm, and the supporting tools (`eksctl`, `kubectl`, `helm`, Session Manager plugin, `jq`) verified. Nothing in this file costs money except one CloudTrail trail's S3 storage (cents).

---

## 1. Account Bootstrap (do this once, by hand, in the console)

The root user exists before IAM does, so the first few steps are necessarily manual.

1. **Create the account** with an email alias you control (`aws-lab@example.com`). Use a plus-address or group alias so the account is not tied to one person's mailbox.
2. **Enable MFA on root** (IAM console -> Security credentials). Prefer a FIDO2 hardware key; AWS allows up to eight MFA devices on root, so register a backup.
3. **Delete any root access keys.** Root should never have access keys. Verify later with `aws iam get-account-summary` (`AccountAccessKeysPresent` must be `0`).
4. **Set the account alias** so sign-in URLs are readable: `https://capstone-lab.signin.aws.amazon.com/console`.
5. **Enable IAM Identity Center** (formerly AWS SSO) in `us-east-1`. Choose "Identity Center directory" as the identity source for a solo lab. In an enterprise it would be Entra ID or Okta via SCIM.
6. **Create one Identity Center user** for yourself, a group `platform-admins`, and a permission set `AdministratorAccess` (session duration 4 hours). Assign group -> permission set -> account.
7. **Enable AWS Organizations** with all features (not consolidated billing only). Even a single-account lab benefits because SCPs, org-wide CloudTrail, and delegated administrators all require it. The capstone later adds a second account.

From here on, every step is CLI.

---

## 2. Install AWS CLI v2

AWS CLI v1 is end-of-life for new features. Only v2 supports `aws configure sso`, `aws sso login`, auto-prompt, YAML output, and the `aws s3` improvements used here.

### Linux (x86_64 and ARM64)

```bash
ARCH=$(uname -m)   # x86_64 or aarch64
curl -fsSL "https://awscli.amazonaws.com/awscli-exe-linux-${ARCH}.zip" -o /tmp/awscliv2.zip
unzip -q /tmp/awscliv2.zip -d /tmp
sudo /tmp/aws/install --update
rm -rf /tmp/aws /tmp/awscliv2.zip
aws --version
```

Expected output:

```text
aws-cli/2.17.50 Python/3.12.6 Linux/6.8.0-45-generic exe/x86_64.ubuntu.24
```

### macOS

```bash
# Homebrew (recommended; tracks the latest v2)
brew install awscli

# or the official pkg
curl -fsSL "https://awscli.amazonaws.com/AWSCLIV2.pkg" -o /tmp/AWSCLIV2.pkg
sudo installer -pkg /tmp/AWSCLIV2.pkg -target /
```

### Windows

```powershell
winget install --id Amazon.AWSCLI --exact
# or
msiexec.exe /i https://awscli.amazonaws.com/AWSCLIV2.msi /qn
aws --version
```

For this wiki, prefer WSL2 (Ubuntu) on Windows so that every Bash example runs unchanged.

### Container

```bash
docker run --rm -it \
  -v "$HOME/.aws:/root/.aws:ro" \
  public.ecr.aws/aws-cli/aws-cli:2.17.50 sts get-caller-identity
```

Mounting `~/.aws` read-only works for SSO because the token cache lives in `~/.aws/sso/cache`; mount it read-write if the container needs to run `aws sso login`.

### Shell completion and defaults

```bash
# Bash
echo 'complete -C "$(command -v aws_completer)" aws' >> ~/.bashrc
# Zsh
echo 'autoload bashcompinit && bashcompinit && complete -C "$(command -v aws_completer)" aws' >> ~/.zshrc
```

Set sane global defaults in `~/.aws/config` (created in the next section):

```ini
[default]
region = us-east-1
output = json
cli_pager =
cli_auto_prompt = on-partial
```

`cli_pager =` (empty) disables the pager so output streams straight to the terminal; `cli_auto_prompt = on-partial` gives interactive completion only when a command is incomplete.

---

## 3. Authenticate with IAM Identity Center (SSO)

### 3.1 Configure the SSO session and profile

```bash
aws configure sso
```

The prompts and reasonable answers:

```text
SSO session name (Recommended): capstone
SSO start URL [None]: https://d-9067xxxxxx.awsapps.com/start
SSO region [None]: us-east-1
SSO registration scopes [sso:account:access]:
Attempting to automatically open the SSO authorization page in your default browser.
...
The only AWS account available to you is: 123456789012
Using the account ID 123456789012
The only role available to you is: AdministratorAccess
Using the role name "AdministratorAccess"
Default client Region [None]: us-east-1
CLI default output format (json if not specified) [None]: json
Profile name [AdministratorAccess-123456789012]: lab-admin
```

This writes two blocks to `~/.aws/config`:

```ini
[sso-session capstone]
sso_start_url = https://d-9067xxxxxx.awsapps.com/start
sso_region = us-east-1
sso_registration_scopes = sso:account:access

[profile lab-admin]
sso_session = capstone
sso_account_id = 123456789012
sso_role_name = AdministratorAccess
region = us-east-1
output = json
```

Add a least-privilege daily-driver profile later (Lab 1 creates a `PowerUserAccess`-style permission set):

```ini
[profile lab-dev]
sso_session = capstone
sso_account_id = 123456789012
sso_role_name = CapstoneDeveloper
region = us-east-1
```

### 3.2 Log in and verify

```bash
aws sso login --sso-session capstone
aws sts get-caller-identity --profile lab-admin
```

Expected output:

```json
{
    "UserId": "AROAEXAMPLEID:you@example.com",
    "Account": "123456789012",
    "Arn": "arn:aws:sts::123456789012:assumed-role/AWSReservedSSO_AdministratorAccess_abcdef0123456789/you@example.com"
}
```

Note the ARN: an SSO permission set is an IAM role named `AWSReservedSSO_<PermissionSet>_<hash>`, and your session is an assumed-role session. This is the identity chain from the [README mental model](README.md#2-identity-is-a-chain-of-short-lived-credentials).

Set the default profile for the shell so you can omit `--profile`:

```bash
export AWS_PROFILE=lab-admin
aws sts get-caller-identity --query Account --output text
# 123456789012
```

### 3.3 Exporting credentials for tools that do not understand SSO

Most modern tools (Terraform AWS provider 5+, `eksctl`, `kubectl` via `aws eks get-token`, boto3 1.26+) read SSO profiles natively. For anything that needs environment variables:

```bash
eval "$(aws configure export-credentials --profile lab-admin --format env)"
env | grep '^AWS_' | sed 's/=.*/=<redacted>/'
```

```text
AWS_ACCESS_KEY_ID=<redacted>
AWS_SECRET_ACCESS_KEY=<redacted>
AWS_SESSION_TOKEN=<redacted>
AWS_CREDENTIAL_EXPIRATION=<redacted>
```

These are the one-hour role-session credentials, not long-lived keys. Never paste them into a file.

### 3.4 Credential resolution order (know this for debugging)

The CLI and all SDKs resolve credentials in this order; the first hit wins:

1. Command-line options (`--profile`, `--region`)
2. Environment variables (`AWS_ACCESS_KEY_ID`, `AWS_PROFILE`, `AWS_REGION`, ...)
3. Web identity token file (`AWS_WEB_IDENTITY_TOKEN_FILE` + `AWS_ROLE_ARN`), used by IRSA
4. `~/.aws/credentials` shared file
5. `~/.aws/config` profile (including `sso_session` and `credential_process`)
6. Container credential provider (`AWS_CONTAINER_CREDENTIALS_FULL_URI`), used by ECS and EKS Pod Identity
7. EC2 Instance Metadata Service (IMDSv2)

If `get-caller-identity` returns an unexpected principal, walk this list.

---

## 4. Account-Wide Guardrails (run once per account)

These commands turn on the defaults every production account should have. They are idempotent and free.

```bash
# 4.1 Block public S3 access at the account level (covers every current and future bucket)
aws s3control put-public-access-block \
  --account-id 123456789012 \
  --public-access-block-configuration \
    BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true

# 4.2 Encrypt every new EBS volume by default (per region)
aws ec2 enable-ebs-encryption-by-default --region us-east-1
aws ec2 get-ebs-encryption-by-default --region us-east-1 --query EbsEncryptionByDefault
# true

# 4.3 Require IMDSv2 on all new instances launched in the region
aws ec2 modify-instance-metadata-defaults --region us-east-1 \
  --http-tokens required --http-put-response-hop-limit 2

# 4.4 Confirm root has no access keys and MFA is on
aws iam get-account-summary \
  --query 'SummaryMap.{RootKeys:AccountAccessKeysPresent,RootMFA:AccountMFAEnabled}'
# { "RootKeys": 0, "RootMFA": 1 }

# 4.5 Set a strong IAM password policy (for the rare break-glass IAM user)
aws iam update-account-password-policy \
  --minimum-password-length 16 --require-symbols --require-numbers \
  --require-uppercase-characters --require-lowercase-characters \
  --max-password-age 90 --password-reuse-prevention 24 --no-allow-users-to-change-password

# 4.6 Enable a multi-region CloudTrail trail to a dedicated bucket (storage costs cents per month)
TRAIL_BUCKET="capstone-cloudtrail-123456789012"
aws s3api create-bucket --bucket "$TRAIL_BUCKET" --region us-east-1
aws s3api put-bucket-policy --bucket "$TRAIL_BUCKET" --policy "$(cat <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AWSCloudTrailAclCheck",
      "Effect": "Allow",
      "Principal": {"Service": "cloudtrail.amazonaws.com"},
      "Action": "s3:GetBucketAcl",
      "Resource": "arn:aws:s3:::${TRAIL_BUCKET}",
      "Condition": {"StringEquals": {"aws:SourceArn": "arn:aws:cloudtrail:us-east-1:123456789012:trail/capstone-trail"}}
    },
    {
      "Sid": "AWSCloudTrailWrite",
      "Effect": "Allow",
      "Principal": {"Service": "cloudtrail.amazonaws.com"},
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::${TRAIL_BUCKET}/AWSLogs/123456789012/*",
      "Condition": {
        "StringEquals": {
          "s3:x-amz-acl": "bucket-owner-full-control",
          "aws:SourceArn": "arn:aws:cloudtrail:us-east-1:123456789012:trail/capstone-trail"
        }
      }
    }
  ]
}
EOF
)"
aws cloudtrail create-trail --name capstone-trail --s3-bucket-name "$TRAIL_BUCKET" \
  --is-multi-region-trail --enable-log-file-validation
aws cloudtrail start-logging --name capstone-trail
aws cloudtrail get-trail-status --name capstone-trail --query IsLogging
# true
```

### 4.7 Delete default VPCs in every region you will not use

A default VPC in `ap-southeast-2` is an invitation to launch resources where you will never look. This loop removes them everywhere except `us-east-1` and `us-east-2` (keep two regions for the DR lab).

```bash
for REGION in $(aws ec2 describe-regions --query 'Regions[].RegionName' --output text); do
  case "$REGION" in us-east-1|us-east-2) continue ;; esac
  VPC_ID=$(aws ec2 describe-vpcs --region "$REGION" --filters Name=isDefault,Values=true \
            --query 'Vpcs[0].VpcId' --output text)
  [ "$VPC_ID" = "None" ] && continue
  echo "Deleting default VPC $VPC_ID in $REGION"
  IGW=$(aws ec2 describe-internet-gateways --region "$REGION" \
          --filters Name=attachment.vpc-id,Values="$VPC_ID" --query 'InternetGateways[0].InternetGatewayId' --output text)
  [ "$IGW" != "None" ] && aws ec2 detach-internet-gateway --region "$REGION" --internet-gateway-id "$IGW" --vpc-id "$VPC_ID" \
                        && aws ec2 delete-internet-gateway --region "$REGION" --internet-gateway-id "$IGW"
  for SUBNET in $(aws ec2 describe-subnets --region "$REGION" --filters Name=vpc-id,Values="$VPC_ID" --query 'Subnets[].SubnetId' --output text); do
    aws ec2 delete-subnet --region "$REGION" --subnet-id "$SUBNET"
  done
  aws ec2 delete-vpc --region "$REGION" --vpc-id "$VPC_ID"
done
```

Also opt out of regions you will never use at the account level so nothing can be created there at all:

```bash
aws account disable-region --region-name ap-east-1
aws account list-regions --region-opt-status-contains DISABLED --query 'Regions[].RegionName'
```

---

## 5. Budget Alarm (before creating anything that costs money)

```bash
aws budgets create-budget --account-id 123456789012 \
  --budget '{
    "BudgetName": "capstone-lab-monthly",
    "BudgetLimit": {"Amount": "10", "Unit": "USD"},
    "TimeUnit": "MONTHLY",
    "BudgetType": "COST",
    "CostFilters": {},
    "CostTypes": {"IncludeCredit": false, "IncludeRefund": false}
  }' \
  --notifications-with-subscribers '[
    {"Notification": {"NotificationType": "ACTUAL", "ComparisonOperator": "GREATER_THAN", "Threshold": 50, "ThresholdType": "PERCENTAGE"},
     "Subscribers": [{"SubscriptionType": "EMAIL", "Address": "aws-lab@example.com"}]},
    {"Notification": {"NotificationType": "ACTUAL", "ComparisonOperator": "GREATER_THAN", "Threshold": 80, "ThresholdType": "PERCENTAGE"},
     "Subscribers": [{"SubscriptionType": "EMAIL", "Address": "aws-lab@example.com"}]},
    {"Notification": {"NotificationType": "FORECASTED", "ComparisonOperator": "GREATER_THAN", "Threshold": 100, "ThresholdType": "PERCENTAGE"},
     "Subscribers": [{"SubscriptionType": "EMAIL", "Address": "aws-lab@example.com"}]}
  ]'

aws budgets describe-budgets --account-id 123456789012 \
  --query 'Budgets[].{name:BudgetName,limit:BudgetLimit.Amount,spent:CalculatedSpend.ActualSpend.Amount}' --output table
```

Also enable Cost Explorer (console, one click, takes 24 hours to populate) and turn on the cost-allocation tags `Project`, `Environment`, `Owner`, and `TTL` once resources with those tags exist:

```bash
aws ce update-cost-allocation-tags-status \
  --cost-allocation-tags-status TagKey=Project,Status=Active TagKey=Environment,Status=Active TagKey=Owner,Status=Active TagKey=TTL,Status=Active
```

---

## 6. Supporting Tools

| Tool | Purpose | Install (Linux) | Verify |
|---|---|---|---|
| `jq` | JSON processing beyond `--query` | `sudo apt-get install -y jq` | `jq --version` |
| Session Manager plugin | `aws ssm start-session` into EC2 without SSH or bastions | see below | `session-manager-plugin --version` |
| `eksctl` | Opinionated EKS cluster creation; used in Lab 6 for speed, replaced by Terraform in Cluster 05 | see below | `eksctl version` |
| `kubectl` | Kubernetes client, version within one minor of the EKS control plane | see below | `kubectl version --client` |
| `helm` | Chart installs (Karpenter, ALB controller, External Secrets) | `curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 \| bash` | `helm version` |
| `aws-vault` (optional) | Keeps credentials in the OS keychain | `brew install aws-vault` | `aws-vault --version` |
| `granted` (optional) | Fast SSO profile switching with `assume` | https://granted.dev | `assume --version` |

```bash
# Session Manager plugin (Debian/Ubuntu)
curl -fsSL "https://s3.amazonaws.com/session-manager-downloads/plugin/latest/ubuntu_64bit/session-manager-plugin.deb" -o /tmp/smp.deb
sudo dpkg -i /tmp/smp.deb

# eksctl
ARCH=amd64; [ "$(uname -m)" = "aarch64" ] && ARCH=arm64
curl -fsSL "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_Linux_${ARCH}.tar.gz" | tar xz -C /tmp
sudo install -m 0755 /tmp/eksctl /usr/local/bin/eksctl

# kubectl (match EKS minor; 1.31 shown)
curl -fsSLO "https://dl.k8s.io/release/v1.31.4/bin/linux/${ARCH}/kubectl"
sudo install -m 0755 kubectl /usr/local/bin/kubectl && rm kubectl
```

---

## 7. Verification Checklist

Run every line; all should succeed with the stated result before starting the labs.

```bash
aws --version | grep -q 'aws-cli/2' && echo "CLI v2: ok"
aws sts get-caller-identity --query Arn --output text | grep -q 'assumed-role/AWSReservedSSO' && echo "SSO auth: ok"
[ "$(aws iam get-account-summary --query 'SummaryMap.AccountAccessKeysPresent')" = "0" ] && echo "Root keys: none"
[ "$(aws iam get-account-summary --query 'SummaryMap.AccountMFAEnabled')" = "1" ] && echo "Root MFA: on"
aws s3control get-public-access-block --account-id 123456789012 \
  --query 'PublicAccessBlockConfiguration.RestrictPublicBuckets' | grep -q true && echo "S3 BPA: on"
aws ec2 get-ebs-encryption-by-default --query EbsEncryptionByDefault | grep -q true && echo "EBS default encryption: on"
aws cloudtrail get-trail-status --name capstone-trail --query IsLogging | grep -q true && echo "CloudTrail: logging"
aws budgets describe-budgets --account-id 123456789012 --query 'length(Budgets)' | grep -qv '^0$' && echo "Budget: set"
aws iam list-users --query 'length(Users)'   # 0 is the goal: no IAM users at all
aws organizations describe-organization --query 'Organization.FeatureSet' --output text   # ALL
```

Expected output:

```text
CLI v2: ok
SSO auth: ok
Root keys: none
Root MFA: on
S3 BPA: on
EBS default encryption: on
CloudTrail: logging
Budget: set
0
ALL
```

---

## 8. Common Setup Problems

| Symptom | Cause | Fix |
|---|---|---|
| `Error loading SSO Token: Token for capstone does not exist` | Not logged in, or logged in with a different session name | `aws sso login --sso-session capstone` |
| `The SSO session associated with this profile has expired or is otherwise invalid` | Permission set session (default 1 h) expired | `aws sso login --sso-session capstone`; raise the permission set duration to 4-8 h for lab use |
| `Unable to locate credentials` | `AWS_PROFILE` unset and no `[default]` section | `export AWS_PROFILE=lab-admin` or pass `--profile` |
| `An error occurred (UnrecognizedClientException)` or `InvalidClientTokenId` | Stale env vars from `export-credentials` override the SSO profile | `unset AWS_ACCESS_KEY_ID AWS_SECRET_ACCESS_KEY AWS_SESSION_TOKEN` |
| Browser does not open on headless host | No display | `aws sso login --use-device-code` and open the URL on another device |
| `aws configure sso` writes a legacy profile with `sso_start_url` inside `[profile]` | Older CLI 2.8 or earlier | Upgrade; the `sso-session` block (2.9+) is required for token refresh |
| `AccessDeniedException` on `budgets:CreateBudget` from an SSO admin | Budgets API needs the management account or IAM access to billing enabled | In the management account root user, enable "Activate IAM Access" under Account settings |
| Windows: `'aws' is not recognized` | PATH not refreshed | Open a new terminal, or `refreshenv` in Chocolatey shells |

Next: [02-core-concepts.md](02-core-concepts.md).
