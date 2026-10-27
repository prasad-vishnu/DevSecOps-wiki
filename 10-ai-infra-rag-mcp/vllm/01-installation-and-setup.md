# vLLM: Installation and Setup

| | |
|---|---|
| **Cluster** | 10 AI Infra & RAG/MCP |
| **Role** | Anchor |
| **Prerequisites** | Python 3.12+, Docker 27+, optionally an NVIDIA GPU with driver 550+ (CUDA 12.x) |
| **Target version** | vLLM 0.10+ |
| **Capstone phase** | Phase 6 (Months 21-24) |

> After this file you will have vLLM running in three ways: a CPU-only install on your laptop serving a 0.5B model (free, good enough to learn the API and metrics), the official Docker image on a GPU host, and a first deployment on a Kubernetes cluster with a GPU node. You will also know how to authenticate to Hugging Face for gated models without leaking a token.

---

## 1. Choose Your Path

| Path | Hardware | Cost | Use it for |
|---|---|---|---|
| A. CPU build (`pip` or Docker `vllm/vllm-openai:...-cpu` / build from source) | Any x86-64 with AVX-512 preferred, 8 GB RAM | $0 | Learning the API, chat templates, tool calling, metrics, writing the agent client |
| B. GPU `pip install vllm` | Linux, NVIDIA driver 550+, 1x GPU with 16 GB+ | Colab T4 free / spot `g5.xlarge` ~$0.30-0.40/h | Offline inference, quantised 7-8B models, benchmarks |
| C. Docker `vllm/vllm-openai` | Same as B plus NVIDIA Container Toolkit | Same | Exactly what Kubernetes will run |
| D. Kubernetes | GPU node with GPU Operator | Spot node, nightly teardown | The capstone |

vLLM supports AMD ROCm, Intel XPU/Gaudi, AWS Neuron (Inferentia/Trainium), Google TPU and CPU backends. This folder targets NVIDIA CUDA and CPU; the serving layer and API are identical across backends.

---

## 2. Path A: CPU-Only Install (Laptop Lab)

The CPU backend is slower by one to two orders of magnitude than a GPU, but a 0.5B-1.5B instruct model produces a few tokens per second, which is plenty for learning.

### 2.1 Linux / WSL2 with pip

```bash
python3.12 -m venv ~/.venvs/vllm-cpu
source ~/.venvs/vllm-cpu/bin/activate
pip install --upgrade pip

# Install CPU PyTorch first so pip does not pull the CUDA wheels (several GB).
pip install torch --index-url https://download.pytorch.org/whl/cpu

# Pre-built CPU wheels are published for x86-64 Linux from the vLLM release page.
# If a wheel is unavailable for your version, build from source (see 2.3).
pip install vllm --extra-index-url https://download.pytorch.org/whl/cpu
```

Set the CPU backend's KV cache size (GiB) and thread binding before serving:

```bash
export VLLM_CPU_KVCACHE_SPACE=4          # GiB reserved for KV cache
export VLLM_CPU_OMP_THREADS_BIND=0-7     # pin OpenMP threads to cores 0-7
export VLLM_TARGET_DEVICE=cpu
```

### 2.2 Docker CPU image

```bash
docker run --rm -it \
  --name vllm-cpu \
  -p 8000:8000 \
  -e VLLM_CPU_KVCACHE_SPACE=4 \
  -v "$HOME/.cache/huggingface:/root/.cache/huggingface" \
  public.ecr.aws/q9t5s3a7/vllm-cpu-release-repo:latest \
  --model Qwen/Qwen2.5-0.5B-Instruct \
  --dtype bfloat16 \
  --max-model-len 4096
```

The CPU image name has moved between releases; check https://docs.vllm.ai/en/latest/getting_started/installation/cpu.html for the current tag, or build it yourself:

### 2.3 Build the CPU image from source

