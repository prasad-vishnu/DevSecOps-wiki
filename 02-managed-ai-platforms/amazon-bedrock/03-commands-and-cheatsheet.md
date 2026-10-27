# Amazon Bedrock: Commands and Cheatsheet

| | |
|---|---|
| **Cluster** | 02 Managed AI Platforms |
| **Role** | Anchor |
| **Prerequisites** | [01-installation-and-setup.md](01-installation-and-setup.md); `AWS_PROFILE` and `AWS_REGION` exported |
| **Target version** | AWS CLI v2.17+, boto3 1.35+ |
| **Capstone phase** | Phase 6 (Months 21-24) |

> A task-organised reference for every `aws bedrock*` CLI command and boto3 call used in this folder. Each entry has a one-line explanation. Long-form flags are shown; where the CLI accepts a short form it is noted. Placeholders: `<MODEL_ID>`, `<PROFILE_ID>`, `<KB_ID>`, `<DS_ID>`, `<GUARDRAIL_ID>`, `<AGENT_ID>`, `<ALIAS_ID>`, `<BUCKET_NAME>`, `123456789012` (account), `us-east-1` (region).

Set once per shell:

```bash
export AWS_REGION=us-east-1
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export MODEL_ID=us.anthropic.claude-sonnet-5-5
export CHEAP_MODEL_ID=us.anthropic.claude-haiku-5-5
```

---

## 1. Discover Models and Access

| Command | Explanation |
|---|---|
| `aws bedrock list-foundation-models` | All models in the region with provider, modalities, streaming and inference-type support |
| `aws bedrock list-foundation-models --by-provider Anthropic --by-output-modality TEXT` | Filter by provider and modality |
| `aws bedrock get-foundation-model --model-identifier <MODEL_ID>` | Details for one model including `modelArn` and `inferenceTypesSupported` |
| `aws bedrock get-foundation-model-availability --model-id <MODEL_ID>` | Whether your account has agreement + entitlement in this region |
| `aws bedrock list-inference-profiles --type-equals SYSTEM_DEFINED` | Cross-region profile IDs (`us.`, `eu.`, `apac.`, `global.`) and their destination model ARNs |
| `aws bedrock list-inference-profiles --type-equals APPLICATION` | Your own tagged profiles |
| `aws bedrock get-inference-profile --inference-profile-identifier <PROFILE_ID>` | Destination regions for a profile |
| `aws bedrock create-inference-profile --inference-profile-name <NAME> --model-source copyFrom=<PROFILE_ARN> --tags key=CostCenter,value=<CC>` | Create an application profile for cost allocation and IAM scoping |
| `aws bedrock delete-inference-profile --inference-profile-identifier <ARN>` | Remove an application profile |
| `aws bedrock list-foundation-model-agreement-offers --model-id <MODEL_ID>` | Show Marketplace offer for a gated model |
| `aws bedrock create-foundation-model-agreement --model-id <MODEL_ID> --offer-token <TOKEN>` | Accept the agreement (where permitted by SCPs) |

Useful JMESPath one-liners:

```bash
# IDs of Anthropic models that support on-demand (no profile needed)
aws bedrock list-foundation-models --by-provider Anthropic \
  --query 'modelSummaries[?contains(inferenceTypesSupported, `ON_DEMAND`)].modelId'

# Every system profile and its destination regions, flattened
aws bedrock list-inference-profiles --type-equals SYSTEM_DEFINED \
  --query 'inferenceProfileSummaries[].[inferenceProfileId, join(`,`, models[].modelArn)]' --output text \
  | sed -E 's/arn:aws:bedrock:([a-z0-9-]+)::foundation-model\/[^,]*/\1/g'
```

---

## 2. Inference: `aws bedrock-runtime`

