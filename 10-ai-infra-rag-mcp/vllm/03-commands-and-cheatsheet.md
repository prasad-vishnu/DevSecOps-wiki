# vLLM: Commands and Cheatsheet

| | |
|---|---|
| **Cluster** | 10 AI Infra & RAG/MCP |
| **Role** | Anchor |
| **Prerequisites** | [01-installation-and-setup.md](01-installation-and-setup.md), [02-core-concepts.md](02-core-concepts.md) |
| **Target version** | vLLM 0.10+ |
| **Capstone phase** | Phase 6 (Months 21-24) |

> Reference card for the `vllm` CLI, every flag that matters in production, the HTTP API with curl and Python examples, the offline API, environment variables, PromQL for the `/metrics` endpoint, and `kubectl` one-liners for a vLLM Deployment. Each entry has a one-line explanation. Long-form flags are shown; short forms are listed where they exist (`-tp`, `-pp`, `-q`, `-O`).

---

## 1. The `vllm` CLI

| Command | What it does |
|---|---|
| `vllm serve <model> [flags]` | Start the OpenAI-compatible server for a Hugging Face model id or local path |
| `vllm serve --config serve.yaml` | Read flags from a YAML file (keys are flag names without `--`) |
| `vllm chat --url http://localhost:8000/v1` | Interactive chat REPL against a running server |
| `vllm complete --url ...` | Interactive raw completion REPL |
| `vllm bench serve --model <m> --dataset-name sharegpt ...` | Online benchmark: TTFT, ITL, throughput at a request rate |
| `vllm bench throughput --model <m> --input-len 512 --output-len 128` | Offline throughput benchmark |
| `vllm bench latency --model <m> --batch-size 8` | Offline latency per batch |
| `vllm run-batch -i requests.jsonl -o results.jsonl --model <m>` | OpenAI Batch-API style offline job |
| `vllm collect-env` | Print environment for bug reports |
| `python -m vllm.entrypoints.openai.api_server [flags]` | Legacy spelling of `vllm serve` (same flags) |

```yaml
# serve.yaml  (vllm serve --config serve.yaml)
model: Qwen/Qwen2.5-7B-Instruct-AWQ
served-model-name: [qwen2.5-7b]
quantization: awq_marlin
dtype: half
max-model-len: 8192
max-num-seqs: 64
gpu-memory-utilization: 0.90
enable-prefix-caching: true
enable-auto-tool-choice: true
tool-call-parser: hermes
port: 8000
```

---

## 2. `vllm serve` Flags by Task

### 2.1 Model and identity

| Flag | Explanation |
|---|---|
| `<model>` / `--model` | HF repo id (`meta-llama/Llama-3.1-8B-Instruct`) or local directory |
| `--served-model-name NAME [NAME...]` | Name(s) clients use in `"model"`; decouples API from checkpoint path |
| `--revision`, `--tokenizer`, `--tokenizer-revision` | Pin a git revision; use a different tokenizer repo |
| `--tokenizer-mode auto\|slow\|mistral` | `mistral` for Mistral-format tokenizers (`tekken.json`) |
| `--trust-remote-code` | Allow model repo Python code (review it first; supply-chain risk) |
| `--download-dir PATH` | Where weights are cached (defaults to `HF_HOME`) |
| `--load-format auto\|safetensors\|pt\|gguf\|tensorizer\|runai_streamer` | Weight loading backend; `runai_streamer` streams from S3 |
| `--config-format auto\|hf\|mistral` | Config file format |
| `--task auto\|generate\|embed\|classify\|score\|reward` (older) / `--runner generate\|pooling` (newer) | Run a pooling model for embeddings/reranking instead of generation |

### 2.2 Memory and context

