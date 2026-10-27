# Amazon Bedrock: Installation and Setup

| | |
|---|---|
| **Cluster** | 02 Managed AI Platforms |
| **Role** | Anchor |
| **Prerequisites** | An AWS account you can administer (or a sandbox account), IAM fundamentals, a terminal with Python 3.12+ |
| **Target version** | AWS CLI v2.17+, boto3 1.35+, Python 3.12+ |
| **Capstone phase** | Phase 6 (Months 21-24) |

> After this file you will have the AWS CLI and boto3 configured with short-lived credentials, model access enabled for the models this folder uses, a least-privilege IAM role for invoking them, and a verified first `Converse` call with a token-usage printout. Nothing here costs more than a few cents.

---

## 1. Install the AWS CLI v2

Bedrock commands live in three CLI namespaces: `aws bedrock` (control plane), `aws bedrock-runtime` (inference), `aws bedrock-agent` / `aws bedrock-agent-runtime` (Knowledge Bases and Agents), plus `aws bedrock-agentcore` and `aws bedrock-agentcore-control` for AgentCore. All ship with AWS CLI v2; keep it current because new model families and parameters appear monthly.

### Linux (x86_64)

```bash
curl -sSL "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o /tmp/awscliv2.zip
unzip -q /tmp/awscliv2.zip -d /tmp
sudo /tmp/aws/install --update
aws --version
```

Expected output:

```text
aws-cli/2.27.x Python/3.13.x Linux/6.x exe/x86_64
```

### macOS

```bash
brew install awscli
# or the official pkg:
curl -sSL "https://awscli.amazonaws.com/AWSCLIV2.pkg" -o /tmp/AWSCLIV2.pkg
sudo installer -pkg /tmp/AWSCLIV2.pkg -target /
```

### Windows

```powershell
winget install --id Amazon.AWSCLI --exact
# or: msiexec.exe /i https://awscli.amazonaws.com/AWSCLIV2.msi
aws --version
```

### Container (CI runners, dev containers)

```dockerfile
FROM python:3.12-slim
RUN apt-get update && apt-get install -y --no-install-recommends curl unzip ca-certificates \
 && curl -sSL "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o /tmp/awscliv2.zip \
 && unzip -q /tmp/awscliv2.zip -d /tmp && /tmp/aws/install && rm -rf /tmp/aws /tmp/awscliv2.zip \
 && apt-get purge -y unzip && rm -rf /var/lib/apt/lists/*
RUN useradd --create-home --uid 10001 app
USER app
WORKDIR /home/app
RUN pip install --no-cache-dir --user "boto3>=1.35" "botocore>=1.35"
```

---

## 2. Authenticate Without Static Keys

Never put `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` in a file for Bedrock work. Use one of these, in order of preference:

| Context | Mechanism | Command |
|---|---|---|
| Laptop, organisation uses IAM Identity Center | SSO session, 8-hour tokens | `aws configure sso` then `aws sso login --profile bedrock-dev` |
| Laptop, standalone account | Assume a role from a minimal-permission user with MFA | `aws sts assume-role --role-arn ... --serial-number ... --token-code ...` via a profile with `role_arn` + `mfa_serial` |
| GitHub Actions | OIDC federation to a role | `aws-actions/configure-aws-credentials@v4` with `role-to-assume` |
| EKS pod | IRSA or EKS Pod Identity | ServiceAccount annotation; no config in code |
| Lambda | Execution role | Nothing to configure |
| EC2 | Instance profile | Nothing to configure |

Minimal `~/.aws/config` for SSO:

```ini
[sso-session example]
sso_start_url = https://example.awsapps.com/start
sso_region = us-east-1
sso_registration_scopes = sso:account:access

[profile bedrock-dev]
sso_session = example
sso_account_id = 123456789012
sso_role_name = BedrockDeveloper
region = us-east-1
output = json
```

```bash
aws sso login --profile bedrock-dev
export AWS_PROFILE=bedrock-dev
aws sts get-caller-identity
```

Expected output:

```json
{
    "UserId": "AROAEXAMPLEID:vishnu",
    "Account": "123456789012",
    "Arn": "arn:aws:sts::123456789012:assumed-role/AWSReservedSSO_BedrockDeveloper_abc123/vishnu"
}
```

---

## 3. Choose a Region and Enable Model Access