```bash
git clone --depth 1 https://github.com/vllm-project/vllm.git
cd vllm
docker build -f docker/Dockerfile.cpu -t vllm-cpu-env --shm-size=4g .
docker run --rm -it -p 8000:8000 --shm-size=4g \
  -v "$HOME/.cache/huggingface:/root/.cache/huggingface" \
  vllm-cpu-env --model Qwen/Qwen2.5-0.5B-Instruct --max-model-len 4096
```

### 2.4 First request

In a second terminal:

```bash
curl -s http://localhost:8000/v1/models | python -m json.tool
```

Expected output (trimmed):

```json
{
  "object": "list",
  "data": [
    {
      "id": "Qwen/Qwen2.5-0.5B-Instruct",
      "object": "model",
      "owned_by": "vllm",
      "max_model_len": 4096
    }
  ]
}
```

```bash
curl -s http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "Qwen/Qwen2.5-0.5B-Instruct",
    "messages": [{"role": "user", "content": "In one sentence, what is a Kubernetes PodDisruptionBudget?"}],
    "max_tokens": 64,
    "temperature": 0
  }' | python -m json.tool
```

Expected output (trimmed):

```json
{
  "id": "chatcmpl-5f2c...",
  "object": "chat.completion",
  "model": "Qwen/Qwen2.5-0.5B-Instruct",
  "choices": [
    {
      "index": 0,
      "message": {
        "role": "assistant",
        "content": "A PodDisruptionBudget limits how many pods of an application can be unavailable at once during voluntary disruptions such as node drains."
      },
      "finish_reason": "stop"
    }
  ],
  "usage": {"prompt_tokens": 31, "completion_tokens": 27, "total_tokens": 58}
}
```

---

## 3. Path B: GPU pip Install

### 3.1 Verify the host

```bash
nvidia-smi --query-gpu=name,driver_version,memory.total --format=csv
```

Expected output:

```text
name, driver_version, memory.total [MiB]
NVIDIA A10G, 560.35.03, 23028 MiB
```

Driver 550+ is required for CUDA 12.4+ wheels. You do **not** need the CUDA toolkit on the host; the PyTorch wheels bundle the CUDA runtime. You do need the driver.

### 3.2 Install

```bash
# uv is dramatically faster for the multi-GB torch wheels
curl -LsSf https://astral.sh/uv/install.sh | sh
uv venv ~/.venvs/vllm --python 3.12
source ~/.venvs/vllm/bin/activate
uv pip install vllm            # pulls the matching CUDA 12.x torch build
python -c "import vllm, torch; print(vllm.__version__, torch.cuda.is_available())"
```

Expected output:

```text
0.10.1 True
```

For a specific CUDA minor (for example CUDA 11.8 hosts), use the wheels from the release page:

```bash
export VLLM_VERSION=0.10.1
export PYTHON_VERSION=312
uv pip install "https://github.com/vllm-project/vllm/releases/download/v${VLLM_VERSION}/vllm-${VLLM_VERSION}+cu118-cp38-abi3-manylinux1_x86_64.whl" \
  --extra-index-url https://download.pytorch.org/whl/cu118
```

### 3.3 Serve

```bash
vllm serve Qwen/Qwen2.5-7B-Instruct-AWQ \
  --quantization awq_marlin \
  --dtype half \
  --max-model-len 8192 \
  --gpu-memory-utilization 0.90 \
  --port 8000
```

Startup log lines worth recognising:

```text
INFO ... Automatically detected platform cuda.
INFO ... Initializing a V1 LLM engine (v0.10.1) with config: model='Qwen/Qwen2.5-7B-Instruct-AWQ', ... quantization=awq_marlin, ...
INFO ... Loading weights took 12.41 seconds
INFO ... Model loading took 5.21 GiB and 14.02 seconds
INFO ... GPU KV cache size: 118,944 tokens
INFO ... Maximum concurrency for 8,192 tokens per request: 14.52x
INFO ... Graph capturing finished in 9 secs, took 0.52 GiB
INFO ... Starting vLLM API server on http://0.0.0.0:8000
INFO ... Application startup complete.
```

The `GPU KV cache size` and `Maximum concurrency` lines are the two numbers you will tune in [02-core-concepts.md](02-core-concepts.md).