| Flag | Explanation |
|---|---|
| `--dtype auto\|half\|float16\|bfloat16\|float32` | Weight/activation dtype; `auto` follows `config.json` (BF16 needs Ampere+) |
| `--max-model-len N` | Context window served (prompt + completion); default from model config, often too large for your VRAM |
| `--gpu-memory-utilization 0.0-1.0` | Fraction of each GPU vLLM may use (default 0.9); everything after weights becomes KV cache |
| `--kv-cache-dtype auto\|fp8\|fp8_e4m3\|fp8_e5m2` | Quantise the KV cache; ~2x token capacity |
| `--block-size 8\|16\|32\|64\|128` | PagedAttention block size in tokens (16 default on CUDA) |
| `--swap-space GiB` | CPU swap per GPU for preempted sequences (V0 only; V1 recomputes) |
| `--cpu-offload-gb GiB` | Offload part of weights to CPU RAM (slow, lets a 13B fit a 16 GB card) |
| `--max-seq-len-to-capture N` | Longest sequence captured in CUDA graphs |
| `--enforce-eager` | Disable CUDA graphs: faster startup, lower memory, slower decode; use for debugging |

### 2.3 Scheduler and throughput

| Flag | Explanation |
|---|---|
| `--max-num-seqs N` | Max sequences per iteration (concurrency ceiling) |
| `--max-num-batched-tokens N` | Token budget per iteration; governs chunked prefill granularity |
| `--enable-chunked-prefill` / `--no-enable-chunked-prefill` | Split long prefills across iterations (default on in V1) |
| `--enable-prefix-caching` / `--no-enable-prefix-caching` | Automatic prefix caching (default on in V1) |
| `--prefix-caching-hash-algo builtin\|sha256` | Hash function; `sha256` for multi-tenant collision safety |
| `--scheduling-policy fcfs\|priority` | Enable per-request `priority` field |
| `--num-scheduler-steps N` | Multi-step scheduling (V0) |
| `--preemption-mode recompute\|swap` | What to do when evicting a sequence |
| `--disable-log-requests` | Stop logging every prompt (do this in production for PII) |

### 2.4 Quantisation and speculation

| Flag | Explanation |
|---|---|
| `--quantization, -q awq\|awq_marlin\|gptq\|gptq_marlin\|fp8\|bitsandbytes\|compressed-tensors\|gguf\|modelopt\|torchao` | Weight quantisation method; usually auto-detected |
| `--speculative-config '{...}'` | JSON: `{"model": ..., "num_speculative_tokens": 5}` or `{"method": "ngram", ...}` or `{"method": "eagle3", ...}` |
| `--speculative-model`, `--num-speculative-tokens` | Legacy spellings of the above |

### 2.5 Parallelism

| Flag | Explanation |
|---|---|
| `--tensor-parallel-size, -tp N` | Shard every layer across N GPUs on one node |
| `--pipeline-parallel-size, -pp N` | Split layers into N stages (across nodes) |
| `--data-parallel-size, -dp N` | Run N engine replicas inside one deployment (DeepSeek-style MoE serving) |
| `--enable-expert-parallel` | Shard MoE experts across GPUs |
| `--distributed-executor-backend ray\|mp\|uni\|external_launcher` | `mp` single node (default), `ray` multi-node |
| `--max-parallel-loading-workers N` | Limit concurrent weight loads to avoid CPU RAM spikes |

### 2.6 API server

| Flag | Explanation |
|---|---|
| `--host 0.0.0.0`, `--port 8000` | Bind address |
| `--api-key KEY` / env `VLLM_API_KEY` | Require `Authorization: Bearer KEY` (single shared key; put real auth in the gateway) |
| `--root-path /vllm` | When served behind a path-prefixing ingress |
| `--allowed-origins`, `--allowed-methods`, `--allowed-headers` | CORS |
| `--ssl-keyfile`, `--ssl-certfile`, `--ssl-ca-certs` | TLS termination in-process (prefer the mesh/ingress) |
| `--uvicorn-log-level info\|warning\|error` | HTTP log verbosity |
| `--max-log-len N` | Truncate logged prompts |
| `--response-role assistant` | Role name in chat responses |
| `--return-tokens-as-token-ids` | Debug tokenisation |
| `--enable-request-id-headers` | Echo `X-Request-Id` for tracing |
| `--middleware module.path:Class` | Inject ASGI middleware (auth, OTel) |
| `--disable-fastapi-docs` | Hide `/docs` in production |

### 2.7 Chat, tools, structured output

