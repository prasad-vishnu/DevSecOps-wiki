# LangGraph: Interview Questions

| | |
|---|---|
| **Cluster** | 03 Agentic Frameworks |
| **Role** | Anchor |
| **Prerequisites** | The whole folder |
| **Target version** | LangGraph 1.x |
| **Capstone phase** | Phase 6 (Months 21-24) |

> Thirty questions from junior to principal with model answers. Use them to check understanding after the labs and to prepare for agent-engineering interviews; the principal-level ones expect you to reason about failure modes, security and trade-offs, not recite APIs.

---

## Junior

### Q1. What are the four components of an agent? (Junior)

**Answer.** An LLM that can emit tool calls, a set of tools with JSON schemas, a loop that executes tool calls and feeds results back until the model stops calling tools, and state that lets the loop persist across steps and processes. LangGraph makes the loop and the state explicit.

### Q2. What is the difference between a chain and a graph in the LangChain ecosystem? (Junior)

**Answer.** A chain (LCEL) is a static DAG: each runnable passes its output to the next, no cycles, no persistence. A LangGraph graph has typed state, arbitrary edges including cycles and conditional branches, and a checkpointer, which enables loops, pausing and resumption.

### Q3. What do `START` and `END` represent? (Junior)

**Answer.** Sentinel node names. `add_edge(START, "x")` defines the entry; routing to `END` stops the graph. They are constants, not strings you type yourself.

### Q4. What does a node return? (Junior)

**Answer.** A partial state update: a dict containing only the keys it changed. The runtime merges it into the state using the key's reducer (or overwrites if none). Returning the whole mutated state is a bug waiting to happen.

### Q5. What is `thread_id` for? (Junior)

**Answer.** It identifies a persisted conversation or workflow instance in the checkpointer. Runs with the same `thread_id` share history; `invoke(None, config)` resumes the thread.

### Q6. What does `create_react_agent` build? (Junior)

**Answer.** A two-node graph: `agent` (model with tools bound) and `tools` (`ToolNode`), with a conditional edge from `agent` that goes to `tools` if the last message has tool calls and to `END` otherwise, and a fixed edge from `tools` back to `agent`.

### Q7. How do you check which LangGraph version you have? (Junior)

**Answer.** `pip show langgraph` or `importlib.metadata.version("langgraph")`. It matters because keyword names (`retry` -> `retry_policy`), class aliases (`MemorySaver`/`InMemorySaver`) and the `__interrupt__` return key changed across 0.2 -> 1.0.

---

## Mid-level

### Q8. Explain reducers and why `add_messages` is special. (Mid-level)

**Answer.** A reducer defines how a new value for a state key merges with the existing one; without one the value is overwritten. `operator.add` concatenates lists. `add_messages` appends messages but upserts by message `id`, so returning a message with an existing id replaces it, and `RemoveMessage(id)` deletes it. That gives you history editing and trimming through the normal update path.

### Q9. What is a super-step and why does it matter? (Mid-level)

**Answer.** LangGraph executes Pregel-style: all currently active nodes run in parallel in one super-step, their updates are applied together through reducers, and a checkpoint is written. Nodes in the same super-step see the same input state and cannot see each other's writes. Two nodes writing the same key in one super-step need a reducer or you get `InvalidUpdateError`. `recursion_limit` counts super-steps.

### Q10. How does `interrupt()` work internally? (Mid-level)

**Answer.** The node calls `interrupt(payload)`; the runtime records the interrupt in the checkpoint and ends the run, returning `__interrupt__`. When the caller invokes `Command(resume=value)` on the same thread, the node is re-executed from its start and this time `interrupt()` returns `value`. Hence code before the call runs twice and must be idempotent. It requires a checkpointer.

### Q11. Compare `interrupt()` with `interrupt_before`. (Mid-level)

**Answer.** `interrupt_before=["node"]` is a static breakpoint set at compile time: it pauses before the node every time and resumes with `invoke(None)`; no value is passed back. `interrupt()` is dynamic: called conditionally inside a node, carries a payload out and a resume value in. Use static breakpoints for debugging and coarse gates; dynamic interrupts for product approval flows.

### Q12. What is the difference between a checkpointer and a store? (Mid-level)

**Answer.** The checkpointer persists per-thread state after every super-step (short-term memory, resumption, time travel). The `BaseStore` is cross-thread key-value memory with namespaces and optional vector search (long-term memory such as "payments-api is owned by team-checkout"). Both are injected at `compile()`.