Bedrock is regional. Model availability differs by region; `us-east-1`, `us-west-2`, `eu-central-1`, `eu-west-1`, `ap-northeast-1`, and `ap-southeast-2` have the broadest catalogues. Cross-region inference profiles (`us.`, `eu.`, `apac.`, `global.`) widen availability but your *source* region must support the profile.

### 3.1 List what exists

```bash
export AWS_REGION=us-east-1

aws bedrock list-foundation-models \
  --by-output-modality TEXT \
  --query 'modelSummaries[].{id:modelId,provider:providerName,streaming:responseStreamingSupported,inference:inferenceTypesSupported}' \
  --output table
```

Expected output (abridged; the exact list changes frequently):

```text
--------------------------------------------------------------------------------------------
|                                  ListFoundationModels                                    |
+----------------------------------------------+-----------+-----------+-------------------+
|  id                                          | provider  | streaming | inference         |
+----------------------------------------------+-----------+-----------+-------------------+
|  anthropic.claude-sonnet-5-5                 | Anthropic | True      | ['INFERENCE_PROFILE'] |
|  anthropic.claude-haiku-5-5                  | Anthropic | True      | ['INFERENCE_PROFILE'] |
|  anthropic.claude-opus-5-5                   | Anthropic | True      | ['INFERENCE_PROFILE'] |
|  anthropic.claude-sonnet-4-5-20250929-v1:0   | Anthropic | True      | ['INFERENCE_PROFILE'] |
|  amazon.nova-pro-v1:0                        | Amazon    | True      | ['ON_DEMAND', 'INFERENCE_PROFILE'] |
|  amazon.nova-lite-v1:0                       | Amazon    | True      | ['ON_DEMAND', 'INFERENCE_PROFILE'] |
|  amazon.titan-embed-text-v2:0                | Amazon    | False     | ['ON_DEMAND']     |
|  meta.llama3-3-70b-instruct-v1:0             | Meta      | True      | ['INFERENCE_PROFILE'] |
+----------------------------------------------+-----------+-----------+-------------------+
```

Two naming conventions coexist. Older models carry a dated, ARN-versioned suffix (`anthropic.claude-sonnet-4-5-20250929-v1:0`); newer Anthropic models use the bare first-party name with the `anthropic.` prefix (`anthropic.claude-sonnet-5-5`). **Model IDs change over time. Never hard-code one from a blog post; list them, then put the chosen ID in configuration.** When `inferenceTypesSupported` contains only `INFERENCE_PROFILE`, you cannot call the bare model ID on demand; you must use an inference profile ID.

### 3.2 List inference profiles

```bash
aws bedrock list-inference-profiles \
  --type-equals SYSTEM_DEFINED \
  --query 'inferenceProfileSummaries[?contains(inferenceProfileId, `anthropic`)].{id:inferenceProfileId,status:status,regions:join(`,`, models[].modelArn)}' \
  --output table
```

Expected output (abridged):

```text
|  us.anthropic.claude-sonnet-5-5   | ACTIVE | arn:aws:bedrock:us-east-1::foundation-model/anthropic.claude-sonnet-5-5,arn:aws:bedrock:us-west-2::foundation-model/anthropic.claude-sonnet-5-5,... |
|  us.anthropic.claude-haiku-5-5    | ACTIVE | ... |
|  global.anthropic.claude-sonnet-5-5 | ACTIVE | ... |
```

### 3.3 Enable access

Most models are now available instantly; some Anthropic models require a one-time use-case form the first time an account requests them. Model access is granted per **account and region**.

```bash
# Check current entitlement for one model
aws bedrock get-foundation-model-availability \
  --model-id anthropic.claude-sonnet-5-5 \
  --query '{agreement:agreementAvailability.status,entitlement:entitlementAvailability,region:regionAvailability}'
```

Expected output:

```json
{
    "agreement": "AVAILABLE",
    "entitlement": "AVAILABLE",
    "region": "AVAILABLE"
}
```

If `agreement` is `NOT_AVAILABLE`, open the console at **Bedrock > Model catalog**, select the model, and choose **Request access** (or use `aws bedrock create-foundation-model-agreement` where the account allows it). In an AWS Organizations setup, check for an SCP denying `aws-marketplace:Subscribe`; Bedrock model agreements are Marketplace subscriptions under the hood.

Models this folder uses:

| Purpose | ID used in examples | Notes |
|---|---|---|
| Primary chat / tools | `us.anthropic.claude-sonnet-5-5` | Cross-region profile; replace with what `list-inference-profiles` shows |
| Cheap routing / classification | `us.anthropic.claude-haiku-5-5` | |
| Amazon first-party | `amazon.nova-lite-v1:0`, `amazon.nova-pro-v1:0` | On-demand without a profile |
| Embeddings | `amazon.titan-embed-text-v2:0` | 1024/512/256 dims, used by Knowledge Bases |

