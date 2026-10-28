# AWS CloudFormation: Installation and Setup

| | |
|---|---|
| **Cluster** | 05 Infrastructure as Code |
| **Role** | Bridge (anchor is [Terraform](../terraform/README.md)) |
| **Prerequisites** | An AWS sandbox account; IAM Identity Center (SSO) or another short-lived credential source; Python 3.12+ for cfn-lint; Node.js 20+ only if you will use CDK |
| **Target version** | AWS CLI v2, cfn-lint 1.x, cfn-guard 3.x, Rain 1.x, SAM CLI 1.x, AWS CDK v2 |
| **Capstone phase** | Phase 2 (Months 5-8) |

> CloudFormation is a managed service, so there is nothing to install for the engine itself. What you install is the client toolchain: the AWS CLI to call the API, cfn-lint and cfn-guard to catch errors and policy violations before AWS sees the template, and optionally Rain, SAM and CDK. After this file you will have short-lived credentials configured, a linted editor, and a bootstrap stack that provides an encrypted artefacts bucket and a least-privilege service role for the labs.

---

## 1. What You Need and Why

| Tool | Purpose | Required for |
|---|---|---|
| AWS CLI v2 | Every CloudFormation API call: stacks, change sets, drift, StackSets | All labs |
| cfn-lint | Validates templates against the resource specification and best-practice rules; far stricter than `validate-template` | All labs, CI |
| cfn-guard | Policy-as-code rules evaluated against templates (and used by Guard Hooks) | Lab 8, CI |
| Rain | Friendlier deploy output, `rain fmt`, `rain build` scaffolding | Optional |
| SAM CLI | Builds and deploys templates that use the `AWS::Serverless-2016-10-31` transform | Optional |
| AWS CDK v2 | Generates CloudFormation from TypeScript, Python, Java, Go or .NET | Lab 7b (optional) |
| `jq` | Reading JSON output in scripts | Recommended |

`aws cloudformation validate-template` only checks syntax and basic structure. It does not catch a misspelt property, a wrong type or an invalid enum value. That is cfn-lint's job, so make cfn-lint your first check, not the API.

---

## 2. Install the AWS CLI v2

### Linux (x86_64)

```bash
curl --silent --show-error --location "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" --output awscliv2.zip
unzip -q awscliv2.zip
sudo ./aws/install --update
aws --version
```

For ARM64 hosts replace `x86_64` with `aarch64`.

### macOS

```bash
curl --silent --show-error --location "https://awscli.amazonaws.com/AWSCLIV2.pkg" --output AWSCLIV2.pkg
sudo installer -pkg AWSCLIV2.pkg -target /
aws --version
```

### Windows (PowerShell)

```powershell
winget install --id Amazon.AWSCLI --exact
aws --version
```

Expected output (versions will differ):

```text
aws-cli/2.22.x Python/3.12.x Linux/6.x exe/x86_64
```

### Container

```bash
docker run --rm -it \
  --volume "$HOME/.aws:/root/.aws:ro" \
  --volume "$PWD:/work" --workdir /work \
  public.ecr.aws/aws-cli/aws-cli:latest cloudformation list-stacks
```

Mount `~/.aws` read-only. For CI, use OIDC instead (section 6).

---

## 3. Authenticate with Short-Lived Credentials

Do not create IAM user access keys for this work. Use IAM Identity Center (SSO) on a workstation and OIDC in CI.

```bash
aws configure sso
# SSO session name: lab
# SSO start URL: https://<YOUR_SSO_PORTAL>.awsapps.com/start
# SSO region: <REGION>
# Choose the sandbox account and a permission set, then:
# CLI default client Region: eu-west-1
# CLI profile name: cfn-lab

aws sso login --profile cfn-lab
export AWS_PROFILE=cfn-lab
export AWS_REGION=eu-west-1
aws sts get-caller-identity
```

Expected output:

```json
{
    "UserId": "AROAEXAMPLEID000:you@example.com",
    "Account": "123456789012",
    "Arn": "arn:aws:sts::123456789012:assumed-role/AWSReservedSSO_SandboxAdmin_0000000000000000/you@example.com"
}
```

On PowerShell set the variables with `$env:AWS_PROFILE = "cfn-lab"` and `$env:AWS_REGION = "eu-west-1"`.

CloudFormation is regional. A stack created in `eu-west-1` does not exist in `us-east-1`. Most "stack does not exist" errors on a new machine are a region mismatch.

---

## 4. Install the Template Toolchain

### cfn-lint

```bash
pipx install cfn-lint          # or: python3 -m pip install --user cfn-lint
cfn-lint --version
```

Expected: `cfn-lint 1.x.y`. cfn-lint v1 ships with the resource schemas for all regions; update it regularly (`pipx upgrade cfn-lint`) so new resource types and properties are recognised.

Optional extras:

```bash
pipx inject cfn-lint pydot     # enables --build-graph to render the dependency graph
```

### cfn-guard

```bash
# macOS / Linux with Homebrew
brew install cloudformation-guard

# Any Linux / macOS without Homebrew
curl --proto '=https' --tlsv1.2 --silent --show-error --fail \
  https://raw.githubusercontent.com/aws-cloudformation/cloudformation-guard/main/install-guard.sh | sh
export PATH="$HOME/.guard/bin:$PATH"

cfn-guard --version
```

On Windows, download the release archive from the GitHub releases page or run `cargo install cfn-guard` if you have a Rust toolchain.

> Piping a script from the internet into `sh` is a convenience shortcut. In a shared or production environment, download the release binary, verify its checksum, and pin the version.

### Rain (optional)

```bash
brew install rain
# or, with Go 1.22+:
go install github.com/aws-cloudformation/rain/cmd/rain@latest
rain --version
```

### AWS SAM CLI (optional)

```bash
brew install aws-sam-cli       # macOS / Linux with Homebrew
sam --version
```

Windows and other Linux options use the official installers listed in the SAM Developer Guide.

### AWS CDK v2 (optional, Lab 7b)

```bash
npm install --global aws-cdk
cdk --version
python3 -m pip install --user aws-cdk-lib constructs   # Python bindings for the lab
```

Pin the CDK CLI version per project in `package.json` or a lab README; CDK CLI and `aws-cdk-lib` versions must be compatible.

---

## 5. Editor Setup

Templates use YAML short-form tags (`!Ref`, `!Sub`, `!GetAtt`) that generic YAML linters reject. Configure VS Code once:

1. Install the **AWS Toolkit** and **CloudFormation Linter** (`kddejong.vscode-cfn-lint`) extensions. The linter runs cfn-lint on save and provides schema-based completion.
2. Add the custom tags to user or workspace settings:

```json
{
  "yaml.customTags": [
    "!And sequence", "!If sequence", "!Not sequence", "!Equals sequence", "!Or sequence",
    "!FindInMap sequence", "!Base64 scalar", "!Cidr sequence", "!Ref scalar",
    "!Sub scalar", "!Sub sequence", "!GetAtt scalar", "!GetAtt sequence",
    "!GetAZs scalar", "!ImportValue scalar", "!Select sequence", "!Split sequence",
    "!Join sequence", "!Condition scalar", "!Transform mapping"
  ],
  "files.associations": {
    "**/cloudformation/**/*.yaml": "yaml"
  }
}
```

If `markdownlint` or `yamllint` runs in pre-commit, exclude CloudFormation templates from `yamllint` or configure it to allow unknown tags.

---

## 6. CI Authentication with GitHub Actions OIDC (preview)

CI never stores AWS keys. The workflow exchanges its GitHub OIDC token for a role session:

```yaml
permissions:
  id-token: write
  contents: read

steps:
  - uses: actions/checkout@v4
  - uses: aws-actions/configure-aws-credentials@v4
    with:
      role-to-assume: arn:aws:iam::123456789012:role/gha-cfn-plan
      aws-region: eu-west-1
  - run: aws cloudformation describe-stacks --stack-name capstone-dev-incident-api
```

The GitHub OIDC identity provider is created by the Terraform `global/iam` layer (see [Terraform labs](../terraform/04-hands-on-labs.md), Lab 7). Lab 8 in [04-hands-on-labs.md](04-hands-on-labs.md) adds two CloudFormation-specific roles: `gha-cfn-plan` can only create change sets, and `gha-cfn-deploy` can only execute them from an approved environment.

---

## 7. First-Run Configuration: the Lab Bootstrap Stack

The labs need two things: an encrypted S3 bucket for packaged templates and Lambda code, and a **service role** that CloudFormation assumes instead of using your own permissions. Creating them with CloudFormation is also your first deployment.

Create `bootstrap/cfn-lab-bootstrap.yaml`:

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: Artefacts bucket and service role for CloudFormation labs

Resources:
  ArtefactsBucket:
    Type: AWS::S3::Bucket
    DeletionPolicy: Delete
    UpdateReplacePolicy: Retain
    Properties:
      BucketEncryption:
        ServerSideEncryptionConfiguration:
          - ServerSideEncryptionByDefault:
              SSEAlgorithm: aws:kms
            BucketKeyEnabled: true
      PublicAccessBlockConfiguration:
        BlockPublicAcls: true
        BlockPublicPolicy: true
        IgnorePublicAcls: true
        RestrictPublicBuckets: true
      OwnershipControls:
        Rules:
          - ObjectOwnership: BucketOwnerEnforced
      VersioningConfiguration:
        Status: Enabled
      LifecycleConfiguration:
        Rules:
          - Id: expire-artefacts
            Status: Enabled
            ExpirationInDays: 30
            NoncurrentVersionExpiration:
              NoncurrentDays: 7

  ArtefactsBucketPolicy:
    Type: AWS::S3::BucketPolicy
    Properties:
      Bucket: !Ref ArtefactsBucket
      PolicyDocument:
        Version: "2012-10-17"
        Statement:
          - Sid: DenyInsecureTransport
            Effect: Deny
            Principal: "*"
            Action: "s3:*"
            Resource:
              - !GetAtt ArtefactsBucket.Arn
              - !Sub "${ArtefactsBucket.Arn}/*"
            Condition:
              Bool:
                aws:SecureTransport: "false"

  LabServiceRole:
    Type: AWS::IAM::Role
    Properties:
      RoleName: cfn-lab-service-role
      AssumeRolePolicyDocument:
        Version: "2012-10-17"
        Statement:
          - Effect: Allow
            Principal:
              Service: cloudformation.amazonaws.com
            Action: sts:AssumeRole
            Condition:
              StringEquals:
                aws:SourceAccount: !Ref AWS::AccountId
      ManagedPolicyArns:
        - !Sub "arn:${AWS::Partition}:iam::aws:policy/PowerUserAccess"
      Policies:
        - PolicyName: lab-scoped-iam
          PolicyDocument:
            Version: "2012-10-17"
            Statement:
              - Sid: ManageLabRolesOnly
                Effect: Allow
                Action:
                  - iam:CreateRole
                  - iam:DeleteRole
                  - iam:GetRole
                  - iam:PassRole
                  - iam:TagRole
                  - iam:UntagRole
                  - iam:UpdateAssumeRolePolicy
                  - iam:PutRolePolicy
                  - iam:GetRolePolicy
                  - iam:DeleteRolePolicy
                  - iam:AttachRolePolicy
                  - iam:DetachRolePolicy
                  - iam:ListRolePolicies
                  - iam:ListAttachedRolePolicies
                Resource: !Sub "arn:${AWS::Partition}:iam::${AWS::AccountId}:role/cfn-lab/*"

