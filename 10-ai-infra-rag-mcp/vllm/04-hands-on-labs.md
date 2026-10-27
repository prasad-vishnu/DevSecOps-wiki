# vLLM: Hands-On Labs

| | |
|---|---|
| **Cluster** | 10 AI Infra & RAG/MCP |
| **Role** | Anchor |
| **Prerequisites** | [01-installation-and-setup.md](01-installation-and-setup.md); Labs 4+ need a GPU node (spot `g5.xlarge` or Colab T4) |
| **Target version** | vLLM 0.10+, Kubernetes 1.31+ |
| **Capstone phase** | Phase 6 (Months 21-24) |

> Eight progressive labs. Labs 1-3 run on a laptop CPU for free and teach the API, streaming, tool calling, structured output and metrics. Labs 4-8 use a single spot GPU (A10G/L4/T4) and build the capstone's production deployment: Kubernetes manifests, Prometheus scraping with a TTFT SLO, autoscaling on queue depth, LoRA adapters, multi-GPU, and a benchmark that yields a cost-per-million-tokens number for your ADR. Every GPU lab ends with teardown.

Cost guardrail: before any GPU lab, confirm the nightly teardown workflow from [05-production-patterns.md](05-production-patterns.md) is enabled, and set a 3-hour timer.

---

## Lab 1: CPU Serving and the OpenAI API

**Objective.** Serve `Qwen/Qwen2.5-0.5B-Instruct` on CPU, exercise every core endpoint, read the startup log.

**Prerequisites.** Docker Desktop or Linux with Docker; 8 GB RAM free.

**Time.** 40 minutes.

### Steps

1. Start the CPU server (build the image if the public tag is unavailable; see [01-installation-and-setup.md](01-installation-and-setup.md) section 2.3).

   ```bash
   mkdir -p ~/vllm-labs && cd ~/vllm-labs
   docker run -d --name vllm-cpu -p 8000:8000 --shm-size=4g \
     -e VLLM_CPU_KVCACHE_SPACE=4 \
     -v "$HOME/.cache/huggingface:/root/.cache/huggingface" \
     vllm-cpu-env --model Qwen/Qwen2.5-0.5B-Instruct --dtype bfloat16 \
     --max-model-len 4096 --served-model-name qwen-0.5b \
     --enable-auto-tool-choice --tool-call-parser hermes
   docker logs -f vllm-cpu
   ```

   Expected output (wait for):

   ```text
   INFO ... # CPU blocks: 8192
   INFO ... Maximum concurrency for 4,096 tokens per request: 32.00x
   INFO ... Application startup complete.
   ```

2. Query the endpoints.

   ```bash
   curl -s localhost:8000/v1/models | jq '.data[0] | {id, max_model_len}'
   curl -s localhost:8000/v1/chat/completions -H 'Content-Type: application/json' -d '{
     "model":"qwen-0.5b","max_tokens":60,"temperature":0,
     "messages":[{"role":"user","content":"What is a Kubernetes Service of type ClusterIP?"}]}' | jq -r '.choices[0].message.content'
   ```

3. Observe tokens-per-second on CPU.

   ```bash
   time curl -s localhost:8000/v1/chat/completions -H 'Content-Type: application/json' -d '{
     "model":"qwen-0.5b","max_tokens":200,"messages":[{"role":"user","content":"Write a haiku about etcd."}]}' | jq '.usage'
   ```

   Expected output: `completion_tokens` around 30-200 and wall time of a few seconds (roughly 5-20 tokens/s on a modern laptop CPU).

4. Stream and watch SSE frames arrive one token at a time.

   ```bash
   curl -N -s localhost:8000/v1/chat/completions -H 'Content-Type: application/json' -d '{
     "model":"qwen-0.5b","stream":true,"messages":[{"role":"user","content":"Count to ten."}]}' | head -20
   ```

### Verify

`curl -s localhost:8000/metrics | grep -c '^vllm:'` returns more than 30 lines, and `vllm:request_success_total` has incremented by your request count.

### Cleanup

Leave the container running for Labs 2 and 3.

