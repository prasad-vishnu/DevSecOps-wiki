# vLLM: Interview Questions

| | |
|---|---|
| **Cluster** | 10 AI Infra & RAG/MCP |
| **Role** | Anchor |
| **Prerequisites** | Whole folder |
| **Target version** | vLLM 0.10+ |
| **Capstone phase** | Phase 6 (Months 21-24) |

> Thirty questions from junior to principal level, each with a model answer. Junior questions check vocabulary; mid-level questions check that you can operate the engine; senior questions check capacity planning and failure handling; principal questions check architecture and economic judgement.

---

## Junior

### Q1. What problem does vLLM solve? (Junior)

**Answer.** It serves large language models efficiently on GPUs. Naive serving wastes most GPU memory on fragmented, over-reserved KV cache and most GPU compute on static batches. vLLM's PagedAttention manages the KV cache in small blocks like virtual memory pages, and continuous batching re-plans the batch every decode step, giving several times the throughput of Hugging Face `generate()` at the same latency, behind an OpenAI-compatible API.

### Q2. What is the KV cache? (Junior)

**Answer.** During attention, every token's key and value projections for every layer are stored so later tokens can attend to them without recomputation. Its size grows linearly with sequence length and is `2 x layers x kv_heads x head_dim x bytes` per token: about 128 KiB per token for Llama-3.1-8B in FP16. It is the dominant consumer of GPU memory after the weights.

### Q3. What does `--max-model-len` control? (Junior)

**Answer.** The maximum context (prompt plus completion) a single request may use. vLLM must be able to fit at least one request of that length in the KV cache, so a large value with little free memory fails at startup. It is also a safety cap against huge prompts.

### Q4. Which endpoint does a client call for chat, and why does that matter? (Junior)

**Answer.** `POST /v1/chat/completions`, the OpenAI schema. Because vLLM implements the OpenAI protocol, the official OpenAI SDK, LangChain, Strands, LiteLLM and most tooling work by changing only `base_url` and `model`.

### Q5. What is the difference between prefill and decode? (Junior)

**Answer.** Prefill processes the whole prompt in one compute-bound pass and writes the KV cache; it determines time-to-first-token. Decode generates one token per pass, reading all weights and the cache each time; it is memory-bandwidth-bound and determines inter-token latency.

### Q6. Name three ways to make a model fit a smaller GPU. (Junior)

**Answer.** Quantise the weights (AWQ/GPTQ 4-bit, FP8), reduce `--max-model-len` so less KV cache is needed, and shard across GPUs with `--tensor-parallel-size`. `--cpu-offload-gb` and `--kv-cache-dtype fp8` are further options.

### Q7. What does `vllm:num_requests_waiting` tell you? (Junior)

**Answer.** How many requests are queued because the scheduler could not admit them (concurrency or KV cache limits). Sustained non-zero values mean you are saturated and TTFT is rising; it is the primary autoscaling signal.

---

## Mid-level

### Q8. Explain PagedAttention with the OS analogy. (Mid-level)

**Answer.** The KV cache is divided into fixed blocks (16 tokens). Each sequence has a block table mapping logical to physical blocks, like a page table. Blocks are allocated on demand and can live anywhere in GPU memory, so there is no contiguous reservation and waste is under one block per sequence. Shared prefixes map to the same physical blocks (copy-on-write), which is what makes prefix caching and parallel sampling cheap. A custom attention kernel gathers blocks through the table.

### Q9. What is continuous batching and why does it beat static batching? (Mid-level)

**Answer.** Static batching waits until every sequence in a batch finishes, so short requests idle while long ones run and new requests wait. Continuous (iteration-level) batching re-forms the batch at every forward pass: finished sequences leave, waiting ones join as soon as their blocks fit. GPU utilisation stays high and latency for short requests drops dramatically.

### Q10. What does chunked prefill fix? (Mid-level)

