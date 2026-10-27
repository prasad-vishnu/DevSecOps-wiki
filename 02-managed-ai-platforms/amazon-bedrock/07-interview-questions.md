# Amazon Bedrock: Interview Questions

| | |
|---|---|
| **Cluster** | 02 Managed AI Platforms |
| **Role** | Anchor |
| **Prerequisites** | Whole folder; Labs 1-8 strongly recommended |
| **Target version** | Bedrock GA APIs, 2026 |
| **Capstone phase** | Phase 6 (Months 21-24) |

> Thirty questions from junior to principal level with model answers. Use them to self-test and to prepare for platform, AI infrastructure, and principal engineer interviews. Answers are deliberately concise; be ready to expand each with an example from your capstone.

---

## Junior

### Q1. What is Amazon Bedrock and how does it differ from calling a model vendor's public API? (Junior)

**Answer.** Bedrock is an AWS-managed, serverless service that exposes foundation models from multiple vendors behind AWS APIs. The difference is operational, not model quality: calls are authenticated with IAM (no API keys), recorded in CloudTrail, reachable over PrivateLink inside a VPC, encrypted with KMS where data is stored, billed on the AWS invoice, and AWS contractually does not use prompts or completions to train models or share them with vendors.

### Q2. Name the Bedrock API endpoints and what each is for. (Junior)

**Answer.** `bedrock` (control plane: model access, provisioned throughput, logging, guardrails, evaluation), `bedrock-runtime` (inference: Converse, InvokeModel, ApplyGuardrail), `bedrock-agent` (build Knowledge Bases, Agents, prompts, flows), `bedrock-agent-runtime` (Retrieve, RetrieveAndGenerate, InvokeAgent), and `bedrock-agentcore` / `bedrock-agentcore-control` for AgentCore.

### Q3. What is the difference between `InvokeModel` and `Converse`? (Junior)

**Answer.** `InvokeModel` takes the provider's native JSON body (Anthropic Messages schema, Nova schema), so switching models means rewriting request and response handling, but it exposes provider features immediately. `Converse` is Bedrock's unified chat schema (`system`, `messages`, `toolConfig`, `inferenceConfig`, `guardrailConfig`) that works across chat models with a `modelId` change. Use Converse by default; use InvokeModel for embeddings, image models, and provider-specific features not yet mapped.

### Q4. How do you find which models are available and whether you can call them? (Junior)

**Answer.** `aws bedrock list-foundation-models` lists models with `inferenceTypesSupported`; `aws bedrock list-inference-profiles` lists cross-region profile IDs; `aws bedrock get-foundation-model-availability --model-id <ID>` reports agreement and entitlement status. Model IDs change over time, so they belong in configuration, not code.

### Q5. What does `stopReason: tool_use` mean and what must your code do? (Junior)

**Answer.** The model has emitted one or more `toolUse` blocks and is waiting for results. The code must execute each tool, append the assistant message as returned, then append a single user message containing a `toolResult` block for every `toolUseId`, and call Converse again. Repeat until `stopReason` is `end_turn` or an iteration cap is hit.

### Q6. What is an inference profile? (Junior)

**Answer.** A routing object used in place of a model ID. System-defined profiles (`us.`, `eu.`, `apac.`, `global.` prefixes) route requests across regions in a geography for higher capacity; application profiles are user-created wrappers that carry tags for cost allocation and can be targeted by IAM. Many newer models can only be invoked through a profile.

### Q7. How is streaming delivered and how do you consume it in Python? (Junior)

**Answer.** `ConverseStream` returns an AWS EventStream; boto3 exposes `response["stream"]` as an iterator of single-key dicts: `messageStart`, `contentBlockStart`, `contentBlockDelta` (text or tool-input fragments), `contentBlockStop`, `messageStop` (with `stopReason`), and `metadata` (usage, latency). You concatenate `delta.text` for display and accumulate `delta.toolUse.input` per block index, parsing JSON at `contentBlockStop`.

---

## Mid-level

### Q8. Why might `AccessDeniedException` appear intermittently when using a cross-region inference profile, and how do you fix it? (Mid-level)

**Answer.** The profile routes each request to one of several regions; authorisation is evaluated against both the profile ARN and the destination region's `foundation-model` ARN. If the identity policy lists only the profile (or only one region's model ARN), requests routed elsewhere fail. Fix by including `arn:aws:bedrock:us-*::foundation-model/<MODEL>` (or each destination region explicitly) alongside the profile ARN.

### Q9. Compare on-demand, provisioned throughput, and batch inference. When do you use each? (Mid-level)