### What you learned

The server contract is OpenAI's; the engine prints its capacity at startup; CPU is fine for API development but not for throughput.

---

## Lab 2: Tool Calling and Structured Output

**Objective.** Make the model emit a Prometheus tool call and a schema-constrained JSON object, as the capstone agent will.

**Prerequisites.** Lab 1 running.

**Time.** 40 minutes.

### Steps

1. Create `lab2_tools.py`.

   ```python
   import json
   from openai import OpenAI

   client = OpenAI(base_url="http://localhost:8000/v1", api_key="x")
   MODEL = "qwen-0.5b"

   tools = [{
       "type": "function",
       "function": {
           "name": "prometheus_query",
           "description": "Run an instant PromQL query and return the result.",
           "parameters": {"type": "object",
                          "properties": {"query": {"type": "string", "description": "PromQL"}},
                          "required": ["query"]},
       },
   }]

   messages = [
       {"role": "system", "content": "You are an SRE agent. Use tools when metrics are needed."},
       {"role": "user", "content": "How many pods are not ready in namespace prod right now?"},
   ]
   first = client.chat.completions.create(model=MODEL, messages=messages, tools=tools, tool_choice="auto", temperature=0)
   call = first.choices[0].message.tool_calls[0]
   print("tool:", call.function.name, "args:", call.function.arguments)

   # Simulate tool execution and send the result back
   messages.append(first.choices[0].message)
   messages.append({"role": "tool", "tool_call_id": call.id, "content": json.dumps({"result": 3})})
   second = client.chat.completions.create(model=MODEL, messages=messages, temperature=0)
   print("answer:", second.choices[0].message.content)
   ```

   ```bash
   python lab2_tools.py
   ```

   Expected output (small models vary; the shape is what matters):

   ```text
   tool: prometheus_query args: {"query": "count(kube_pod_status_ready{namespace=\"prod\",condition=\"false\"})"}
   answer: There are currently 3 pods not ready in the prod namespace.
   ```

2. Add structured output to `lab2_struct.py`.

   ```python
   from openai import OpenAI
   client = OpenAI(base_url="http://localhost:8000/v1", api_key="x")

   schema = {
       "type": "object",
       "properties": {
           "severity": {"type": "string", "enum": ["SEV1", "SEV2", "SEV3"]},
           "component": {"type": "string"},
           "action": {"type": "string", "enum": ["restart", "rollback", "scale", "investigate"]},
       },
       "required": ["severity", "component", "action"],
       "additionalProperties": False,
   }
   resp = client.chat.completions.create(
       model="qwen-0.5b", temperature=0,
       messages=[{"role": "user", "content": "Alert: payment-api error rate 40% since deploy v2.3.1 ten minutes ago."}],
       response_format={"type": "json_schema", "json_schema": {"name": "triage", "schema": schema}},
   )
   print(resp.choices[0].message.content)
   ```

   Expected output:

   ```json
   {"severity": "SEV1", "component": "payment-api", "action": "rollback"}
   ```

3. Prove the constraint: change `"enum"` for `action` to `["dance"]` and re-run. The model is forced to output `"dance"`; guided decoding masks every other token.

### Verify

`python -c "import json; json.loads(open('/dev/stdin').read())" <<< "$(python lab2_struct.py)"` exits 0.

### Cleanup

None.

### What you learned

Tool-call parsing is a server-side flag matched to the model family; structured output is a hard guarantee from token masking, not a prompt request.

---

## Lab 3: Metrics into Prometheus and Grafana (Local)

**Objective.** Scrape vLLM's `/metrics` with Prometheus, build a TTFT panel, and generate load.

**Prerequisites.** Lab 1 running; Docker Compose.

**Time.** 45 minutes.

### Steps