---

## 4. Create a Least-Privilege IAM Role

Do not develop with `AdministratorAccess`. Create a role that can invoke exactly the profiles you listed, read model metadata, and nothing else. The lab files add scoped statements for Knowledge Bases, Guardrails, and Agents as they are introduced.

`bedrock-invoke-policy.json`:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ListModels",
      "Effect": "Allow",
      "Action": [
        "bedrock:ListFoundationModels",
        "bedrock:GetFoundationModel",
        "bedrock:ListInferenceProfiles",
        "bedrock:GetInferenceProfile",
        "bedrock:GetFoundationModelAvailability"
      ],
      "Resource": "*"
    },
    {
      "Sid": "InvokeViaProfile",
      "Effect": "Allow",
      "Action": [
        "bedrock:InvokeModel",
        "bedrock:InvokeModelWithResponseStream"
      ],
      "Resource": [
        "arn:aws:bedrock:us-east-1:123456789012:inference-profile/us.anthropic.claude-sonnet-5-5",
        "arn:aws:bedrock:us-east-1:123456789012:inference-profile/us.anthropic.claude-haiku-5-5",
        "arn:aws:bedrock:us-*::foundation-model/anthropic.claude-sonnet-5-5",
        "arn:aws:bedrock:us-*::foundation-model/anthropic.claude-haiku-5-5",
        "arn:aws:bedrock:us-east-1::foundation-model/amazon.nova-lite-v1:0",
        "arn:aws:bedrock:us-east-1::foundation-model/amazon.titan-embed-text-v2:0"
      ]
    }
  ]
}
```

Why both the `inference-profile` ARN **and** the `foundation-model` ARNs in every destination region: a cross-region profile authorises against the profile *and* against the underlying model in whichever region the request lands. Omit the second and you get `AccessDeniedException` only on the requests that were routed to the other region, which is a maddening intermittent failure.

```bash
aws iam create-policy \
  --policy-name BedrockInvokeScoped \
  --policy-document file://bedrock-invoke-policy.json

# Trust policy lets your SSO role (or your user) assume it
cat > trust.json <<'EOF'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "AWS": "arn:aws:iam::123456789012:root" },
    "Action": "sts:AssumeRole",
    "Condition": { "Bool": { "aws:MultiFactorAuthPresent": "true" } }
  }]
}
EOF

aws iam create-role --role-name BedrockDeveloper --assume-role-policy-document file://trust.json
aws iam attach-role-policy --role-name BedrockDeveloper \
  --policy-arn arn:aws:iam::123456789012:policy/BedrockInvokeScoped
```

Add a profile that assumes it:

```ini
[profile bedrock-lab]
source_profile = bedrock-dev
role_arn = arn:aws:iam::123456789012:role/BedrockDeveloper
region = us-east-1
```

---

## 5. Install boto3 and Verify

```bash
python3.12 -m venv .venv && source .venv/bin/activate
pip install --upgrade pip
pip install "boto3>=1.35" "botocore>=1.35"
python -c "import boto3; print(boto3.__version__)"
```

Optional, for later labs:

```bash
pip install "anthropic[bedrock]"   # Anthropic SDK with Bedrock clients
pip install strands-agents strands-agents-tools   # Cluster 03 anchor
pip install bedrock-agentcore bedrock-agentcore-starter-toolkit   # AgentCore CLI and SDK
```

### 5.1 First call with the CLI

```bash
export AWS_PROFILE=bedrock-lab
export MODEL_ID=us.anthropic.claude-sonnet-5-5

aws bedrock-runtime converse \
  --model-id "$MODEL_ID" \
  --messages '[{"role":"user","content":[{"text":"Reply with exactly: Bedrock is reachable."}]}]' \
  --inference-config '{"maxTokens":50,"temperature":0}' \
  --query '{text:output.message.content[0].text,usage:usage,stop:stopReason}'
```

Expected output:

```json
{
    "text": "Bedrock is reachable.",
    "usage": {
        "inputTokens": 17,
        "outputTokens": 7,
        "totalTokens": 24
    },
    "stop": "end_turn"
}
```

### 5.2 First call with boto3

`hello_bedrock.py`:

```python
import os
import boto3
from botocore.config import Config