| Flag | Explanation |
|---|---|
| `--chat-template PATH_OR_STRING` | Override the model's Jinja chat template |
| `--chat-template-content-format auto\|string\|openai` | How multimodal content parts are rendered |
| `--enable-auto-tool-choice` | Allow `tool_choice: "auto"` |
| `--tool-call-parser hermes\|llama3_json\|llama4_pythonic\|mistral\|granite\|granite-20b-fc\|internlm\|jamba\|pythonic\|deepseek_v3\|xlam\|...` | Parse the model's tool-call syntax into OpenAI `tool_calls` |
| `--tool-parser-plugin PATH` | Custom parser |
| `--guided-decoding-backend auto\|xgrammar\|guidance\|outlines` | Structured-output engine |
| `--reasoning-parser deepseek_r1\|qwen3\|...` | Split `<think>` content into `reasoning_content` |
| `--enable-reasoning` | (older) enable reasoning parsing |

### 2.8 LoRA

| Flag | Explanation |
|---|---|
| `--enable-lora` | Turn on multi-LoRA serving |
| `--lora-modules name=/path/or/hf-id [name=path...]` | Adapters to load at start; each becomes a `model` name |
| `--max-loras N` | Adapters resident per batch |
| `--max-lora-rank 16\|32\|64\|128\|256` | Max rank supported |
| `--max-cpu-loras N` | Adapters cached in CPU RAM |
| `--lora-dtype`, `--fully-sharded-loras` | Precision and TP sharding of adapters |
| env `VLLM_ALLOW_RUNTIME_LORA_UPDATING=True` | Enable `POST /v1/load_lora_adapter` / `/v1/unload_lora_adapter` |

### 2.9 Multimodal

| Flag | Explanation |
|---|---|
| `--limit-mm-per-prompt '{"image": 4, "video": 1}'` | Max media items per request |
| `--mm-processor-kwargs '{...}'` | Processor options (e.g. image resolution) |
| `--allowed-local-media-path` | Permit `file://` URLs from this directory |

---

## 3. Environment Variables

| Variable | Explanation |
|---|---|
| `HF_TOKEN` | Hugging Face token for gated repos |
| `HF_HOME`, `HF_HUB_CACHE` | Cache location (mount a PVC here) |
| `HF_HUB_ENABLE_HF_TRANSFER=1` | Rust-based parallel downloader |
| `HF_HUB_OFFLINE=1` | Never hit the Hub (air-gapped) |
| `VLLM_API_KEY` | Same as `--api-key` |
| `VLLM_LOGGING_LEVEL=DEBUG` | Engine log level |
| `VLLM_USE_V1=1\|0` | Force V1/V0 engine (V1 is default) |
| `VLLM_ATTENTION_BACKEND=FLASH_ATTN\|FLASHINFER\|XFORMERS\|TRITON_ATTN` | Attention kernel override |
| `VLLM_CPU_KVCACHE_SPACE=GiB`, `VLLM_CPU_OMP_THREADS_BIND` | CPU backend settings |
| `CUDA_VISIBLE_DEVICES=0,1` | Pin GPUs (Kubernetes sets this via the device plugin) |
| `NCCL_DEBUG=INFO`, `NCCL_SOCKET_IFNAME=eth0`, `NCCL_IB_DISABLE=1` | NCCL diagnostics and network selection for multi-GPU |
| `VLLM_WORKER_MULTIPROC_METHOD=spawn` | Multiprocessing start method |
| `VLLM_RPC_TIMEOUT=ms` | Engine RPC timeout for slow startups |
| `VLLM_NO_USAGE_STATS=1` / `DO_NOT_TRACK=1` | Disable anonymous usage telemetry |
| `VLLM_ALLOW_LONG_MAX_MODEL_LEN=1` | Permit `--max-model-len` beyond the model's trained length (RoPE scaling) |

---

## 4. HTTP API Cheatsheet

Set once:

```bash
export VLLM_URL=http://localhost:8000
export MODEL=qwen2.5-7b
```

### 4.1 Models, health, metrics

```bash
curl -s $VLLM_URL/health -o /dev/null -w '%{http_code}\n'            # 200 when engine ready
curl -s $VLLM_URL/version                                             # {"version":"0.10.1"}
curl -s $VLLM_URL/v1/models | jq '.data[].id'                         # served names + LoRA adapters
curl -s $VLLM_URL/metrics | grep -E '^vllm:(num_requests|gpu_cache)'  # key gauges
```