1. Create `prometheus.yml` and `compose.yaml`.

   ```yaml
   # prometheus.yml
   global: {scrape_interval: 5s}
   scrape_configs:
     - job_name: vllm
       static_configs:
         - targets: ["host.docker.internal:8000"]
   ```

   ```yaml
   # compose.yaml
   services:
     prometheus:
       image: prom/prometheus:v3.5.0
       ports: ["9090:9090"]
       volumes: ["./prometheus.yml:/etc/prometheus/prometheus.yml:ro"]
       extra_hosts: ["host.docker.internal:host-gateway"]
     grafana:
       image: grafana/grafana:12.1.0
       ports: ["3000:3000"]
       environment:
         GF_AUTH_ANONYMOUS_ENABLED: "true"
         GF_AUTH_ANONYMOUS_ORG_ROLE: Admin   # lab only; never in production
   ```

   ```bash
   docker compose up -d
   ```

2. Generate load with a small script.

   ```python
   # load.py
   import asyncio, time
   from openai import AsyncOpenAI
   client = AsyncOpenAI(base_url="http://localhost:8000/v1", api_key="x")

   async def one(i):
       t0 = time.perf_counter()
       stream = await client.chat.completions.create(model="qwen-0.5b", stream=True, max_tokens=40,
           messages=[{"role": "user", "content": f"Give one fact about Kubernetes concept number {i}."}])
       first = None
       async for chunk in stream:
           if first is None and chunk.choices and chunk.choices[0].delta.content:
               first = time.perf_counter() - t0
       return first

   async def main():
       ttfts = await asyncio.gather(*(one(i) for i in range(16)))
       print("client-side TTFT p50/max:", sorted(ttfts)[8], max(ttfts))
   asyncio.run(main())
   ```

   ```bash
   python load.py
   ```