REGION = os.environ.get("AWS_REGION", "us-east-1")
MODEL_ID = os.environ.get("MODEL_ID", "us.anthropic.claude-sonnet-5-5")

# Standard retry mode gives exponential backoff with jitter on ThrottlingException.
client = boto3.client(
    "bedrock-runtime",
    region_name=REGION,
    config=Config(retries={"max_attempts": 8, "mode": "standard"}, read_timeout=120),
)

response = client.converse(
    modelId=MODEL_ID,
    system=[{"text": "You are a terse SRE assistant."}],
    messages=[{"role": "user", "content": [{"text": "In one sentence, what is an error budget?"}]}],
    inferenceConfig={"maxTokens": 128, "temperature": 0.2},
)

print(response["output"]["message"]["content"][0]["text"])
print(response["usage"], response["stopReason"], response["metrics"])
```

```bash
python hello_bedrock.py
```

Expected output:

```text
An error budget is the amount of unreliability a service may accumulate over a window before it violates its SLO.
{'inputTokens': 31, 'outputTokens': 27, 'totalTokens': 58} end_turn {'latencyMs': 1184}
```

### 5.3 Verify streaming works end to end

```bash
aws bedrock-runtime converse-stream \
  --model-id "$MODEL_ID" \
  --messages '[{"role":"user","content":[{"text":"Count from 1 to 5, one number per line."}]}]' \
  --inference-config '{"maxTokens":40}' \
  /dev/stdout | head -c 600
```

You should see a sequence of `messageStart`, `contentBlockDelta`, `contentBlockStop`, `messageStop`, and `metadata` events. The CLI writes the raw event stream; applications use the SDK iterator shown in [02-core-concepts.md](02-core-concepts.md#5-streaming-with-conversestream).

---

## 6. Enable Model Invocation Logging (Recommended Day One)

Logging is off by default. Turn it on to CloudWatch Logs now so every lab is auditable; the production file covers S3 delivery and KMS.

```bash
aws logs create-log-group --log-group-name /aws/bedrock/model-invocations
aws logs put-retention-policy --log-group-name /aws/bedrock/model-invocations --retention-in-days 30

cat > logging-trust.json <<'EOF'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "Service": "bedrock.amazonaws.com" },
    "Action": "sts:AssumeRole",
    "Condition": {
      "StringEquals": { "aws:SourceAccount": "123456789012" },
      "ArnLike": { "aws:SourceArn": "arn:aws:bedrock:us-east-1:123456789012:*" }
    }
  }]
}
EOF
aws iam create-role --role-name BedrockInvocationLogging --assume-role-policy-document file://logging-trust.json
aws iam put-role-policy --role-name BedrockInvocationLogging --policy-name WriteLogs --policy-document '{
  "Version":"2012-10-17",
  "Statement":[{"Effect":"Allow","Action":["logs:CreateLogStream","logs:PutLogEvents"],
  "Resource":"arn:aws:logs:us-east-1:123456789012:log-group:/aws/bedrock/model-invocations:*"}]}'

aws bedrock put-model-invocation-logging-configuration --logging-config '{
  "cloudWatchConfig": {
    "logGroupName": "/aws/bedrock/model-invocations",
    "roleArn": "arn:aws:iam::123456789012:role/BedrockInvocationLogging"
  },
  "textDataDeliveryEnabled": true,
  "imageDataDeliveryEnabled": false,
  "embeddingDataDeliveryEnabled": false
}'

aws bedrock get-model-invocation-logging-configuration
```

Re-run `hello_bedrock.py`, then:

```bash
aws logs tail /aws/bedrock/model-invocations --since 5m --format short | head -n 3
```

Each entry is a JSON document with `modelId`, `input.inputBodyJson`, `output.outputBodyJson`, token counts, and the caller identity ARN.

---

## 7. Setup Checklist

- [ ] AWS CLI v2 >= 2.17 installed; `aws --version` works
- [ ] Credentials come from SSO, assumed role, OIDC, or an instance/pod identity; no static keys on disk
- [ ] `AWS_REGION` chosen and model access confirmed with `get-foundation-model-availability`
- [ ] `BedrockDeveloper` role with `BedrockInvokeScoped` policy created; profile assumes it
- [ ] `aws bedrock-runtime converse` returns `end_turn`
- [ ] `hello_bedrock.py` prints usage and latency
- [ ] Invocation logging enabled to CloudWatch with 30-day retention

Next: [02-core-concepts.md](02-core-concepts.md).
