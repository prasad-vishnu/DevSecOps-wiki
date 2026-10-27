# ADR-0004: LangGraph for the stateful incident agent, Strands Agents for lightweight tool agents

| | |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-10-27 |
| **Deciders** | AI lead, SRE lead |
| **Cluster** | 03 Agentic Frameworks |
| **Capstone phase** | Phase 6 |

## Context

The Incident & Platform Agent must: gather alert context, retrieve runbooks, query Prometheus and Kubernetes through MCP servers, propose a remediation, **pause for human approval before any destructive action** (for example `rollout restart`), resume possibly hours later when an on-call engineer approves in Slack, survive pod restarts mid-incident, and leave an auditable, replayable trail. The same team also wants small, disposable agents for tasks like "summarise this postmortem" or "draft a Jira ticket from this alert" where a durable graph is overkill.

Forces:

- **Durability and resumability.** An incident can span hours; the agent's state must persist across process restarts and be keyed by incident id.
- **Human-in-the-loop.** Approval gates are a hard safety requirement (OWASP LLM Top 10: excessive agency).
- **Auditability.** Every step must be inspectable and replayable (time travel) for postmortems.
- **Model portability.** Bedrock Claude by default, self-hosted vLLM models as a cost fallback, Vertex and Azure OpenAI for the bridge exercises.
- **Observability.** OpenTelemetry tracing of every LLM and tool call is mandatory (Golden Rule 3).
- **Developer velocity.** Small agents should take minutes to write.

## Decision

We will build the **incident agent on LangGraph** as an explicit `StateGraph` with a typed state schema, a **Postgres checkpointer** keyed by `thread_id = incident_id`, `interrupt()` nodes for approval, and `Command(resume=...)` to continue after a Slack approval. Tools come from MCP servers via `langchain-mcp-adapters`. We will build **lightweight, single-purpose agents on Strands Agents**, where the model drives the loop over `@tool` functions and MCP clients with built-in OpenTelemetry tracing, deployed as Lambda functions or small Deployments.

LangGraph's checkpointing, time travel (`get_state_history`, `update_state`) and first-class interrupts map directly onto the durability, approval and audit requirements. Strands' minimal surface (model, system prompt, tools) is the right fit when there is no long-lived state and no destructive action.

## Alternatives Considered

| Option | Pros | Cons | Why rejected (or deferred) |
|---|---|---|---|
| Strands Agents for the incident agent | Fastest to write; model-driven loop handles branching naturally; native MCP client; OTel built in; Bedrock-first | Persistence and approval gates are session-manager and hook based rather than a first-class graph interrupt; replaying a specific step for a postmortem is less direct; model-driven loops are harder to constrain for destructive tools | Chosen for lightweight agents. Re-evaluate as Strands' multi-agent and session primitives mature |
| LangChain `AgentExecutor` / legacy agents | Familiar | Deprecated in favour of LangGraph; no checkpointing | Rejected |
| Bedrock Agents / AgentCore Runtime | Managed runtime, memory, gateway, identity; no servers | AWS-only; action groups via OpenAPI/Lambda add indirection; approval flow is less flexible; harder to run the same agent on kind | Deferred; documented as the managed deployment target for the Strands agents |
| Custom ReAct loop in plain Python | Zero dependencies; full control | Re-implements checkpointing, interrupts, streaming and tracing badly | Rejected |
| CrewAI / AutoGen / OpenAI Agents SDK | Multi-agent abstractions | Not in the roadmap; weaker durability story or provider lock-in | Rejected for the capstone |

## Consequences

### Positive

- Approval gates and durability are declarative properties of the graph, not scattered application logic.
- Postmortems can replay the agent's exact decision path from the checkpoint history.
- Model provider is swappable through `init_chat_model`; the same graph runs on Bedrock, vLLM, Vertex or Azure OpenAI.
- Small agents stay small.

### Negative / Risks

- Two agent frameworks to maintain. Mitigation: shared tool layer (MCP servers) so tools are written once; shared OTel conventions; ADR revisit every phase.
- LangGraph API churn. Mitigation: pin versions in `uv.lock`; integration tests for the graph; a thin adapter around checkpointer and interrupt APIs.
- Postgres checkpointer stores prompts and tool outputs, which may include sensitive data. Mitigation: redact at the tool boundary, encrypt at rest, 30-day retention, RBAC on the checkpoint schema.
- Model-driven Strands loops could call tools unexpectedly. Mitigation: Strands agents are only given read-only tools; destructive tools exist solely behind the LangGraph interrupt.

### Neutral / Follow-ups

- Build the same incident agent in Strands as a bridge exercise and record a comparison table (lines of code, time to first working version, how approval was implemented, trace quality).
- Evaluate LangGraph Platform or AgentCore Runtime if hosting the agent outside the cluster becomes attractive.

## Compliance and Security Impact

Destructive Kubernetes actions require a recorded human approval with the approver's identity stored in the checkpoint. The Kubernetes MCP server's ServiceAccount is read-only except for one verb on one resource type. All prompts, retrieved contexts and tool calls are traced with `gen_ai.*` attributes and sampled into evaluation datasets.

## Cost Impact

Framework cost is zero. Token cost is governed by model routing: a small vLLM-hosted model for context gathering and classification, Bedrock Claude for the reasoning step, with semantic caching of runbook summaries. Target: under $0.05 per triage run on the lab budget.

## Validation

DeepEval `ToolCorrectnessMetric` ≥ 0.9 and `TaskCompletionMetric` ≥ 0.8; 100% of destructive actions in the test suite blocked until approval; successful resume after a forced pod restart mid-incident. Review at end of Phase 6.

## References

- [../../03-agentic-frameworks/README.md](../../03-agentic-frameworks/README.md)
- [../../03-agentic-frameworks/langgraph/](../../03-agentic-frameworks/langgraph/)
- [../../03-agentic-frameworks/strands-agents/](../../03-agentic-frameworks/strands-agents/)
- [../../10-ai-infra-rag-mcp/mcp/](../../10-ai-infra-rag-mcp/mcp/)