**Answer.** Without it, a very long prompt occupies an entire iteration, stalling every decoding sequence and spiking inter-token latency. Chunked prefill splits the prompt across iterations within the `--max-num-batched-tokens` budget and mixes prefill chunks with decode tokens. The long request's TTFT gets slightly worse; everyone else's ITL gets much better.

### Q11. How does automatic prefix caching work and how do you maximise hits? (Mid-level)

**Answer.** Each full block is hashed by its content and the hash of the preceding prefix. New requests whose prompt starts with already-resident blocks skip prefill for those tokens. To maximise hits, place stable content (system prompt, few-shot examples, shared documents) first and variable content last, because caching is strictly prefix-based. Monitor `vllm:prefix_cache_hits_total / vllm:prefix_cache_queries_total`.

### Q12. Compare AWQ, GPTQ, FP8 and bitsandbytes. (Mid-level)

**Answer.** AWQ and GPTQ are weight-only 4-bit methods requiring a calibrated checkpoint; they cut memory ~4x and speed decode (less bandwidth), with Marlin kernels on Ampere+. FP8 (W8A8) halves memory and is near-lossless; on Hopper/Ada it also speeds compute, on Ampere it is emulated and slower. bitsandbytes quantises on the fly without a checkpoint, convenient but with slower kernels. Choose AWQ/GPTQ for A10G/A100, FP8 for H100/L4.

### Q13. How do you make a model emit OpenAI-style tool calls with vLLM? (Mid-level)

**Answer.** Start with `--enable-auto-tool-choice --tool-call-parser <family>` where the parser matches the model's native format (`hermes` for Qwen2.5, `llama3_json` for Llama 3.x, `mistral`, `deepseek_v3`). The server then parses the model's text into `tool_calls`. The client sends `tools` and `tool_choice: "auto"` and returns tool results as `role: tool` messages.

### Q14. How is structured output guaranteed? (Mid-level)

**Answer.** Guided decoding compiles the JSON schema, regex or grammar into a finite-state machine over tokens (xgrammar, llguidance or Outlines). At each decode step, logits for tokens that would violate the grammar are masked, so the output cannot be invalid. The first request per schema pays a compile cost; subsequent ones hit a cache.

### Q15. Why is `/health` insufficient as the only probe? (Mid-level)

**Answer.** It confirms the API server and engine process are alive, not that the GPU is making progress; a hung CUDA context can keep it green. Pair it with TTFT and queue alerts and a synthetic canary that sends a tiny prompt periodically.

### Q16. How do you deploy a LoRA-fine-tuned variant without a second GPU? (Mid-level)

**Answer.** `--enable-lora --lora-modules name=path` loads adapters alongside the base model; each adapter is exposed as its own model name and requests for different adapters are batched together. Set `--max-loras` and `--max-lora-rank` to the adapters' needs, and optionally enable runtime loading with `VLLM_ALLOW_RUNTIME_LORA_UPDATING`.

---

## Senior

### Q17. Size the GPU for Llama-3.1-8B, 8k context, 30 concurrent requests, p95 TTFT under 800 ms. (Senior)

**Answer.** KV per token is 128 KiB; a full 8k request is 1 GiB; 30 concurrent is 30 GiB, more than any 24 GB card after 16 GB of FP16 weights. Options: AWQ 4-bit weights (~5.5 GB) on a 24 GB A10G leaves ~15 GB KV, around 15 full requests; realistic requests average far less than 8k, so vLLM's "maximum concurrency" understates real capacity, but 30 at p95 needs either FP8 KV cache (doubling capacity) or two replicas. I would run two `g5.xlarge` spot replicas with AWQ + FP8 KV, `--max-num-seqs 64`, chunked prefill at 8192 tokens, prefix caching, and validate with `vllm bench serve` at 30 and 45 concurrency before trusting the TTFT SLO. One H100 would also do it with headroom but costs 5-8x more.