### 4.2 Chat completion (non-streaming)

```bash
curl -s $VLLM_URL/v1/chat/completions -H 'Content-Type: application/json' -d @- <<EOF | jq -r '.choices[0].message.content'
{"model":"$MODEL","temperature":0.2,"max_tokens":120,
 "messages":[{"role":"system","content":"You are a Kubernetes SRE."},
             {"role":"user","content":"Explain OOMKilled in two sentences."}]}
EOF
```

### 4.3 Streaming (SSE)

```bash
curl -N -s $VLLM_URL/v1/chat/completions -H 'Content-Type: application/json' -d @- <<EOF
{"model":"$MODEL","stream":true,"stream_options":{"include_usage":true},
 "messages":[{"role":"user","content":"Count to five."}]}
EOF
```

Expected output (abridged):

```text
data: {"id":"chatcmpl-...","choices":[{"index":0,"delta":{"role":"assistant","content":""}}]}
data: {"id":"chatcmpl-...","choices":[{"index":0,"delta":{"content":"1"}}]}
data: {"id":"chatcmpl-...","choices":[{"index":0,"delta":{"content":", 2"}}]}
...
data: {"id":"chatcmpl-...","choices":[],"usage":{"prompt_tokens":13,"completion_tokens":11,"total_tokens":24}}
data: [DONE]
```

### 4.4 Tool calling

```bash
curl -s $VLLM_URL/v1/chat/completions -H 'Content-Type: application/json' -d @- <<EOF | jq '.choices[0].message.tool_calls'
{"model":"$MODEL","tool_choice":"auto",
 "messages":[{"role":"user","content":"What is the 5xx rate for service api in the last 5 minutes?"}],
 "tools":[{"type":"function","function":{"name":"prometheus_query",
   "description":"Run an instant PromQL query",
   "parameters":{"type":"object","properties":{"query":{"type":"string"}},"required":["query"]}}}]}
EOF
```

Expected output:

```json
[
  {
    "id": "chatcmpl-tool-9c1e...",
    "type": "function",
    "function": {
      "name": "prometheus_query",
      "arguments": "{\"query\": \"sum(rate(http_requests_total{service=\\\"api\\\",status=~\\\"5..\\\"}[5m]))\"}"
    }
  }
]
```

### 4.5 Structured output

```bash
curl -s $VLLM_URL/v1/chat/completions -H 'Content-Type: application/json' -d @- <<EOF | jq -r '.choices[0].message.content'
{"model":"$MODEL","messages":[{"role":"user","content":"Classify severity of: 'checkout latency p99 is 9s'"}],
 "extra_body":{},
 "response_format":{"type":"json_schema","json_schema":{"name":"sev","schema":{
   "type":"object","properties":{"severity":{"type":"string","enum":["SEV1","SEV2","SEV3"]},
   "reason":{"type":"string","maxLength":120}},"required":["severity","reason"],"additionalProperties":false}}}}
EOF
```

Expected output:

```json
{"severity": "SEV2", "reason": "Customer-facing latency far above SLO but service is still available."}
```

vLLM-native variants: `"guided_json": {...}`, `"guided_choice": ["SEV1","SEV2","SEV3"]`, `"guided_regex": "SEV[1-3]"`, `"guided_grammar": "<EBNF>"`.

### 4.6 Completions, embeddings, tokenize, rerank

```bash
# Raw completion with logprobs
curl -s $VLLM_URL/v1/completions -H 'Content-Type: application/json' \
  -d "{\"model\":\"$MODEL\",\"prompt\":\"The capital of France is\",\"max_tokens\":3,\"logprobs\":2}" | jq '.choices[0]'

# Embeddings (server started with an embedding model, e.g. vllm serve BAAI/bge-m3 --task embed)
curl -s $VLLM_URL/v1/embeddings -H 'Content-Type: application/json' \
  -d '{"model":"bge-m3","input":["restart the pod","delete the deployment"]}' | jq '.data[0].embedding | length'
# -> 1024

# Tokenize
curl -s $VLLM_URL/tokenize -H 'Content-Type: application/json' \
  -d "{\"model\":\"$MODEL\",\"prompt\":\"kubectl rollout undo\"}" | jq '.count'

# Rerank (vllm serve BAAI/bge-reranker-v2-m3 --task score)
curl -s $VLLM_URL/v1/rerank -H 'Content-Type: application/json' \
  -d '{"model":"bge-reranker-v2-m3","query":"pod crashlooping","documents":["Runbook: CrashLoopBackOff","Runbook: DNS outage"]}' | jq '.results[] | {index, relevance_score}'
```