| Command | Explanation |
|---|---|
| `aws bedrock-runtime converse --model-id <ID> --messages '<JSON>'` | Single vendor-neutral chat turn |
| `aws bedrock-runtime converse --model-id <ID> --messages file://messages.json --system '[{"text":"..."}]' --inference-config '{"maxTokens":512,"temperature":0}'` | With system prompt and inference config from files |
| `aws bedrock-runtime converse ... --tool-config file://tools.json` | Include tool definitions |
| `aws bedrock-runtime converse ... --guardrail-config '{"guardrailIdentifier":"<GUARDRAIL_ID>","guardrailVersion":"1","trace":"enabled"}'` | Apply a guardrail |
| `aws bedrock-runtime converse ... --additional-model-request-fields '{"thinking":{"type":"adaptive"}}'` | Provider-specific passthrough fields |
| `aws bedrock-runtime converse-stream --model-id <ID> --messages '<JSON>' out.bin` | Streaming; the CLI writes raw EventStream to the file argument |
| `aws bedrock-runtime invoke-model --model-id <ID> --content-type application/json --accept application/json --body file://body.json out.json` | Native-schema request (embeddings, images, provider betas) |
| `aws bedrock-runtime invoke-model --model-id amazon.titan-embed-text-v2:0 --body '{"inputText":"hello","dimensions":512,"normalize":true}' --cli-binary-format raw-in-base64-out out.json` | Titan embedding; `--cli-binary-format` needed when passing inline JSON |
| `aws bedrock-runtime invoke-model-with-response-stream --model-id <ID> --body file://body.json out.bin` | Native streaming |
| `aws bedrock-runtime apply-guardrail --guardrail-identifier <GUARDRAIL_ID> --guardrail-version 1 --source OUTPUT --content '[{"text":{"text":"..."}}]'` | Evaluate arbitrary text against a guardrail |
| `aws bedrock-runtime count-tokens --model-id <MODEL_ID> --input '{"converse":{"messages":[...]}}'` | Token count without invoking (supported models) |
| `aws bedrock-runtime get-async-invoke --invocation-arn <ARN>` / `list-async-invokes` | Status of asynchronous (video/long) invocations |

Converse request file skeleton (`messages.json`):

```json
[
  {"role": "user", "content": [{"text": "Summarise the attached runbook in three bullets."},
                                {"document": {"format": "md", "name": "runbook",
                                              "source": {"bytes": "<BASE64_CONTENT>"}}}]}
]
```

For the CLI, `bytes` must be base64 text; for boto3 pass raw `bytes`.

---

## 3. Inference: boto3 Patterns

```python
import boto3, json
from botocore.config import Config

rt = boto3.client("bedrock-runtime", region_name="us-east-1",
                  config=Config(retries={"max_attempts": 8, "mode": "adaptive"},
                                read_timeout=300, connect_timeout=10))
```

| Call | Explanation |
|---|---|
| `rt.converse(modelId=, messages=, system=, inferenceConfig=, toolConfig=, guardrailConfig=, additionalModelRequestFields=, requestMetadata=)` | One turn; returns `output.message`, `stopReason`, `usage`, `metrics` |
| `rt.converse_stream(**same)` | Returns `resp["stream"]`, an iterator of event dicts |
| `rt.invoke_model(modelId=, body=json.dumps(...), contentType="application/json", accept="application/json")` | Native body; read with `json.loads(resp["body"].read())` |
| `rt.invoke_model_with_response_stream(...)` | `for ev in resp["body"]: json.loads(ev["chunk"]["bytes"])` |
| `rt.apply_guardrail(guardrailIdentifier=, guardrailVersion=, source="INPUT" or "OUTPUT", content=[...])` | Returns `action`, `outputs`, `assessments` |
| `rt.count_tokens(modelId=, input={"converse": {...}})` | Pre-flight token count |

Streaming text helper:

```python
def stream(messages, **kw):
    for ev in rt.converse_stream(modelId=MODEL_ID, messages=messages, **kw)["stream"]:
        if "contentBlockDelta" in ev and "text" in ev["contentBlockDelta"]["delta"]:
            yield ev["contentBlockDelta"]["delta"]["text"]
```

Streaming tool-call assembler:

```python
def stream_with_tools(messages, tool_config):
    text, tools, buf = [], {}, {}
    for ev in rt.converse_stream(modelId=MODEL_ID, messages=messages, toolConfig=tool_config)["stream"]:
        if "contentBlockStart" in ev and "toolUse" in ev["contentBlockStart"]["start"]:
            idx = ev["contentBlockStart"]["contentBlockIndex"]
            tools[idx] = ev["contentBlockStart"]["start"]["toolUse"]; buf[idx] = ""
        elif "contentBlockDelta" in ev:
            idx, d = ev["contentBlockDelta"]["contentBlockIndex"], ev["contentBlockDelta"]["delta"]
            if "text" in d: text.append(d["text"])
            if "toolUse" in d: buf[idx] += d["toolUse"]["input"]
        elif "contentBlockStop" in ev:
            idx = ev["contentBlockStop"]["contentBlockIndex"]
            if idx in tools: tools[idx]["input"] = json.loads(buf[idx] or "{}")
        elif "messageStop" in ev:
            stop = ev["messageStop"]["stopReason"]
    return "".join(text), list(tools.values()), stop
```