### Q18. Your TTFT p95 doubled after a traffic increase but GPU utilisation is 55 %. Explain and fix. (Senior)

**Answer.** Queueing, not compute, is the bottleneck: `num_requests_waiting` is probably non-zero because `--max-num-seqs` or KV capacity caps admissions, or chunked prefill's token budget is making long prompts wait several iterations. Check waiting, cache usage and preemptions. If cache usage is near 1.0, add KV capacity (FP8 KV, shorter max-model-len, quantised weights) or replicas. If cache is fine but waiting is high, raise `--max-num-seqs`. If prefill time dominates, raise `--max-num-batched-tokens`. 55 % GPU utilisation means there is compute left; the scheduler just is not allowed to use it.

### Q19. When would you choose tensor parallelism over more replicas? (Senior)

**Answer.** Only when the model does not fit a single GPU's memory with acceptable quantisation, or when a single request's latency must be minimised and NVLink is available. TP adds an all-reduce per layer, couples failure domains and complicates scheduling (all GPUs in one pod). If the model fits, replicas scale throughput linearly and fail independently.

### Q20. Design the weight-distribution strategy for 10 replicas of a 70B model. (Senior)

**Answer.** Do not download from Hugging Face in production pods. Mirror the checkpoint in CI to an S3 bucket under a versioned prefix with SHA256 verification, then either stream directly with `--load-format runai_streamer` or pull with an `s5cmd` init container into a per-node cache. For many replicas per node, use a shared RWX volume (FSx for Lustre) or node-local NVMe with a DaemonSet pre-warmer. Serving pods run with `HF_HUB_OFFLINE=1` and IRSA for S3; no Hugging Face token exists in the serving namespace.

### Q21. How do you autoscale vLLM, and what goes wrong with the obvious approach? (Senior)

**Answer.** Scale on queue depth (`vllm:num_requests_waiting`) or KV cache usage via KEDA or the Prometheus Adapter, with slow scale-down (10 min) and max replicas bounded by GPU quota. CPU-based HPA never triggers because GPU workloads barely use CPU. Scale-up latency is minutes (node provision plus weight load), so stabilisation windows and headroom matter more than reactivity, and cold weight loading must be solved first or the autoscaler just adds pods stuck in startup.

### Q22. Compare vLLM, TGI, SGLang, TensorRT-LLM and Ollama. (Senior)

**Answer.** vLLM: broadest model and hardware support, fastest-moving community, strong throughput; the default choice. TGI: mature HF server, slower development recently. SGLang: RadixAttention gives excellent prefix sharing and a programming frontend; often leads on DeepSeek-style MoE. TensorRT-LLM (with Triton or NIM): highest raw NVIDIA performance via compiled engines, at the cost of engine builds per model/GPU and vendor lock-in. Ollama: llama.cpp wrapper for laptops and tiny traffic; no continuous batching at vLLM's level and limited metrics. For a Kubernetes platform team, vLLM first, SGLang if your workload is agentic with heavy prefix reuse, TensorRT-LLM/NIM when you have frozen models and need the last 20 %.

### Q23. What security controls do you put around a vLLM endpoint? (Senior)

**Answer.** ClusterIP only behind a gateway doing OIDC/JWT auth and per-client rate and token limits; `--api-key` from a secret store as defence in depth; NetworkPolicy admitting only the agent and monitoring namespaces; non-root restricted Pod Security; pinned image digests with scanning and signature verification; no `--trust-remote-code`; weights from a verified internal mirror in offline mode; request logging disabled and token-count-only telemetry via OTel; memory limits and body-size caps against resource exhaustion.

### Q24. How do you handle spot interruptions for GPU serving? (Senior)

**Answer.** Run at least two replicas across AZs with a PDB; let Karpenter handle the interruption notice by cordoning and draining; give the pod a 120 s grace period and a `preStop` sleep so the load balancer stops sending traffic before SIGTERM; make clients retry with backoff. Keep warm weights on a volume or S3 so the replacement pod starts in minutes, not tens of minutes. For strict SLOs, mix spot with an on-demand baseline replica.