### 4.7 LoRA at runtime

```bash
curl -s -X POST $VLLM_URL/v1/load_lora_adapter -H 'Content-Type: application/json' \
  -d '{"lora_name":"runbook-style","lora_path":"/adapters/runbook-style"}'
curl -s -X POST $VLLM_URL/v1/unload_lora_adapter -H 'Content-Type: application/json' -d '{"lora_name":"runbook-style"}'
```

---

## 5. Python Snippets

### 5.1 OpenAI SDK with vLLM extras

```python
from openai import OpenAI
client = OpenAI(base_url="http://localhost:8000/v1", api_key=os.environ.get("VLLM_API_KEY", "x"))

# Streaming
with client.chat.completions.create(model="qwen2.5-7b", stream=True,
        messages=[{"role": "user", "content": "List three kubectl debugging commands."}]) as stream:
    for chunk in stream:
        print(chunk.choices[0].delta.content or "", end="", flush=True)

# vLLM-specific sampling and guided decoding via extra_body
resp = client.chat.completions.create(
    model="qwen2.5-7b",
    messages=[{"role": "user", "content": "Pick one: restart, rollback, scale"}],
    extra_body={"guided_choice": ["restart", "rollback", "scale"], "top_k": 20, "repetition_penalty": 1.05},
)
print(resp.choices[0].message.content)   # -> rollback
```

### 5.2 Offline API

```python
from vllm import LLM, SamplingParams
from vllm.sampling_params import GuidedDecodingParams

llm = LLM("Qwen/Qwen2.5-0.5B-Instruct", max_model_len=2048)
sp = SamplingParams(temperature=0, max_tokens=32,
                    guided_decoding=GuidedDecodingParams(choice=["yes", "no"]))
out = llm.chat([{"role": "user", "content": "Is 'ImagePullBackOff' a scheduling error?"}], sp)
print(out[0].outputs[0].text)   # -> no
```

### 5.3 LangChain / Strands pointing at vLLM

```python
# LangChain
from langchain_openai import ChatOpenAI
llm = ChatOpenAI(base_url="http://vllm.llm.svc:8000/v1", api_key="x", model="qwen2.5-7b", temperature=0)

# Strands Agents (OpenAI-compatible provider)
from strands.models.openai import OpenAIModel
model = OpenAIModel(client_args={"base_url": "http://vllm.llm.svc:8000/v1", "api_key": "x"}, model_id="qwen2.5-7b")
```

---

## 6. PromQL for `/metrics`

```promql
# Queue depth per model (autoscaling signal)
sum by (model_name) (vllm:num_requests_waiting)

# Running sequences vs configured max-num-seqs
sum by (model_name) (vllm:num_requests_running)

# KV cache saturation (alert > 0.9 for 5m)
max by (model_name) (vllm:gpu_cache_usage_perc)

# Preemption rate (should be ~0)
sum by (model_name) (rate(vllm:num_preemptions_total[5m]))

# Tokens per second, prompt and generation
sum by (model_name) (rate(vllm:prompt_tokens_total[1m]))
sum by (model_name) (rate(vllm:generation_tokens_total[1m]))

# TTFT p95 (SLO < 0.8s)
histogram_quantile(0.95, sum by (le, model_name) (rate(vllm:time_to_first_token_seconds_bucket[5m])))

# Inter-token latency p95
histogram_quantile(0.95, sum by (le, model_name) (rate(vllm:time_per_output_token_seconds_bucket[5m])))

# End-to-end p99
histogram_quantile(0.99, sum by (le) (rate(vllm:e2e_request_latency_seconds_bucket[5m])))

# Prefix cache hit rate
sum(rate(vllm:prefix_cache_hits_total[5m])) / sum(rate(vllm:prefix_cache_queries_total[5m]))

# Fraction of requests truncated by max_tokens (quality signal)
sum(rate(vllm:request_success_total{finished_reason="length"}[1h])) / sum(rate(vllm:request_success_total[1h]))

# Requests per second
sum(rate(vllm:request_success_total[1m]))

# Average tokens per scheduler iteration (batching efficiency)
rate(vllm:iteration_tokens_total_sum[5m]) / rate(vllm:iteration_tokens_total_count[5m])
```