Titan embeddings:

```python
def embed(text: str, dims: int = 1024) -> list[float]:
    body = {"inputText": text, "dimensions": dims, "normalize": True}
    r = rt.invoke_model(modelId="amazon.titan-embed-text-v2:0", body=json.dumps(body),
                        contentType="application/json", accept="application/json")
    return json.loads(r["body"].read())["embedding"]
```

Retry with explicit backoff when you need control beyond botocore:

```python
import random, time
from botocore.exceptions import ClientError

RETRYABLE = {"ThrottlingException", "ServiceUnavailableException", "ModelNotReadyException",
             "ModelTimeoutException", "InternalServerException"}

def converse_with_backoff(max_attempts=8, base=0.5, cap=20.0, **kwargs):
    for attempt in range(max_attempts):
        try:
            return rt.converse(**kwargs)
        except ClientError as e:
            code = e.response["Error"]["Code"]
            if code not in RETRYABLE or attempt == max_attempts - 1:
                raise
            sleep = min(cap, base * 2 ** attempt) * random.uniform(0.5, 1.5)   # full jitter
            time.sleep(sleep)
```

---

## 4. Provisioned Throughput and Batch

| Command | Explanation |
|---|---|
| `aws bedrock create-provisioned-model-throughput --provisioned-model-name <NAME> --model-id <MODEL_ID> --model-units 1 [--commitment-duration OneMonth\|SixMonths]` | Buy reserved capacity; omit commitment for hourly no-commit where offered |
| `aws bedrock list-provisioned-model-throughputs` | Status (`Creating`, `InService`, `Failed`) and ARNs |
| `aws bedrock get-provisioned-model-throughput --provisioned-model-id <ARN>` | Details and desired vs current MUs |
| `aws bedrock update-provisioned-model-throughput --provisioned-model-id <ARN> --desired-model-units 2` | Resize |
| `aws bedrock delete-provisioned-model-throughput --provisioned-model-id <ARN>` | **Stop the hourly bill** |
| `aws bedrock create-model-invocation-job --job-name <NAME> --model-id <ID> --role-arn <ROLE_ARN> --input-data-config '{"s3InputDataConfig":{"s3Uri":"s3://<BUCKET_NAME>/in/"}}' --output-data-config '{"s3OutputDataConfig":{"s3Uri":"s3://<BUCKET_NAME>/out/"}}'` | Start batch inference over JSONL |
| `aws bedrock list-model-invocation-jobs --status-equals InProgress` | Monitor batch jobs |
| `aws bedrock get-model-invocation-job --job-identifier <JOB_ARN>` | Job detail including `message` on failure |
| `aws bedrock stop-model-invocation-job --job-identifier <JOB_ARN>` | Cancel |

Batch input line format (Converse-style body under `modelInput`):

```json
{"recordId": "alert-0001", "modelInput": {"messages": [{"role": "user", "content": [{"text": "Classify severity: ..."}]}], "inferenceConfig": {"maxTokens": 20}}}
```

---

## 5. Guardrails: `aws bedrock`

| Command | Explanation |
|---|---|
| `aws bedrock create-guardrail --name <NAME> --blocked-input-messaging "..." --blocked-outputs-messaging "..." --topic-policy-config file://topics.json --content-policy-config file://content.json --sensitive-information-policy-config file://pii.json --word-policy-config file://words.json --contextual-grounding-policy-config file://grounding.json` | Create a guardrail (returns `guardrailId`, version `DRAFT`) |
| `aws bedrock get-guardrail --guardrail-identifier <GUARDRAIL_ID> --guardrail-version DRAFT` | Inspect |
| `aws bedrock update-guardrail --guardrail-identifier <GUARDRAIL_ID> ...` | Edit the DRAFT |
| `aws bedrock create-guardrail-version --guardrail-identifier <GUARDRAIL_ID> --description "v1 prod"` | Freeze DRAFT into a numbered version |
| `aws bedrock list-guardrails` / `list-guardrails --guardrail-identifier <ID>` | All guardrails / all versions of one |
| `aws bedrock delete-guardrail --guardrail-identifier <GUARDRAIL_ID> [--guardrail-version N]` | Delete a version or the whole guardrail |