---

## 4. Path C: Docker on a GPU Host

### 4.1 NVIDIA Container Toolkit

```bash
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg
curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
  sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
  sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
sudo apt-get update && sudo apt-get install -y nvidia-container-toolkit
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
docker run --rm --gpus all nvidia/cuda:12.4.1-base-ubuntu22.04 nvidia-smi
```

### 4.2 Run the official image

```bash
docker run --rm -d \
  --name vllm \
  --runtime nvidia --gpus all \
  --ipc=host \
  -p 8000:8000 \
  -v "$HOME/.cache/huggingface:/root/.cache/huggingface" \
  --env "HF_TOKEN=${HF_TOKEN}" \
  vllm/vllm-openai:v0.10.1 \
  --model meta-llama/Llama-3.1-8B-Instruct \
  --dtype bfloat16 \
  --max-model-len 16384 \
  --gpu-memory-utilization 0.92 \
  --enable-prefix-caching \
  --served-model-name llama-3.1-8b
```

Notes:

- `--ipc=host` (or `--shm-size=8g`) is required: PyTorch tensor-parallel workers share memory via `/dev/shm`.
- The image's entrypoint is `vllm serve`; everything after the image name is a flag to it. In older images the entrypoint was `python3 -m vllm.entrypoints.openai.api_server`, which accepts the same flags.
- Pin the image tag. `latest` changes weekly.
- `HF_TOKEN` is passed from your shell environment, never typed inline.

---

## 5. Hugging Face Authentication for Gated Models

Llama, Mistral and Gemma checkpoints are gated. Accept the licence on the model page, then create a **read-only, fine-grained** token scoped to "Read access to contents of all public gated repos you can access".

Locally:

```bash
pip install -U "huggingface_hub[cli]" hf_transfer
hf auth login            # paste token when prompted; stored in ~/.cache/huggingface/token
# or non-interactively from a secret manager:
export HF_TOKEN="$(aws secretsmanager get-secret-value --secret-id hf/read-token --query SecretString --output text)"
export HF_HUB_ENABLE_HF_TRANSFER=1   # parallel downloads, 5-10x faster on fast links
hf download meta-llama/Llama-3.1-8B-Instruct --exclude "original/*"
```

In Kubernetes, store it as a Secret sourced from your secret manager (External Secrets Operator in the capstone), and mount it as an environment variable:

```yaml
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: hf-token
  namespace: llm
spec:
  refreshInterval: 1h
  secretStoreRef:
    kind: ClusterSecretStore
    name: aws-secrets-manager
  target:
    name: hf-token
  data:
    - secretKey: HF_TOKEN
      remoteRef:
        key: hf/read-token
```

```yaml
env:
  - name: HF_TOKEN
    valueFrom:
      secretKeyRef:
        name: hf-token
        key: HF_TOKEN
  - name: HF_HUB_ENABLE_HF_TRANSFER
    value: "1"
```

Never bake the token into an image or a ConfigMap, and never commit `~/.cache/huggingface/token`.

---

## 6. Path D: First Kubernetes Deployment

Prerequisite: a node advertising `nvidia.com/gpu` (see [../gpu-workloads-kubernetes/01-installation-and-setup.md](../gpu-workloads-kubernetes/01-installation-and-setup.md)).