### Q13. What are the streaming modes and when do you use each? (Mid-level)

**Answer.** `values` (full state per step, debugging), `updates` (node name to partial update, progress UIs), `messages` (LLM token chunks with node metadata, chat UIs), `custom` (anything a node emits via `get_stream_writer`, status lines), `debug` (executor events). You can request a list of modes and get `(mode, chunk)` tuples; `subgraphs=True` adds nested graph events.

### Q14. How do you implement map-reduce in LangGraph? (Mid-level)

**Answer.** A conditional edge whose router returns `[Send("worker", payload) for payload in items]`. Each `Send` becomes a task in the next super-step with its own input; worker results merge into a parent key with an additive reducer; a fan-in edge from the worker to the reduce node runs the reducer once.

### Q15. How do subgraphs exchange state with the parent? (Mid-level)

**Answer.** Either the subgraph shares key names with the parent and is added directly as a node (only overlapping keys flow), or you wrap it in a function that maps parent state to subgraph input and output back. Subgraphs get their own `checkpoint_ns`; interrupts propagate to the parent.

### Q16. How would you test a graph without calling a real model? (Mid-level)

**Answer.** Subclass `BaseChatModel` to return scripted `AIMessage`s with `tool_calls` and implement `bind_tools` to return itself; compile with `MemorySaver`; assert on `get_state(cfg).next`, on the final state and on `get_state_history` to check the path. Test both resume branches with `Command(resume=...)`. Unit-test nodes by calling the functions with dicts.

### Q17. What does `Command` do and when would you return it from a node? (Mid-level)

**Answer.** `Command(goto=..., update=...)` lets a node decide the next node and update state in one return value, replacing a separate conditional edge. `Command(graph=Command.PARENT, goto=...)` routes from a subgraph to the parent, which is how agent handoffs work. `Command(resume=...)` is the input form used to resume interrupts.

---

## Senior

### Q18. Design the state for an incident-triage agent and justify each key. (Senior)

**Answer.** `messages` with `add_messages` for the ReAct transcript; `alert` (input, overwritten); `plan` as a validated Pydantic dump so the executor acts on structured data, not prose; `approved` tri-state (`None`/`True`/`False`) to distinguish "not yet asked" from "rejected"; `remediated` marker for idempotency on resume; `report` for output. Secrets never go in state because it is checkpointed and traced. Add `input_schema`/`output_schema` to narrow the API contract.

### Q19. Where do you put a side effect relative to an approval gate, and why? (Senior)

**Answer.** In a node *after* the `interrupt()` node, with a re-check of `approved` and a completion marker. Reasons: the interrupting node re-runs on resume, so anything before `interrupt()` executes twice; `update_state` can alter `approved`, so the executor must validate; and at-least-once execution means a crash after the mutation but before the checkpoint can re-run the node, so the marker and an idempotent tool protect you.

### Q20. How do checkpoints interact with horizontal scaling? (Senior)

**Answer.** Checkpoints are per-thread rows written transactionally, so any replica can resume any thread. The hazard is two replicas resuming the same thread concurrently (double approval clicks); serialise per `thread_id` with a lock or a `FOR UPDATE` row in your own incidents table. Use a shared async pool per process, `prepare_threshold=0` behind pgbouncer, and `durability="sync"` on gated paths.

### Q21. Explain the security boundary between read-only and mutating tools in a LangGraph design. (Senior)

**Answer.** Bind only read-only tools to the model; the mutating tool is invoked by code in an executor node after a durable approval. Run the executor under a separate ServiceAccount with namespace-scoped `patch` RBAC, ideally in a separate process. Treat tool output as untrusted data (delimiters, system prompt instruction, size truncation, secret scrubbing). This addresses OWASP LLM01 (injection) and LLM06 (excessive agency) structurally rather than with prompt wording.

### Q22. Compare supervisor, swarm and hierarchical multi-agent patterns. (Senior)

**Answer.** Supervisor: a central agent routes to worker agents (as tools or nodes) and they return to it; auditable, predictable, one bottleneck. Swarm: agents hand off directly via handoff tools and the last active agent answers; natural for conversations, harder to audit. Hierarchical: supervisors of supervisors as subgraphs; scales organisationally at the cost of latency. In LangGraph all are graphs; `langgraph-supervisor` and `langgraph-swarm` are reference implementations built on `Command(graph=PARENT)`.