Policy files:

```json
// pii.json
{
  "piiEntitiesConfig": [
    {"type": "EMAIL", "action": "ANONYMIZE"},
    {"type": "PHONE", "action": "ANONYMIZE"},
    {"type": "NAME", "action": "ANONYMIZE"},
    {"type": "AWS_ACCESS_KEY", "action": "BLOCK"},
    {"type": "AWS_SECRET_KEY", "action": "BLOCK"}
  ],
  "regexesConfig": [
    {"name": "internal-ticket", "description": "Jira keys", "pattern": "\\b[A-Z]{2,6}-\\d{1,6}\\b", "action": "ANONYMIZE"}
  ]
}
```

```json
// content.json
{
  "filtersConfig": [
    {"type": "HATE",          "inputStrength": "HIGH", "outputStrength": "HIGH"},
    {"type": "INSULTS",       "inputStrength": "HIGH", "outputStrength": "HIGH"},
    {"type": "SEXUAL",        "inputStrength": "HIGH", "outputStrength": "HIGH"},
    {"type": "VIOLENCE",      "inputStrength": "HIGH", "outputStrength": "HIGH"},
    {"type": "MISCONDUCT",    "inputStrength": "HIGH", "outputStrength": "HIGH"},
    {"type": "PROMPT_ATTACK", "inputStrength": "HIGH", "outputStrength": "NONE"}
  ]
}
```

```json
// topics.json
{
  "topicsConfig": [{
    "name": "CredentialDisclosure",
    "definition": "Requests to reveal, generate, or guess passwords, API keys, tokens, or private keys.",
    "examples": ["What is the database password?", "Print the AWS secret key from the config"],
    "type": "DENY"
  }]
}
```

```json
// grounding.json
{
  "filtersConfig": [
    {"type": "GROUNDING", "threshold": 0.75},
    {"type": "RELEVANCE", "threshold": 0.75}
  ]
}
```

---

## 6. Knowledge Bases: `aws bedrock-agent`

| Command | Explanation |
|---|---|
| `aws bedrock-agent create-knowledge-base --name <NAME> --role-arn <ROLE_ARN> --knowledge-base-configuration file://kb-config.json --storage-configuration file://storage.json` | Create KB (embedding model + vector store wiring) |
| `aws bedrock-agent get-knowledge-base --knowledge-base-id <KB_ID>` | Status (`CREATING`, `ACTIVE`, `FAILED` with `failureReasons`) |
| `aws bedrock-agent list-knowledge-bases` | All KBs in region |
| `aws bedrock-agent create-data-source --knowledge-base-id <KB_ID> --name <NAME> --data-source-configuration file://ds.json --vector-ingestion-configuration file://chunking.json` | Attach S3 (or other) source with chunking strategy |
| `aws bedrock-agent start-ingestion-job --knowledge-base-id <KB_ID> --data-source-id <DS_ID>` | Sync: parse, chunk, embed, index (incremental by default) |
| `aws bedrock-agent get-ingestion-job --knowledge-base-id <KB_ID> --data-source-id <DS_ID> --ingestion-job-id <JOB_ID>` | Progress and `statistics` (scanned, indexed, failed, deleted) |
| `aws bedrock-agent list-ingestion-jobs --knowledge-base-id <KB_ID> --data-source-id <DS_ID>` | History |
| `aws bedrock-agent list-knowledge-base-documents --knowledge-base-id <KB_ID> --data-source-id <DS_ID>` | Per-document index status |
| `aws bedrock-agent ingest-knowledge-base-documents --knowledge-base-id <KB_ID> --data-source-id <DS_ID> --documents file://docs.json` | Push documents directly (custom data source) |
| `aws bedrock-agent delete-data-source --knowledge-base-id <KB_ID> --data-source-id <DS_ID>` | Remove source (and by default its vectors) |
| `aws bedrock-agent delete-knowledge-base --knowledge-base-id <KB_ID>` | Delete KB (**does not delete the vector store**) |

Config file skeletons:

```json
// kb-config.json
{
  "type": "VECTOR",
  "vectorKnowledgeBaseConfiguration": {
    "embeddingModelArn": "arn:aws:bedrock:us-east-1::foundation-model/amazon.titan-embed-text-v2:0",
    "embeddingModelConfiguration": {"bedrockEmbeddingModelConfiguration": {"dimensions": 1024, "embeddingDataType": "FLOAT32"}}
  }
}
```

```json
// storage.json (OpenSearch Serverless)
{
  "type": "OPENSEARCH_SERVERLESS",
  "opensearchServerlessConfiguration": {
    "collectionArn": "arn:aws:aoss:us-east-1:123456789012:collection/<COLLECTION_ID>",
    "vectorIndexName": "runbooks-index",
    "fieldMapping": {"vectorField": "embedding", "textField": "text", "metadataField": "metadata"}
  }
}
```

```json
// storage.json (Aurora PostgreSQL pgvector)
{
  "type": "RDS",
  "rdsConfiguration": {
    "resourceArn": "arn:aws:rds:us-east-1:123456789012:cluster:<CLUSTER_NAME>",
    "credentialsSecretArn": "arn:aws:secretsmanager:us-east-1:123456789012:secret:<SECRET_NAME>",
    "databaseName": "kb",
    "tableName": "bedrock_integration.bedrock_kb",
    "fieldMapping": {"primaryKeyField": "id", "vectorField": "embedding", "textField": "chunks", "metadataField": "metadata"}
  }
}
```

```json
// chunking.json (hierarchical)
{
  "chunkingConfiguration": {
    "chunkingStrategy": "HIERARCHICAL",
    "hierarchicalChunkingConfiguration": {
      "levelConfigurations": [{"maxTokens": 1500}, {"maxTokens": 300}],
      "overlapTokens": 60
    }
  }
}
```

```json
// ds.json
{
  "type": "S3",
  "s3Configuration": {"bucketArn": "arn:aws:s3:::<BUCKET_NAME>", "inclusionPrefixes": ["runbooks/"]}
}
```

### Retrieval: `aws bedrock-agent-runtime`

| Command | Explanation |
|---|---|
| `aws bedrock-agent-runtime retrieve --knowledge-base-id <KB_ID> --retrieval-query '{"text":"..."}' --retrieval-configuration '{"vectorSearchConfiguration":{"numberOfResults":5,"overrideSearchType":"HYBRID"}}'` | Chunks + scores + locations |
| `aws bedrock-agent-runtime retrieve-and-generate --input '{"text":"..."}' --retrieve-and-generate-configuration file://rag.json` | Answer with citations; add `--session-id` to continue a conversation |
| `aws bedrock-agent-runtime retrieve-and-generate-stream ...` | Streaming variant |
| `aws bedrock-agent-runtime rerank --queries '[{"type":"TEXT","textQuery":{"text":"..."}}]' --sources file://sources.json --reranking-configuration file://rerank.json` | Rerank your own candidate passages |

---

## 7. Agents: `aws bedrock-agent` and `aws bedrock-agent-runtime`

| Command | Explanation |
|---|---|
| `aws bedrock-agent create-agent --agent-name <NAME> --agent-resource-role-arn <ROLE_ARN> --foundation-model <PROFILE_ID> --instruction "You are ..." --idle-session-ttl-in-seconds 600` | Create agent (DRAFT) |
| `aws bedrock-agent create-agent-action-group --agent-id <AGENT_ID> --agent-version DRAFT --action-group-name <NAME> --action-group-executor lambda=<LAMBDA_ARN> --function-schema file://functions.json` | Tools via function schema + Lambda |
| `aws bedrock-agent create-agent-action-group ... --api-schema s3={s3BucketName=<BUCKET_NAME>,s3ObjectKey=openapi.yaml}` | Tools via OpenAPI schema |
| `aws bedrock-agent create-agent-action-group ... --action-group-executor customControl=RETURN_CONTROL` | Your app executes tools; agent returns the call to you |
| `aws bedrock-agent associate-agent-knowledge-base --agent-id <AGENT_ID> --agent-version DRAFT --knowledge-base-id <KB_ID> --description "Runbooks"` | Attach KB |
| `aws bedrock-agent prepare-agent --agent-id <AGENT_ID>` | Compile DRAFT; required after any change |
| `aws bedrock-agent create-agent-alias --agent-id <AGENT_ID> --agent-alias-name prod` | Alias pointing at a version (creates a version from DRAFT) |
| `aws bedrock-agent get-agent --agent-id <AGENT_ID>` | Status (`PREPARED`, `NOT_PREPARED`, `FAILED`) |
| `aws bedrock-agent-runtime invoke-agent --agent-id <AGENT_ID> --agent-alias-id <ALIAS_ID> --session-id <UUID> --input-text "..." --enable-trace out.bin` | Invoke; output is EventStream of `chunk` and `trace` events |
| `aws bedrock-agent delete-agent --agent-id <AGENT_ID> --skip-resource-in-use-check` | Delete |