Outputs:
  ArtefactsBucketName:
    Value: !Ref ArtefactsBucket
  ServiceRoleArn:
    Value: !GetAtt LabServiceRole.Arn
```

> **Lab shortcut.** `PowerUserAccess` is broad. It is acceptable in a disposable sandbox because IAM changes are restricted to roles under the `/cfn-lab/` path, so the service role cannot escalate its own privileges. In production, give each pipeline a service role scoped to the resource types its stacks contain (see [05-production-patterns.md](05-production-patterns.md)).

Lint, deploy and capture the outputs:

```bash
cfn-lint bootstrap/cfn-lab-bootstrap.yaml

aws cloudformation deploy \
  --stack-name cfn-lab-bootstrap \
  --template-file bootstrap/cfn-lab-bootstrap.yaml \
  --capabilities CAPABILITY_NAMED_IAM \
  --tags project=cfn-lab owner=<YOUR_ALIAS>

export CFN_BUCKET=$(aws cloudformation describe-stacks --stack-name cfn-lab-bootstrap \
  --query "Stacks[0].Outputs[?OutputKey=='ArtefactsBucketName'].OutputValue" --output text)
export CFN_ROLE=$(aws cloudformation describe-stacks --stack-name cfn-lab-bootstrap \
  --query "Stacks[0].Outputs[?OutputKey=='ServiceRoleArn'].OutputValue" --output text)
echo "$CFN_BUCKET $CFN_ROLE"
```

Expected output:

```text
Waiting for changeset to be created..
Waiting for stack create/update to complete
Successfully created/updated stack - cfn-lab-bootstrap
cfn-lab-bootstrap-artefactsbucket-a1b2c3d4e5f6 arn:aws:iam::123456789012:role/cfn-lab-service-role
```

`CAPABILITY_NAMED_IAM` is required because the template creates an IAM role with an explicit `RoleName`. Without it the call fails with `InsufficientCapabilitiesException`. This is CloudFormation making you acknowledge IAM changes explicitly.

This stack runs without a service role, using your SSO permissions, because the role does not exist yet. Every later lab passes `--role-arn "$CFN_ROLE"`.

---

## 8. Verification Checklist

```bash
aws --version
cfn-lint --version
cfn-guard --version
aws sts get-caller-identity --query Account --output text
aws configure get region                                   # eu-west-1
aws cloudformation describe-stacks --stack-name cfn-lab-bootstrap \
  --query "Stacks[0].StackStatus" --output text            # CREATE_COMPLETE
aws cloudformation validate-template \
  --template-body file://bootstrap/cfn-lab-bootstrap.yaml \
  --query Capabilities                                     # ["CAPABILITY_NAMED_IAM"]
```

Notice that `validate-template` reports the capabilities a template needs. That is useful in CI to fail fast with a clear message.

---

## 9. Common Setup Problems

| Symptom | Cause | Fix |
|---|---|---|
| `Unable to locate credentials` | No profile exported, or SSO session expired | `aws sso login --profile cfn-lab`; `export AWS_PROFILE=cfn-lab` |
| `You must specify a region` | No default region in the profile | `export AWS_REGION=eu-west-1` or set it in `~/.aws/config` |
| `Stack with id cfn-lab-bootstrap does not exist` | Different region or account than the one you deployed to | Check `aws configure get region` and `aws sts get-caller-identity` |
| cfn-lint `E3002 Additional properties are not allowed` on a property that exists | cfn-lint is older than the property | `pipx upgrade cfn-lint` |
| VS Code flags `!Ref` as `Unresolved tag` | `yaml.customTags` not configured | Add the settings block from section 5 |
| `cfn-guard: command not found` after the install script | `~/.guard/bin` not on `PATH` | Add the export to your shell profile |

Next: [02-core-concepts.md](02-core-concepts.md).