**Answer.** On-demand bills per token from a shared pool with account quotas; good for dev, bursty or modest production. Provisioned throughput reserves model units billed hourly (with 1- or 6-month commitment discounts) for predictable p99 latency and is required for custom or imported models. Batch inference processes JSONL from S3 asynchronously at roughly half the on-demand price for non-interactive work such as evaluations, backfills, and nightly summarisation.

### Q10. Explain how a Knowledge Base processes a document from S3 to a retrievable chunk. (Mid-level)

**Answer.** An ingestion job lists the S3 prefix, parses each supported file (default parser or FM-based parsing), reads the optional `<file>.metadata.json` sidecar, splits text according to the data source's chunking strategy (fixed, hierarchical, semantic, none, or a Lambda transformation), embeds each chunk with the configured embedding model (for example Titan Text Embeddings V2 at 1024 dimensions), and writes vector, text, and metadata into the vector store (OpenSearch Serverless, Aurora pgvector, Pinecone, Neptune Analytics, S3 Vectors). Sync is incremental; failed documents are reported in job statistics.

### Q11. When would you use `Retrieve` instead of `RetrieveAndGenerate`? (Mid-level)

**Answer.** `Retrieve` returns scored chunks with locations and metadata, leaving prompting and generation to you; use it when an agent framework (Strands, LangGraph) owns the loop, when you need custom prompts or multiple retrieval rounds, or when you want to apply your own reranking and filtering. `RetrieveAndGenerate` is the one-call path with citations and optional `sessionId` memory, ideal for simple Q&A endpoints.

### Q12. What policy types does a Bedrock Guardrail support, and what is the difference between `BLOCK` and `ANONYMIZE` for PII? (Mid-level)

**Answer.** Denied topics, content filters (hate, insults, sexual, violence, misconduct, prompt attack with strength levels), word filters (managed profanity and custom lists), sensitive information (built-in PII entities plus custom regex), contextual grounding (grounding and relevance thresholds), and automated reasoning policies. For PII, `BLOCK` rejects the whole input or output and returns the configured blocked message; `ANONYMIZE` replaces each detected entity with a placeholder such as `{EMAIL}` and lets the rest through.

### Q13. How do `guardContent` qualifiers work and why do they matter for contextual grounding? (Mid-level)

**Answer.** Wrapping text in `guardContent` with `qualifiers` tells the guardrail the role of that text: `query` is the user question, `grounding_source` is the retrieved context, `guard_content` is text to filter. Contextual grounding scores the model's response against the `grounding_source`; without the qualifier the guardrail has no source and either skips the check or scores everything as ungrounded.

### Q14. What is prompt caching in Converse and what commonly breaks it? (Mid-level)

**Answer.** A `cachePoint` content block marks a prefix (system text, tool definitions, early messages) to be cached for roughly five minutes; subsequent requests with a byte-identical prefix pay a discounted cache-read rate. It breaks when anything before the checkpoint changes (timestamps, request IDs, reordered tools), when the prefix is shorter than the model's minimum cacheable length, or when the model does not support caching. Verify with `usage.cacheReadInputTokens`.

### Q15. Describe the Lambda contract for a Bedrock Agent action group. (Mid-level)

**Answer.** The agent invokes the Lambda with an event containing `messageVersion`, `agent`, `actionGroup`, either `function` + `parameters` (function schema) or `apiPath` + `httpMethod` + `requestBody` (OpenAPI schema), and session attributes. The Lambda returns `messageVersion: "1.0"` and a `response` containing the action group, function or API path, and a `functionResponse.responseBody` (or `httpStatusCode` + `responseBody`). The Lambda's resource policy must allow `bedrock.amazonaws.com` with the agent ARN as `aws:SourceArn`.

### Q16. How do you make a service's Bedrock traffic private? (Mid-level)

**Answer.** Create interface VPC endpoints for `com.amazonaws.<region>.bedrock-runtime` (and `bedrock-agent-runtime` if using KBs/agents) in the app's subnets with private DNS enabled so the public hostname resolves to private ENIs; restrict the endpoint's security group to 443 from app subnets; attach an endpoint policy allowing only the app role and approved model ARNs; add an `aws:SourceVpce` condition on the app role's identity policy; remove NAT routes from app subnets so the public path cannot exist.

### Q17. What does model invocation logging capture, where can it go, and what are the security considerations? (Mid-level)

**Answer.** It captures full request and response bodies (text; images and embeddings optional), token counts, caller identity, model ID, resolved inference region, and `requestMetadata`, delivered to CloudWatch Logs, S3, or both, with large payloads offloaded to S3. Since it contains prompts and completions it must be encrypted with a CMK, access-restricted, retention-limited, and kept free of PII in `requestMetadata`; disabling it should be denied by SCP except for a break-glass role.

### Q18. How does botocore's retry configuration relate to Bedrock throttling, and what would you add at application level? (Mid-level)