Function schema file:

```json
{
  "functions": [{
    "name": "getPodStatus",
    "description": "Return phase, restart count and last event for a Kubernetes pod.",
    "parameters": {
      "namespace": {"type": "string", "description": "Kubernetes namespace", "required": true},
      "pod": {"type": "string", "description": "Pod name", "required": true}
    },
    "requireConfirmation": "DISABLED"
  }]
}
```

boto3 `invoke_agent` consumption:

```python
agent_rt = boto3.client("bedrock-agent-runtime", region_name="us-east-1")
resp = agent_rt.invoke_agent(agentId="<AGENT_ID>", agentAliasId="<ALIAS_ID>",
                             sessionId=str(uuid.uuid4()), inputText="Why is payments-api crashlooping?",
                             enableTrace=True)
for ev in resp["completion"]:
    if "chunk" in ev:
        print(ev["chunk"]["bytes"].decode(), end="")
    elif "trace" in ev:
        t = ev["trace"]["trace"]
        step = next(iter(t))          # orchestrationTrace, preProcessingTrace, ...
        print(f"\n[trace:{step}]", json.dumps(t[step])[:200])
```

---

## 8. AgentCore: starter toolkit and CLI

| Command | Explanation |
|---|---|
| `pip install bedrock-agentcore bedrock-agentcore-starter-toolkit` | SDK + `agentcore` CLI |
| `agentcore configure --entrypoint agent.py --name sre-agent` | Generate Dockerfile, runtime config, IAM role, ECR repo settings |
| `agentcore launch` | Build (CodeBuild, arm64), push to ECR, create/update AgentCore Runtime |
| `agentcore launch --local` | Run the same container locally on port 8080 |
| `agentcore invoke '{"prompt": "hello"}'` | Invoke the deployed runtime |
| `agentcore status` | Runtime ARN, endpoint, last deployment |
| `agentcore destroy` | Delete runtime, ECR images, and generated roles |
| `aws bedrock-agentcore-control list-agent-runtimes` | Control-plane listing |
| `aws bedrock-agentcore-control create-gateway --name <NAME> --protocol-type MCP --authorizer-type CUSTOM_JWT --authorizer-configuration file://jwt.json --role-arn <ROLE_ARN>` | Create an MCP gateway |
| `aws bedrock-agentcore-control create-gateway-target --gateway-identifier <GW_ID> --name k8s --target-configuration file://target.json --credential-provider-configurations file://creds.json` | Expose a Lambda/OpenAPI as MCP tools |
| `aws bedrock-agentcore-control create-memory --name <NAME> --event-expiry-duration 30 --memory-strategies file://strategies.json` | Create memory store with long-term strategies |
| `aws bedrock-agentcore invoke-agent-runtime --agent-runtime-arn <ARN> --runtime-session-id <UUID> --payload '{"prompt":"..."}' out.json` | Data-plane invoke |
| `aws bedrock-agentcore create-event --memory-id <MEM_ID> --actor-id <USER> --session-id <SID> --event-timestamp $(date +%s) --payload file://event.json` | Write a memory event |
| `aws bedrock-agentcore retrieve-memory-records --memory-id <MEM_ID> --namespace "/users/<USER>" --search-criteria '{"searchQuery":"preferences"}'` | Semantic recall from long-term memory |

Minimal AgentCore entrypoint:

```python
from bedrock_agentcore import BedrockAgentCoreApp
from strands import Agent

app = BedrockAgentCoreApp()
agent = Agent(model="us.anthropic.claude-sonnet-5-5")   # Strands uses Converse under the hood

@app.entrypoint
def invoke(payload, context):
    return {"result": agent(payload.get("prompt", "")).message}

if __name__ == "__main__":
    app.run()   # serves POST /invocations and GET /ping on :8080
```