```yaml
# vllm-quickstart.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: llm
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: hf-cache
  namespace: llm
spec:
  accessModes: ["ReadWriteOnce"]
  storageClassName: gp3
  resources:
    requests:
      storage: 50Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vllm
  namespace: llm
spec:
  replicas: 1
  selector:
    matchLabels: {app: vllm}
  template:
    metadata:
      labels: {app: vllm}
    spec:
      runtimeClassName: nvidia
      tolerations:
        - key: nvidia.com/gpu
          operator: Exists
          effect: NoSchedule
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 1000
      containers:
        - name: vllm
          image: vllm/vllm-openai:v0.10.1
          args:
            - --model=Qwen/Qwen2.5-7B-Instruct-AWQ
            - --quantization=awq_marlin
            - --dtype=half
            - --max-model-len=8192
            - --gpu-memory-utilization=0.90
            - --served-model-name=qwen2.5-7b
            - --port=8000
          env:
            - name: HF_HOME
              value: /data/hf
            - name: HF_HUB_ENABLE_HF_TRANSFER
              value: "1"
          ports:
            - containerPort: 8000
              name: http
          resources:
            limits:
              nvidia.com/gpu: 1
              memory: 24Gi
              cpu: "4"
            requests:
              memory: 16Gi
              cpu: "2"
          volumeMounts:
            - name: hf-cache
              mountPath: /data/hf
            - name: shm
              mountPath: /dev/shm
          startupProbe:
            httpGet: {path: /health, port: http}
            periodSeconds: 10
            failureThreshold: 90      # 15 min for first download + load
          readinessProbe:
            httpGet: {path: /health, port: http}
            periodSeconds: 10
          livenessProbe:
            httpGet: {path: /health, port: http}
            periodSeconds: 30
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
spec:
  selector: {app: vllm}
  ports:
    - port: 8000
      targetPort: http
```

```bash
kubectl apply -f vllm-quickstart.yaml
kubectl --namespace llm rollout status deployment/vllm --timeout=20m
kubectl --namespace llm port-forward svc/vllm 8000:8000 &
curl -s localhost:8000/v1/models | python -c "import sys,json; print([m['id'] for m in json.load(sys.stdin)['data']])"
```

Expected output:

```text
deployment "vllm" successfully rolled out
['qwen2.5-7b']
```

---

## 7. Python Client Setup

vLLM speaks the OpenAI protocol, so the standard SDK works unchanged:

```bash
pip install openai
```

```python
from openai import OpenAI

client = OpenAI(base_url="http://localhost:8000/v1", api_key="not-used-locally")

resp = client.chat.completions.create(
    model="qwen2.5-7b",
    messages=[
        {"role": "system", "content": "You are an SRE assistant. Answer tersely."},
        {"role": "user", "content": "What does CrashLoopBackOff mean?"},
    ],
    max_tokens=80,
    temperature=0.2,
)
print(resp.choices[0].message.content)
print(resp.usage)
```

Expected output:

```text
CrashLoopBackOff means the kubelet keeps restarting a container that exits shortly after starting, with an exponentially increasing delay (up to 5 minutes) between restarts. Check `kubectl logs --previous` for the exit reason.
CompletionUsage(completion_tokens=52, prompt_tokens=33, total_tokens=85)
```

---

## 8. Verification Checklist

| Check | Command | Expect |
|---|---|---|
| Engine healthy | `curl -s -o /dev/null -w '%{http_code}\n' localhost:8000/health` | `200` |
| Model listed | `curl -s localhost:8000/v1/models` | your `--served-model-name` |
| Metrics exposed | `curl -s localhost:8000/metrics \| grep -c '^vllm:'` | > 30 |
| Tokeniser works | `curl -s localhost:8000/tokenize -H 'Content-Type: application/json' -d '{"model":"qwen2.5-7b","prompt":"hello world"}'` | `{"count":2,"tokens":[...]}` |
| GPU actually used | `nvidia-smi --query-gpu=memory.used --format=csv` | roughly `gpu-memory-utilization x total` |
| Streaming | `curl -N ... -d '{"stream":true,...}'` | `data: {...}` SSE lines ending `data: [DONE]` |

---

## 9. Uninstall / Cleanup

```bash
docker rm -f vllm vllm-cpu 2>/dev/null
kubectl delete namespace llm
rm -rf ~/.venvs/vllm ~/.venvs/vllm-cpu
# Model weights are large; delete only what you no longer need:
hf cache scan
hf cache delete --disable-tui   # interactive selection of revisions to remove
```

On cloud, the GPU node must also go. In the capstone this is the nightly cron; manually:

```bash
kubectl delete nodepool gpu-spot      # Karpenter drains and terminates the instance
```