### Q23. How do you keep cost bounded in a tool-calling loop? (Senior)

**Answer.** `recursion_limit` with `GraphRecursionError` handling; a per-thread tool-call budget counted in state; trimming messages with `RemoveMessage`; truncating tool output; stable prompt prefix for provider prompt caching; routing cheap nodes to a small model; `CachePolicy` on deterministic nodes; token counters from `usage_metadata` exported to Prometheus with alerts.

### Q24. What are the trade-offs of LangGraph Server versus embedding the graph in FastAPI? (Senior)

**Answer.** Server gives threads/runs/cron/Studio/SDK contracts and auth hooks with no route code, and `langgraph.json` as the deployment unit; but it adds a licensing dimension, its own Postgres/Redis requirements, and less control over middleware, auth and streaming semantics. Embedding gives full control and reuses your existing mesh/OTel/RBAC, at the cost of writing and maintaining the API surface. The capstone embeds because the platform already exists.

### Q25. How do you evolve the state schema without breaking in-flight threads? (Senior)

**Answer.** Add keys with `None`-tolerant node code; avoid renaming or changing reducers; drain or expire interrupted threads before incompatible changes; version the graph (`assistant_id`/image tag) and let old threads finish on the old version. Keep a side table of open threads to know what is in flight.

---

## Principal

### Q26. Argue for or against making the agent decide its own control flow versus encoding it in the graph. (Principal)

**Answer.** Model-driven flow (Strands style, or a single ReAct node) minimises code and adapts to novel situations, but guarantees nothing about order: it cannot promise "never act before consulting the runbook". Graph-driven flow trades flexibility for enforceable invariants, testability (trajectory tests), and auditability. For infrastructure remediation, invariants dominate: encode validate -> gather -> propose -> approve -> act as nodes, and let the model be free only inside the gather loop. Use the ReAct node for open-ended investigation, the graph for anything with consequences.

### Q27. A prompt-injected runbook makes the model propose deleting a namespace. Walk through every layer that should stop it. (Principal)

**Answer.** Ingestion: runbooks come only from a reviewed Git repo, so the injection must pass code review. Retrieval: payload filters restrict to the service's runbooks. Prompt: tool output delimited and declared as data. Schema: `RemediationPlan.action` is an enum without `delete`. Graph: the executor only implements `rollout_restart`; unknown actions report and stop. Approval: a human sees the plan. RBAC: the executor's ServiceAccount cannot delete namespaces. Observability: the trace and the Promptfoo CI suite catch it earlier. No single layer is trusted.

### Q28. How would you achieve exactly-once remediation given at-least-once node execution? (Principal)

**Answer.** You cannot get exactly-once across a network; you get idempotent at-least-once. The mutation itself is idempotent (patching the same annotation twice is harmless, or use a client-side idempotency key stored in state before the call). The node checks a `remediated` marker, performs the action, writes the marker in the same update, and the checkpoint is written with `durability="sync"`. A crash between the API call and the checkpoint re-runs the node, which re-patches harmlessly. For non-idempotent actions, record intent first (`remediation_requested_id`), then act, then confirm, mirroring the saga pattern.

### Q29. Design the observability for this agent so an SRE can answer "why did it propose that?" six months later. (Principal)

**Answer.** Checkpoints retained for the audit window give the exact state at every step; `get_state_history` reconstructs the trajectory. OTel spans with `thread_id`, model id, token counts and tool arguments go to Tempo; logs with the same `thread_id` go to Loki; metrics on outcomes, tokens and tool errors go to Prometheus with exemplars linking to traces. Redact secrets at the Collector. Store the approver identity and reason in state. Pin the prompt and tool versions in span attributes (git SHA) so you can diff behaviour across releases.

### Q30. When would you advise a team not to use LangGraph? (Principal)

**Answer.** When the workload is a stateless chain (classification, extraction) where LCEL or a plain function suffices; when the team deploys exclusively on Lambda or Bedrock AgentCore and wants AWS-native defaults (Strands); when there is no appetite to run Postgres/Redis for checkpoints; when the team is small and the churn in the LangChain ecosystem is a maintenance risk they cannot absorb; or when a vendor's managed agent product already meets the requirements and avoiding custom orchestration code is the priority. Recommend LangGraph when explicit control flow, durable interrupts and provider independence are requirements.

---

Back to [README.md](README.md).