---

## 9. Logging, Metrics, and Audit

| Command | Explanation |
|---|---|
| `aws bedrock put-model-invocation-logging-configuration --logging-config file://logging.json` | Enable logging to CloudWatch and/or S3 |
| `aws bedrock get-model-invocation-logging-configuration` | Current config |
| `aws bedrock delete-model-invocation-logging-configuration` | Disable |
| `aws logs tail /aws/bedrock/model-invocations --since 10m --follow` | Live invocation logs |
| `aws logs start-query --log-group-name /aws/bedrock/model-invocations --start-time $(date -d '-1 hour' +%s) --end-time $(date +%s) --query-string 'stats sum(input.inputTokenCount) as in_tok, sum(output.outputTokenCount) as out_tok by modelId'` | Token totals per model (Logs Insights) |
| `aws cloudwatch get-metric-statistics --namespace AWS/Bedrock --metric-name InvocationThrottles --dimensions Name=ModelId,Value=<MODEL_ID> --start-time ... --end-time ... --period 300 --statistics Sum` | Throttle count |
| `aws cloudwatch get-metric-statistics --namespace AWS/Bedrock --metric-name InvocationLatency --dimensions Name=ModelId,Value=<MODEL_ID> ... --statistics p99` | Latency p99 |
| `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=CreateGuardrail` | Who changed a guardrail |
| `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=bedrock.amazonaws.com --max-results 20` | Recent control-plane activity |

Logging config with S3 + CloudWatch and large-payload offload:

```json
{
  "cloudWatchConfig": {
    "logGroupName": "/aws/bedrock/model-invocations",
    "roleArn": "arn:aws:iam::123456789012:role/BedrockInvocationLogging",
    "largeDataDeliveryS3Config": {"bucketName": "<BUCKET_NAME>", "keyPrefix": "bedrock/large/"}
  },
  "s3Config": {"bucketName": "<BUCKET_NAME>", "keyPrefix": "bedrock/invocations/"},
  "textDataDeliveryEnabled": true,
  "imageDataDeliveryEnabled": true,
  "embeddingDataDeliveryEnabled": false,
  "videoDataDeliveryEnabled": false
}
```

---

## 10. Networking and Encryption

| Command | Explanation |
|---|---|
| `aws ec2 create-vpc-endpoint --vpc-id <VPC_ID> --vpc-endpoint-type Interface --service-name com.amazonaws.us-east-1.bedrock-runtime --subnet-ids <SUBNET_A> <SUBNET_B> --security-group-ids <SG_ID> --private-dns-enabled --policy-document file://vpce-policy.json` | PrivateLink endpoint for inference |
| `aws ec2 create-vpc-endpoint ... --service-name com.amazonaws.us-east-1.bedrock-agent-runtime` | For KB retrieval and agent invoke |
| `aws ec2 describe-vpc-endpoints --filters Name=service-name,Values=com.amazonaws.us-east-1.bedrock-runtime --query 'VpcEndpoints[].{id:VpcEndpointId,state:State,dns:DnsEntries[0].DnsName}'` | Verify |
| `aws ec2 delete-vpc-endpoints --vpc-endpoint-ids <VPCE_ID>` | Teardown (hourly cost) |
| `dig +short bedrock-runtime.us-east-1.amazonaws.com` (from inside the VPC) | Should return private 10.x/172.x addresses when private DNS is on |
| `aws kms create-key --description "bedrock-logs" --key-policy file://key-policy.json` | CMK for logs / KB / agents |
| `aws kms create-alias --alias-name alias/bedrock-logs --target-key-id <KEY_ID>` | Friendly alias |
| `aws logs associate-kms-key --log-group-name /aws/bedrock/model-invocations --kms-key-id <KEY_ARN>` | Encrypt invocation logs |

Endpoint policy limiting to one profile and one role:

```json
{
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"AWS": "arn:aws:iam::123456789012:role/payments-api-irsa"},
    "Action": ["bedrock:InvokeModel", "bedrock:InvokeModelWithResponseStream"],
    "Resource": [
      "arn:aws:bedrock:us-east-1:123456789012:inference-profile/us.anthropic.claude-sonnet-5-5",
      "arn:aws:bedrock:us-*::foundation-model/anthropic.claude-sonnet-5-5"
    ]
  }]
}
```