---

## 7. `vllm bench` Quick Reference

```bash
# Online: 200 requests at 4 req/s using ShareGPT prompts
vllm bench serve --backend openai-chat --base-url http://localhost:8000 --endpoint /v1/chat/completions \
  --model qwen2.5-7b --tokenizer Qwen/Qwen2.5-7B-Instruct \
  --dataset-name sharegpt --dataset-path ShareGPT_V3_unfiltered_cleaned_split.json \
  --num-prompts 200 --request-rate 4 --save-result --result-filename bench.json

# Online with synthetic lengths (RAG-like: 3000 in, 400 out), fixed concurrency 20
vllm bench serve --backend openai-chat --base-url http://localhost:8000 --endpoint /v1/chat/completions \
  --model qwen2.5-7b --dataset-name random --random-input-len 3000 --random-output-len 400 \
  --num-prompts 100 --max-concurrency 20 --request-rate inf

# Offline throughput
vllm bench throughput --model Qwen/Qwen2.5-7B-Instruct-AWQ --input-len 1024 --output-len 256 --num-prompts 500
```

Key output fields: `Request throughput (req/s)`, `Output token throughput (tok/s)`, `Mean/Median/P99 TTFT (ms)`, `Mean/P99 TPOT (ms)`, `Mean/P99 ITL (ms)`.

---

## 8. kubectl One-Liners for a vLLM Deployment

```bash
NS=llm
kubectl -n $NS get pods -l app=vllm -o wide                                       # which GPU node
kubectl -n $NS logs deploy/vllm --tail=50 | grep -E 'KV cache size|concurrency'   # capacity lines
kubectl -n $NS exec deploy/vllm -- nvidia-smi --query-gpu=memory.used,utilization.gpu --format=csv
kubectl -n $NS exec deploy/vllm -- curl -s localhost:8000/metrics | grep -E '^vllm:num_requests'
kubectl -n $NS port-forward svc/vllm 8000:8000
kubectl -n $NS describe pod -l app=vllm | grep -A5 Events                         # scheduling failures
kubectl -n $NS rollout restart deploy/vllm                                         # reload weights/flags
kubectl -n $NS get hpa,scaledobject                                                # autoscaling state
kubectl -n $NS top pod -l app=vllm                                                 # CPU/RAM (not GPU)
kubectl get nodes -L nvidia.com/gpu.product,karpenter.sh/capacity-type             # GPU node inventory
```

---

## 9. Quick Decision Table: Which Flag Fixes Which Symptom

| Symptom | Flag to adjust |
|---|---|
| `CUDA out of memory` at startup | lower `--max-model-len`, lower `--gpu-memory-utilization`, quantise, add `-tp` |
| `GPU KV cache size` too small / low concurrency | quantise weights, `--kv-cache-dtype fp8`, lower `--max-model-len` |
| High `vllm:num_preemptions_total` | lower `--max-num-seqs`, raise KV capacity |
| TTFT spikes on long prompts | ensure chunked prefill on, lower `--max-num-batched-tokens` |
| ITL spikes | lower `--max-num-batched-tokens`, lower `--max-num-seqs` |
| Low throughput, low GPU util | raise `--max-num-seqs`, raise `--max-num-batched-tokens`, add load |
| Startup takes minutes | `HF_HUB_ENABLE_HF_TRANSFER=1`, PVC cache, `--load-format safetensors`, `--enforce-eager` while debugging |
| Tool calls returned as plain text | `--enable-auto-tool-choice --tool-call-parser <matching>` |
| Thinking tokens pollute content | `--reasoning-parser <model-family>` |