**Answer.** `Config(retries={"mode": "standard" | "adaptive", "max_attempts": N})` retries `ThrottlingException`, `ServiceUnavailableException` and connection errors with exponential backoff and jitter; adaptive mode adds a client-side rate limiter. At application level add a token bucket sized to the TPM quota, a work queue for bursts, streaming for long outputs, a circuit breaker that fails over to a second profile or cheaper model, and metrics on retries and throttles so SLOs account for them.

---

## Senior

### Q19. Design IAM so that no application call can bypass the approved guardrail. (Senior)

**Answer.** Use the `bedrock:GuardrailIdentifier` condition key: an explicit `Deny` on `bedrock:InvokeModel*` when `StringNotEquals bedrock:GuardrailIdentifier` the approved guardrail ARN(s), plus an `Allow` for `bedrock:ApplyGuardrail` on that guardrail. Any call without `guardrailConfig` or with a different guardrail is rejected at authorisation time regardless of application code. Pin the version via configuration so updates are deliberate, and enforce the same via SCP for all accounts in the OU.

### Q20. You see `InvocationThrottles` rising while provisioned throughput utilisation is at 60%. What is happening and what do you check? (Senior)

**Answer.** Likely the throttled calls are on-demand (different `ModelId` dimension) rather than the provisioned ARN, meaning some code path is not using the provisioned model ARN; or bursts exceed per-minute MU capacity even though average utilisation is 60% (MU throughput is per minute, bursts within a minute throttle); or the throttles are on a different API (KB `Retrieve`). Check metrics by dimension, invocation logs for `modelId` distribution, and the time distribution of calls. Fixes: route all traffic to the provisioned ARN with on-demand spillover, smooth bursts with a client queue, or add MUs.

### Q21. Compare OpenSearch Serverless and Aurora PostgreSQL pgvector as Knowledge Base stores for a 200k-chunk runbook corpus. (Senior)

**Answer.** OpenSearch Serverless offers hybrid BM25 + vector search, managed scaling, and quick setup, but its OCU minimum makes it the dominant fixed cost for small corpora and it requires a separate network policy and data-access policy. Aurora pgvector (Serverless v2) is cheaper at this scale, supports joins with business tables and familiar backup/DR, and can scale to a fraction of an ACU when idle, but the KB integration does not provide BM25 hybrid search and you manage schema, extensions, and Secrets Manager credentials. For 200k chunks with a need for lexical matching on exact error strings, OpenSearch Serverless; for cost-sensitive, mostly semantic retrieval tied to relational metadata, Aurora. Record the decision in an ADR and benchmark recall with RAGAS context recall.

### Q22. How would you implement tenant isolation for a multi-tenant RAG service on Bedrock? (Senior)

**Answer.** Tag every chunk with `tenant_id` metadata and enforce a server-side filter in every `Retrieve` call (never trust the client to send it); for regulated tenants use separate Knowledge Bases or separate accounts. Use per-tenant application inference profiles for cost attribution, per-tenant client token buckets so one tenant cannot consume the account quota, `requestMetadata.tenant_id` on every call for audit, and tenant-scoped credentials for tool execution so a prompt injection in one tenant's documents cannot act on another tenant's systems.

### Q23. What are the differences between Bedrock Agents and AgentCore, and when do you choose each? (Senior)

**Answer.** Bedrock Agents is a fully managed, opinionated orchestrator: you supply model, instructions, action groups (OpenAPI/function schema + Lambda or return-control), and KBs; Bedrock runs the ReAct loop with its prompt templates and session memory. AgentCore is a set of framework-agnostic primitives (Runtime microVMs with an HTTP/MCP contract, Gateway to expose APIs as MCP tools with auth, Memory with long-term extraction, Identity for inbound JWT and outbound OAuth tokens, Observability via OpenTelemetry, Code Interpreter and Browser sandboxes) for hosting agents written in Strands, LangGraph, or anything else. Choose Agents when you want zero agent code and accept its orchestration; choose AgentCore when you need custom loops, framework portability, MCP tooling, or enterprise identity.

### Q24. How do you prevent prompt injection from retrieved documents turning into harmful tool execution? (Senior)

**Answer.** Defence in depth: guardrail `PROMPT_ATTACK` filter on input and `guardContent` qualification so retrieved text is treated as data; system-prompt instructions that tool calls must derive from the user request; strict JSON Schema validation of tool inputs before execution; tool allow-lists per task and least-privilege, short-lived credentials per tool call; human confirmation (`requireConfirmation` in agents, or return-control) for destructive actions; iteration and token caps; and red-team evaluation with Promptfoo in CI. This addresses OWASP LLM01 (prompt injection) and LLM08 (excessive agency).