---

## 11. Quotas and Cost

| Command | Explanation |
|---|---|
| `aws service-quotas list-service-quotas --service-code bedrock --query 'Quotas[?contains(QuotaName, `Sonnet`)].{name:QuotaName,value:Value,adjustable:Adjustable}'` | Current TPM/RPM quotas for a model family |
| `aws service-quotas request-service-quota-increase --service-code bedrock --quota-code <QUOTA_CODE> --desired-value 2000` | Request increase (adjustable quotas only) |
| `aws service-quotas list-requested-service-quota-change-history --service-code bedrock` | Track requests |
| `aws ce get-cost-and-usage --time-period Start=2026-10-01,End=2026-10-31 --granularity MONTHLY --metrics UnblendedCost --filter '{"Dimensions":{"Key":"SERVICE","Values":["Amazon Bedrock"]}}' --group-by Type=DIMENSION,Key=USAGE_TYPE` | Bedrock spend by usage type (input/output tokens per model) |
| `aws ce get-cost-and-usage ... --group-by Type=TAG,Key=CostCenter` | Spend per application inference profile tag (after activating the cost allocation tag) |
| `aws budgets create-budget --account-id 123456789012 --budget file://budget.json --notifications-with-subscribers file://notify.json` | Monthly Bedrock budget alarm |

---

## 12. Evaluation, Prompts, Flows

| Command | Explanation |
|---|---|
| `aws bedrock create-evaluation-job --job-name <NAME> --role-arn <ROLE_ARN> --evaluation-config file://eval.json --inference-config file://eval-models.json --output-data-config '{"s3Uri":"s3://<BUCKET_NAME>/eval/"}'` | Automatic / LLM-judge / RAG evaluation |
| `aws bedrock get-evaluation-job --job-identifier <JOB_ARN>` | Status and output location |
| `aws bedrock-agent create-prompt --name <NAME> --variants file://variants.json --default-variant v1` | Versioned prompt template |
| `aws bedrock-agent create-prompt-version --prompt-identifier <PROMPT_ID>` | Freeze |
| `aws bedrock-runtime converse --model-id arn:aws:bedrock:us-east-1:123456789012:prompt/<PROMPT_ID>:1 --prompt-variables '{"service":{"text":"payments"}}'` | Invoke a managed prompt by ARN |
| `aws bedrock-agent create-flow --name <NAME> --execution-role-arn <ROLE_ARN> --definition file://flow.json` | Define DAG |
| `aws bedrock-agent prepare-flow --flow-identifier <FLOW_ID>` / `create-flow-alias` | Compile and alias |
| `aws bedrock-agent-runtime invoke-flow --flow-identifier <FLOW_ID> --flow-alias-identifier <ALIAS_ID> --inputs file://inputs.json out.bin` | Run |

---

## 13. Teardown Checklist (Run After Every Lab)

```bash
aws bedrock list-provisioned-model-throughputs --query 'provisionedModelSummaries[].provisionedModelArn' --output text \
  | xargs -r -n1 aws bedrock delete-provisioned-model-throughput --provisioned-model-id
aws opensearchserverless list-collections --query 'collectionSummaries[].id' --output text \
  | xargs -r -n1 aws opensearchserverless delete-collection --id
aws bedrock-agent list-knowledge-bases --query 'knowledgeBaseSummaries[].knowledgeBaseId' --output text \
  | xargs -r -n1 aws bedrock-agent delete-knowledge-base --knowledge-base-id
aws bedrock-agent list-agents --query 'agentSummaries[].agentId' --output text \
  | xargs -r -n1 -I{} aws bedrock-agent delete-agent --agent-id {} --skip-resource-in-use-check
aws ec2 describe-vpc-endpoints --filters Name=service-name,Values='com.amazonaws.*.bedrock*' --query 'VpcEndpoints[].VpcEndpointId' --output text \
  | xargs -r aws ec2 delete-vpc-endpoints --vpc-endpoint-ids
agentcore destroy 2>/dev/null || true
```

Guardrails, application inference profiles, prompts, and flows have no hourly cost; delete them for hygiene, not for money.

Next: [04-hands-on-labs.md](04-hands-on-labs.md).