3. In Prometheus (http://localhost:9090) run:

   ```promql
   histogram_quantile(0.95, sum by (le) (rate(vllm:time_to_first_token_seconds_bucket[1m])))
   ```

   and

   ```promql
   sum(vllm:num_requests_running) + sum(vllm:num_requests_waiting)
   ```

4. In Grafana add Prometheus (`http://prometheus:9090`) as a data source and create panels for TTFT p95, running/waiting, and `rate(vllm:generation_tokens_total[1m])`. Add a threshold line at 0.8 s on TTFT.

### Verify

During `python load.py`, `vllm:num_requests_waiting` rises above 0 (CPU can only batch a few sequences) and TTFT p95 climbs. This is the queueing behaviour you will autoscale on in Lab 6.

### Cleanup

```bash
docker compose down
docker rm -f vllm-cpu
```

### What you learned

TTFT is mostly queueing; the metrics to watch are waiting count and cache usage; a histogram quantile over `_bucket` is how you build an SLO.

---

## Lab 4: Quantised 7B on a Single GPU

**Objective.** Serve `Qwen/Qwen2.5-7B-Instruct-AWQ` on a 24 GB GPU, read KV capacity, test context-length limits, compare with `--kv-cache-dtype fp8`.

**Prerequisites.** A GPU host with Docker and NVIDIA Container Toolkit ([01-installation-and-setup.md](01-installation-and-setup.md) section 4), or a Colab T4 with `pip install vllm`. On a T4 (16 GB, no BF16) use `--dtype half`.

**Time.** 60 minutes. Cost: ~$0.40 on a spot `g5.xlarge`.

### Steps

1. Run with a deliberately large context.

   ```bash
   docker run -d --name vllm --runtime nvidia --gpus all --ipc=host -p 8000:8000 \
     -v "$HOME/.cache/huggingface:/root/.cache/huggingface" -e HF_HUB_ENABLE_HF_TRANSFER=1 \
     vllm/vllm-openai:v0.10.1 --model Qwen/Qwen2.5-7B-Instruct-AWQ --quantization awq_marlin --dtype half \
     --max-model-len 32768 --gpu-memory-utilization 0.90 --served-model-name qwen2.5-7b
   docker logs -f vllm 2>&1 | grep -E 'Model loading took|KV cache size|concurrency|startup complete'
   ```

   Expected output:

   ```text
   INFO ... Model loading took 5.21 GiB and 38.90 seconds
   INFO ... GPU KV cache size: 262,144 tokens
   INFO ... Maximum concurrency for 32,768 tokens per request: 8.00x
   INFO ... Application startup complete.
   ```

   Compute by hand: 21.6 GB - 5.2 GB weights - ~1 GB = ~15 GB; / 56 KiB per token = ~270k tokens. Matches.

2. Restart with FP8 KV cache and compare.

   ```bash
   docker rm -f vllm
   docker run -d --name vllm --runtime nvidia --gpus all --ipc=host -p 8000:8000 \
     -v "$HOME/.cache/huggingface:/root/.cache/huggingface" \
     vllm/vllm-openai:v0.10.1 --model Qwen/Qwen2.5-7B-Instruct-AWQ --quantization awq_marlin --dtype half \
     --max-model-len 32768 --gpu-memory-utilization 0.90 --kv-cache-dtype fp8 --served-model-name qwen2.5-7b
   docker logs vllm 2>&1 | grep 'KV cache size'
   ```

   Expected output: roughly double the tokens (`~524,000`).

3. Send a 20k-token prompt and watch `nvidia-smi` while it prefills.

   ```bash
   python - <<'EOF'
   from openai import OpenAI
   c = OpenAI(base_url="http://localhost:8000/v1", api_key="x")
   big = "kubectl get pods\n" * 2500
   r = c.chat.completions.create(model="qwen2.5-7b", max_tokens=20,
       messages=[{"role":"user","content": big + "\nHow many lines above?"}])
   print(r.usage)
   EOF
   watch -n 0.5 nvidia-smi --query-gpu=utilization.gpu,memory.used --format=csv
   ```

4. Trigger the context limit on purpose.

   ```bash
   curl -s localhost:8000/v1/chat/completions -H 'Content-Type: application/json' -d "{\"model\":\"qwen2.5-7b\",\"max_tokens\":40000,\"messages\":[{\"role\":\"user\",\"content\":\"hi\"}]}" | jq .error.message
   ```

   Expected output:

   ```text
   "This model's maximum context length is 32768 tokens. However, you requested 40008 tokens (8 in the messages, 40000 in the completion). Please reduce the length of the messages or completion."
   ```

### Verify

`curl -s localhost:8000/metrics | grep vllm:gpu_cache_usage_perc` shows a small non-zero value right after step 3 (blocks retained by prefix caching).

### Cleanup

`docker rm -f vllm`. Stop the instance if you are done for the day.

### What you learned

Quantisation buys concurrency; the engine's startup line is ground truth; FP8 KV cache doubles tokens; context errors are returned, not crashes.

---

## Lab 5: Production Deployment on Kubernetes

**Objective.** Deploy vLLM to a GPU node with probes, PVC model cache, PDB, NetworkPolicy, API key from a Secret, and a ServiceMonitor.

**Prerequisites.** Cluster with GPU Operator and a Karpenter GPU NodePool ([../gpu-workloads-kubernetes/04-hands-on-labs.md](../gpu-workloads-kubernetes/04-hands-on-labs.md) Lab 4); kube-prometheus-stack installed.

**Time.** 75 minutes. Cost: ~$0.50.

### Steps

1. Namespace, secret, PVC.

   ```bash
   kubectl create namespace llm
   kubectl label namespace llm pod-security.kubernetes.io/enforce=restricted
   # Lab shortcut: generated key. In the capstone this comes from External Secrets.
   kubectl -n llm create secret generic vllm-api --from-literal=VLLM_API_KEY="$(openssl rand -hex 24)"
   kubectl -n llm create secret generic hf-token --from-literal=HF_TOKEN="${HF_TOKEN:?set HF_TOKEN in your shell}"
   ```

2. Apply the deployment bundle `vllm-prod.yaml`.

   ```yaml
   apiVersion: v1
   kind: PersistentVolumeClaim
   metadata: {name: hf-cache, namespace: llm}
   spec:
     accessModes: [ReadWriteOnce]
     storageClassName: gp3
     resources: {requests: {storage: 60Gi}}
   ---
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: vllm
     namespace: llm
     labels: {app: vllm}
   spec:
     replicas: 1
     strategy:
       type: RollingUpdate
       rollingUpdate: {maxSurge: 1, maxUnavailable: 0}   # new pod must be Ready before old is killed
     selector: {matchLabels: {app: vllm}}
     template:
       metadata:
         labels: {app: vllm}
       spec:
         runtimeClassName: nvidia
         terminationGracePeriodSeconds: 120
         nodeSelector: {karpenter.sh/nodepool: gpu-spot}
         tolerations:
           - {key: nvidia.com/gpu, operator: Exists, effect: NoSchedule}
         securityContext:
           runAsNonRoot: true
           runAsUser: 1000
           runAsGroup: 1000
           fsGroup: 1000
           seccompProfile: {type: RuntimeDefault}
         containers:
           - name: vllm
             image: vllm/vllm-openai:v0.10.1
             securityContext:
               allowPrivilegeEscalation: false
               capabilities: {drop: [ALL]}
             args:
               - --model=Qwen/Qwen2.5-7B-Instruct-AWQ
               - --quantization=awq_marlin
               - --dtype=half
               - --max-model-len=8192
               - --max-num-seqs=64
               - --gpu-memory-utilization=0.90
               - --kv-cache-dtype=fp8
               - --enable-prefix-caching
               - --enable-auto-tool-choice
               - --tool-call-parser=hermes
               - --served-model-name=qwen2.5-7b
               - --disable-log-requests
               - --disable-fastapi-docs
               - --port=8000
             env:
               - {name: HF_HOME, value: /data/hf}
               - {name: HF_HUB_ENABLE_HF_TRANSFER, value: "1"}
               - {name: VLLM_API_KEY, valueFrom: {secretKeyRef: {name: vllm-api, key: VLLM_API_KEY}}}
               - {name: HF_TOKEN, valueFrom: {secretKeyRef: {name: hf-token, key: HF_TOKEN}}}
             ports: [{containerPort: 8000, name: http}]
             resources:
               requests: {cpu: "3", memory: 20Gi, nvidia.com/gpu: 1}
               limits:   {cpu: "4", memory: 28Gi, nvidia.com/gpu: 1}
             startupProbe:
               httpGet: {path: /health, port: http}
               periodSeconds: 10
               failureThreshold: 120
             readinessProbe:
               httpGet: {path: /health, port: http}
               periodSeconds: 5
               failureThreshold: 2
             livenessProbe:
               httpGet: {path: /health, port: http}
               periodSeconds: 30
               failureThreshold: 3
             lifecycle:
               preStop:
                 exec: {command: [sh, -c, "sleep 20"]}   # let the LB drain before SIGTERM
             volumeMounts:
               - {name: hf-cache, mountPath: /data/hf}
               - {name: shm, mountPath: /dev/shm}
         volumes:
           - name: hf-cache
             persistentVolumeClaim: {claimName: hf-cache}
           - name: shm
             emptyDir: {medium: Memory, sizeLimit: 8Gi}
   ---
   apiVersion: v1
   kind: Service
   metadata:
     name: vllm
     namespace: llm
     labels: {app: vllm}
   spec:
     selector: {app: vllm}
     ports: [{name: http, port: 8000, targetPort: http}]
   ---
   apiVersion: policy/v1
   kind: PodDisruptionBudget
   metadata: {name: vllm, namespace: llm}
   spec:
     minAvailable: 1
     selector: {matchLabels: {app: vllm}}
   ---
   apiVersion: networking.k8s.io/v1
   kind: NetworkPolicy
   metadata: {name: vllm-ingress, namespace: llm}
   spec:
     podSelector: {matchLabels: {app: vllm}}
     policyTypes: [Ingress]
     ingress:
       - from:
           - namespaceSelector: {matchLabels: {kubernetes.io/metadata.name: incident-agent}}
           - namespaceSelector: {matchLabels: {kubernetes.io/metadata.name: monitoring}}
         ports: [{port: 8000}]
   ---
   apiVersion: monitoring.coreos.com/v1
   kind: ServiceMonitor
   metadata:
     name: vllm
     namespace: llm
     labels: {release: kube-prometheus-stack}
   spec:
     selector: {matchLabels: {app: vllm}}
     endpoints:
       - port: http
         path: /metrics
         interval: 15s
         bearerTokenSecret: {name: vllm-api, key: VLLM_API_KEY}
   ```

   ```bash
   kubectl apply -f vllm-prod.yaml
   kubectl -n llm get pods -w
   ```

   Expected output (first boot includes download, 5-10 minutes):

   ```text
   NAME                    READY   STATUS    RESTARTS   AGE
   vllm-6c9f7b8d4-x2k9p    0/1     Running   0          4m
   vllm-6c9f7b8d4-x2k9p    1/1     Running   0          7m
   ```

3. Test with the API key.

   ```bash
   KEY=$(kubectl -n llm get secret vllm-api -o jsonpath='{.data.VLLM_API_KEY}' | base64 -d)
   kubectl -n llm port-forward svc/vllm 8000:8000 &
   curl -s localhost:8000/v1/models -H "Authorization: Bearer $KEY" | jq '.data[].id'
   curl -s -o /dev/null -w '%{http_code}\n' localhost:8000/v1/models        # 401 without key
   ```

4. Confirm Prometheus sees the target: Prometheus UI, Status > Targets, job `llm/vllm/0` is UP. Import the vLLM Grafana dashboard from the vLLM repo (`examples/online_serving/prometheus_grafana/grafana.json`) or build from the PromQL in [03-commands-and-cheatsheet.md](03-commands-and-cheatsheet.md).

5. Roll a config change with zero downtime: edit `--max-num-seqs=96`, `kubectl apply`, and watch `maxSurge: 1` create the new pod (needs a second GPU or Karpenter will provision one) before removing the old.

### Verify

- `kubectl -n llm get pdb vllm` shows `ALLOWED DISRUPTIONS 0` while one replica exists (node drains will be blocked until a second replica is Ready; that is intentional).
- The Pod runs as UID 1000 (`kubectl -n llm exec deploy/vllm -- id`).

### Cleanup

Keep for Lab 6, then `kubectl delete namespace llm`.

### What you learned

The production checklist for a GPU workload: non-root, restricted PSA, probes tuned for slow startup, PDB, NetworkPolicy, secrets from a store, metrics scraped with auth.

---

## Lab 6: Autoscaling on Queue Depth with KEDA

**Objective.** Scale vLLM replicas from 1 to 3 when `vllm:num_requests_waiting` exceeds 5.

**Prerequisites.** Lab 5; KEDA installed (`helm install keda kedacore/keda -n keda --create-namespace`); Karpenter GPU NodePool with `limits.nvidia.com/gpu: 3`.

**Time.** 45 minutes. Cost: ~$1 (three spot GPUs for ~30 minutes).

### Steps

1. Apply the ScaledObject.

   ```yaml
   apiVersion: keda.sh/v1alpha1
   kind: ScaledObject
   metadata: {name: vllm, namespace: llm}
   spec:
     scaleTargetRef: {name: vllm}
     minReplicaCount: 1
     maxReplicaCount: 3
     pollingInterval: 15
     cooldownPeriod: 600          # GPUs are slow to start; do not flap
     advanced:
       horizontalPodAutoscalerConfig:
         behavior:
           scaleUp:   {stabilizationWindowSeconds: 60,  policies: [{type: Pods, value: 1, periodSeconds: 60}]}
           scaleDown: {stabilizationWindowSeconds: 600, policies: [{type: Pods, value: 1, periodSeconds: 300}]}
     triggers:
       - type: prometheus
         metadata:
           serverAddress: http://kube-prometheus-stack-prometheus.monitoring.svc:9090
           query: sum(vllm:num_requests_waiting{namespace="llm"})
           threshold: "5"
           activationThreshold: "1"
   ```

2. Generate load from inside the cluster.

   ```bash
   kubectl -n llm run loadgen --rm -it --image=python:3.12-slim --env="KEY=$KEY" -- bash -c '
   pip -q install openai && python - <<EOF
   import asyncio, os
   from openai import AsyncOpenAI
   c = AsyncOpenAI(base_url="http://vllm:8000/v1", api_key=os.environ["KEY"])
   async def one(i):
       await c.chat.completions.create(model="qwen2.5-7b", max_tokens=300,
           messages=[{"role":"user","content":"Write a 300 word runbook for incident %d" % i}])
   async def main():
       await asyncio.gather(*(one(i) for i in range(150)))
   asyncio.run(main())
   EOF'
   ```

3. Watch scaling.

   ```bash
   kubectl -n llm get hpa -w
   kubectl get nodes -L karpenter.sh/nodepool -w
   ```

   Expected output:

   ```text
   NAME             REFERENCE         TARGETS      MINPODS   MAXPODS   REPLICAS
   keda-hpa-vllm    Deployment/vllm   23/5 (avg)   1         3         1
   keda-hpa-vllm    Deployment/vllm   23/5 (avg)   1         3         2
   ```

   A new GPU node appears within ~90 s; the new pod becomes Ready after ~3 minutes (weights come from the fresh node's empty cache unless you use a shared RWX volume or S3; see [05-production-patterns.md](05-production-patterns.md)).

### Verify

After load stops, replicas return to 1 after the 10-minute cooldown and Karpenter consolidates the empty nodes.

### Cleanup

```bash
kubectl -n llm delete scaledobject vllm
kubectl delete namespace llm
kubectl delete nodepool gpu-spot     # terminates GPU nodes
```

### What you learned

Autoscale on queue depth, not CPU; GPU scale-up is minutes, so stabilise and cool down aggressively; cold model loading is the real bottleneck.

---

## Lab 7: LoRA Adapters and Multi-GPU Tensor Parallelism

**Objective.** Serve a base model plus two LoRA adapters as separate model names; then run TP=2 on a 2-GPU node (`g5.12xlarge` has 4 A10G; `g6.12xlarge` has 4 L4) or two-node PP with Ray.

**Prerequisites.** Docker GPU host or the K8s manifests from Lab 5.

**Time.** 60 minutes. Cost: ~$2 for a `g5.12xlarge` spot hour.

### Steps

1. LoRA on a single GPU.

   ```bash
   docker run -d --name vllm --runtime nvidia --gpus all --ipc=host -p 8000:8000 \
     -v "$HOME/.cache/huggingface:/root/.cache/huggingface" \
     vllm/vllm-openai:v0.10.1 --model meta-llama/Llama-3.1-8B-Instruct --dtype bfloat16 --max-model-len 8192 \
     --enable-lora --max-lora-rank 64 \
     --lora-modules sql-adapter=<HF_ORG>/llama-3.1-8b-text2sql-lora \
     --served-model-name llama-3.1-8b
   curl -s localhost:8000/v1/models | jq '.data[] | {id, parent}'
   ```

   Expected output:

   ```json
   {"id": "llama-3.1-8b", "parent": null}
   {"id": "sql-adapter", "parent": "llama-3.1-8b"}
   ```

   Requests with `"model": "sql-adapter"` apply the adapter; vLLM batches adapter and base requests together (Punica/SGMV kernels).

2. Tensor parallel across 2 GPUs.

   ```bash
   docker rm -f vllm
   docker run -d --name vllm --runtime nvidia --gpus '"device=0,1"' --ipc=host -p 8000:8000 \
     -v "$HOME/.cache/huggingface:/root/.cache/huggingface" \
     vllm/vllm-openai:v0.10.1 --model meta-llama/Llama-3.1-8B-Instruct --dtype bfloat16 \
     --tensor-parallel-size 2 --max-model-len 32768 --served-model-name llama-3.1-8b
   docker logs vllm 2>&1 | grep -E 'KV cache size|NCCL|rank'
   nvidia-smi --query-gpu=index,memory.used --format=csv
   ```

   Expected: both GPUs show ~8 GB of weights each plus KV cache; total KV tokens roughly 2.3x the single-GPU figure (weights halve per GPU, cache sums).

3. Multi-node with Ray (two hosts, `HEAD_IP` reachable on 6379 and 8000-9000). On the head:

   ```bash
   bash examples/online_serving/run_cluster.sh vllm/vllm-openai:v0.10.1 <HEAD_IP> --head /root/.cache/huggingface -e NCCL_SOCKET_IFNAME=eth0
   ```

   On each worker:

   ```bash
   bash examples/online_serving/run_cluster.sh vllm/vllm-openai:v0.10.1 <HEAD_IP> --worker /root/.cache/huggingface -e NCCL_SOCKET_IFNAME=eth0
   ```

   Then inside the head container:

   ```bash
   ray status                      # shows GPUs from all nodes
   vllm serve meta-llama/Llama-3.1-70B-Instruct --tensor-parallel-size 4 --pipeline-parallel-size 2 \
     --distributed-executor-backend ray --max-model-len 16384
   ```

### Verify

`curl localhost:8000/v1/models` lists the model; `ray status` shows all GPUs in use.

### Cleanup

`docker rm -f vllm node` on all hosts; terminate instances.

### What you learned

Multi-LoRA turns one GPU into many "models"; TP divides weights but needs fast interconnect; Ray is the glue across nodes.

---

## Lab 8: Benchmark and Cost per Million Tokens

**Objective.** Measure throughput and latency at the capstone's traffic shape and compute $/1M tokens vs a managed API.

**Prerequisites.** Lab 4 or 5 running on a `g5.xlarge` (A10G).

**Time.** 45 minutes.

### Steps

1. Run the benchmark at RAG-like lengths.

   ```bash
   pip install vllm[bench] 2>/dev/null || pip install vllm pandas datasets
   vllm bench serve --backend openai-chat --base-url http://localhost:8000 --endpoint /v1/chat/completions \
     --model qwen2.5-7b --tokenizer Qwen/Qwen2.5-7B-Instruct-AWQ \
     --dataset-name random --random-input-len 3000 --random-output-len 400 \
     --num-prompts 120 --max-concurrency 24 --request-rate inf --percentile-metrics ttft,tpot,itl,e2el
   ```

   Expected output (A10G, AWQ, FP8 KV; your numbers will differ):

   ```text
   ============ Serving Benchmark Result ============
   Successful requests:                     120
   Benchmark duration (s):                  118.4
   Total input tokens:                      360000
   Total generated tokens:                  48000
   Request throughput (req/s):              1.01
   Output token throughput (tok/s):         405.4
   Total Token throughput (tok/s):          3445.9
   ---------------Time to First Token----------------
   Mean TTFT (ms):                          612.3
   Median TTFT (ms):                        548.1
   P99 TTFT (ms):                           1420.7
   -----Time per Output Token (excl. 1st token)------
   Mean TPOT (ms):                          52.8
   P99 TPOT (ms):                           71.2
   ==================================================
   ```

2. Compute cost. Spot `g5.xlarge` is roughly $0.30-0.45/h (check current pricing); take $0.40.

   ```text
   tokens/hour  = 3446 tok/s x 3600 = 12.4M total tokens/hour (10.8M input + 1.5M output)
   $/1M tokens  = 0.40 / 12.4 = $0.032 per 1M total tokens at 100 % utilisation
   at 30 % average utilisation: $0.11 per 1M tokens
   ```

   Compare against a managed small model's list price (check the provider page for current numbers; small-model classes are typically on the order of $0.1-0.3 per 1M input and $0.3-1.0 per 1M output). Self-hosting wins only when the GPU is kept busy; write the utilisation break-even into your ADR.

3. Re-run with `--max-concurrency 8` and observe TTFT drop while throughput falls. That is the throughput/latency curve; pick the point that meets the 800 ms p95 SLO.

### Verify

Benchmark JSON saved with `--save-result`; PromQL `histogram_quantile(0.95, ...ttft...)` during the run agrees with the client-side P95 within ~10 %.

### Cleanup

Stop the instance. Confirm with `aws ec2 describe-instances --filters Name=instance-state-name,Values=running --query 'Reservations[].Instances[].InstanceType'` that no GPU instance remains.

### What you learned

How to produce the one number that justifies self-hosting, and that utilisation, not list price, decides it.