### Q25. Describe a CI/CD pipeline for prompts, guardrails, and knowledge base configuration. (Senior)

**Answer.** Prompts, guardrail definitions, chunking configs, and agent definitions live in Git as Terraform plus prompt files. On PR, GitHub Actions assumes a dev role via OIDC, applies to a dev account, runs DeepEval/RAGAS suites (faithfulness, answer relevancy, context recall) and a PII-leak probe set against the dev guardrail and KB, and blocks merge on regression. On merge, it applies to prod, creates a new guardrail version and prompt version, and updates SSM parameters that applications read; rollback is a parameter change. KB ingestion is event-driven with alarms on failed documents.

---

## Principal

### Q26. A regulator asks for proof that no customer PII reached the model unredacted in the last 90 days. How does your Bedrock architecture answer? (Principal)

**Answer.** Input-side guardrail with `ANONYMIZE` on PII entities, enforced by an IAM deny on calls without that guardrail, so no code path could bypass it. Invocation logs delivered to an S3 bucket with SSE-KMS and Object Lock in compliance mode for 90+ days, containing the post-guardrail input, `requestMetadata` with tenant and trace IDs, and the caller identity. CloudTrail proves the logging configuration was never deleted (SCP denies it) and shows every guardrail version change. Guardrail metrics (`InvocationsIntervened`) and a sampled Logs Insights scan for PII patterns provide statistical evidence. The weakness to disclose honestly: guardrail PII detection is probabilistic; residual risk is quantified by red-team tests whose results are also archived.

### Q27. Argue for or against making Bedrock the sole model provider for a platform used by 40 product teams. (Principal)

**Answer.** For: single IAM, network, logging, and billing story; Converse makes multi-vendor model choice a config change; Guardrails and KBs are shared platform capabilities; quotas and costs are governable with application inference profiles and SCPs. Against: Converse lags the newest provider features, some models and features ship elsewhere first, regional availability gaps, and account-level quotas mean 40 teams share one pool unless you shard accounts; vendor lock-in is to AWS primitives rather than to a model. A principal answer is a platform abstraction (LangChain/Strands provider interface) with Bedrock as default, an approved-exception path to another provider (Vertex or Azure OpenAI via the same abstraction), portable evals, and a quarterly review of the exception list.

### Q28. How would you size and operate provisioned throughput for a service with a strict TTFT SLO and a 10x daily peak-to-trough ratio? (Principal)

**Answer.** Measure two weeks of on-demand traffic via invocation logs to get tokens per minute percentiles. Provision MUs for the sustained p50-p80 baseline (committed term for discount), route baseline traffic to the provisioned ARN, and spill over to a cross-region on-demand profile for the peak with a client-side router that watches throttles and MU utilisation. Keep the TTFT SLO on the provisioned path and a looser SLO on spillover, exposed as separate SLIs. Schedule MU scale-up ahead of predictable peaks (no-commit MUs where available), alert on utilisation above 80%, and review monthly. Where the peak is batchable (reports, digests), move it to batch inference to flatten the curve.

### Q29. A team reports Bedrock answer quality regressed after "no changes". Walk through your investigation. (Principal)

**Answer.** Pull invocation logs for the period and diff `modelId` (a profile may have started routing to a different model version), guardrail version, prompt version, and `cacheReadInputTokens` (a broken cache changes nothing semantically but a changed prefix may indicate an unnoticed prompt edit). Check KB ingestion history for a sync that added or removed documents and `retrievalResults` scores for drift. Replay a fixed evaluation set (DeepEval/RAGAS) against current and previous configurations in a dev account to localise the regression. Check CloudTrail for configuration changes by any principal. Most often the root cause is a KB sync, an untracked prompt change, or model routing; the fix is pinning versions and gating changes with evals.

### Q30. What architectural decisions would you record in ADRs for the capstone's Bedrock integration, and what alternatives would each cite? (Principal)

**Answer.** (1) Converse via boto3 as the provider interface versus the Anthropic SDK's Bedrock client: vendor neutrality versus day-one feature access. (2) Knowledge Bases versus self-hosted Qdrant for runbook RAG: managed ingestion and IAM integration versus hybrid search control, cost, and portability to vLLM-based stacks. (3) Guardrails versus application-level filtering: model-agnostic, IAM-enforceable policy versus latency and per-text-unit cost. (4) AgentCore Runtime versus EKS-hosted Strands agent: managed session isolation, memory, and identity versus reuse of the existing Kubernetes platform, Argo CD GitOps, and observability stack. (5) Cross-region profile versus provisioned throughput for the agent's model: elasticity versus deterministic latency. Each ADR states context, decision, consequences, and the evaluation evidence (RAGAS/DeepEval numbers, cost model) that supported it.

---

Back to [README.md](README.md).