---

## Principal

### Q25. Build the business case: self-host with vLLM or use Bedrock? (Principal)

**Answer.** Compute cost per million tokens for the self-hosted option as `hourly_price / (tokens_per_second x 3600 x utilisation / 1e6)` using benchmark throughput at the real traffic shape, then compare against the managed list price for a model of equivalent *task quality*, measured by your eval suite, not parameter count. On spot `g5`-class hardware a 7-8B model undercuts managed small-model pricing above roughly 25-40 % utilisation; at low or bursty utilisation managed wins. Add engineering and on-call cost, GPU quota lead time, data-residency requirements, and the option value of custom weights. The decision is an ADR with exit criteria: if utilisation falls below X for a quarter, migrate back.

### Q26. How would you evolve from bare vLLM Deployments to a fleet? (Principal)

**Answer.** Stage 1: Deployments per model behind a Gateway with KEDA, dashboards, evals in CI. Stage 2 (many replicas, shared prefixes): prefix-aware routing with the vLLM production stack router or the Gateway API Inference Extension so requests land where their KV cache lives; KV offload to CPU/remote with LMCache. Stage 3 (latency-critical at scale on Hopper): llm-d with disaggregated prefill and decode pools, KV transfer over fast interconnects, and scheduler-aware endpoint picking. Each stage is justified by measured queueing or cache-miss cost, not by novelty.

### Q27. A platform team wants a single internal LLM gateway for vLLM and Bedrock. What do you design? (Principal)

**Answer.** An OpenAI-compatible gateway (Envoy AI Gateway or LiteLLM) with virtual keys per team, routing by model name to vLLM pools or Bedrock, per-team token budgets and spend reporting, prompt/response size limits, OTel tracing with GenAI semantic conventions, and policy hooks for PII redaction and injection screening. Behind it, vLLM pools are a detail teams do not see, so capacity can move between self-hosted and managed without client changes. Evals run against the gateway so routing changes are tested.

### Q28. How do you reason about quantisation risk in a regulated workload? (Principal)

**Answer.** Treat quantisation as a model change: run the full eval suite (DeepEval/RAGAS plus task-specific golden sets) for each candidate (FP16 baseline, FP8, AWQ, FP8 KV cache), record deltas, and accept only within an agreed tolerance. Prefer calibrated FP8 checkpoints from the model vendor over ad hoc conversions. Pin the exact quantised artifact by hash and keep the FP16 baseline deployable for A/B comparisons when quality complaints arrive.

### Q29. What are the failure domains of a multi-node TP x PP deployment and how do you contain them? (Principal)

**Answer.** Any GPU, NIC, or host failure takes the whole model instance down because ranks are tightly synchronised through NCCL; a slow rank slows all. Contain by running whole instances per LeaderWorkerSet group, scheduling groups with gang semantics (Kueue/Volcano) so partial allocations never start, health-checking with timeouts that fail fast, and running at least two independent instances behind the gateway. Prefer the largest single node you can get (NVLink/NVSwitch) and PP only across nodes. Treat Xid errors as node-level incidents.

### Q30. What observability do you require before calling an LLM endpoint production-ready? (Principal)

**Answer.** Four layers: engine metrics (`vllm:*` for queue, cache, TTFT/ITL histograms, preemptions, tokens, finish reasons), GPU metrics (DCGM utilisation, memory, temperature, Xid), request traces from the agent with GenAI span attributes (model, token counts, latency, tool calls) correlated by request id through the gateway, and quality telemetry (sampled online eval scores, truncation rate, refusal rate). Dashboards express the SLOs, alerts are burn-rate based, and a cost panel converts tokens to dollars per team. If any layer is missing, incidents will be argued about rather than diagnosed.
