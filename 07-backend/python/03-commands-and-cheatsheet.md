# Python: Commands and Cheatsheet

| | |
|---|---|
| **Cluster** | 07 Backend & APIs |
| **Role** | Language foundation |
| **Prerequisites** | [01-installation-and-setup.md](01-installation-and-setup.md), [02-core-concepts.md](02-core-concepts.md) |
| **Target version** | Python 3.12+, uv 0.5+, ruff 0.6+, mypy 1.11+, pytest 8+ |
| **Capstone phase** | Phase 1 (Months 1-4) |

> Reference tables organised by task. Every command has a one-line explanation. Short flags are shown next to long ones where both exist.

---

## 1. `uv`: Environments, Dependencies, Interpreters

| Command | What it does |
|---|---|
| `uv init --package <name>` | New project with `src/` layout, `pyproject.toml`, `.python-version` |
| `uv init --app <name>` | New flat-layout application (scripts, not a library) |
| `uv python install 3.12` | Download and install CPython 3.12 (python-build-standalone) |
| `uv python list` | Show installed and available interpreters |
| `uv python pin 3.12` | Write `.python-version` for this project |
| `uv add httpx` | Add a runtime dependency, resolve, lock, sync |
| `uv add "pydantic>=2.7,<3"` | Add with a version constraint |
| `uv add --dev pytest` | Add to the `dev` dependency group |
| `uv add --optional aws boto3` | Add to the `aws` extra |
| `uv remove httpx` | Remove a dependency and re-lock |
| `uv sync` | Make `.venv` match `uv.lock` exactly (installs project editable) |
| `uv sync --frozen` | Sync without touching the lockfile; fail if out of date (use in Docker) |
| `uv sync --no-dev --all-extras` | Production install with all extras |
| `uv lock` | Resolve and write `uv.lock` |
| `uv lock --check` | Exit non-zero if the lockfile is stale (CI gate) |
| `uv lock --upgrade` | Upgrade everything within constraints |
| `uv lock --upgrade-package httpx` | Upgrade one package |
| `uv run <cmd>` | Run a command inside the project environment (syncs first) |
| `uv run python -m platform_cli` | Run the package as a module |
| `uv run --with rich python script.py` | Run with an extra temporary dependency |
| `uv tree` | Show the dependency tree |
| `uv tree --invert --package urllib3` | Who depends on `urllib3`? |
| `uv export --format requirements-txt --no-dev -o requirements.txt` | Generate a pinned requirements file for legacy tooling and scanners |
| `uv build` | Build sdist and wheel into `dist/` |
| `uv publish` | Upload to an index (use OIDC trusted publishing in CI) |
| `uvx ruff check .` | Run a tool in an ephemeral environment without installing it |
| `uv tool install ruff` | Install a CLI tool globally (replaces `pipx`) |
| `uv tool list` / `uv tool upgrade --all` | Manage global tools |
| `uv cache clean` | Clear the download/build cache |
| `uv self update` | Update `uv` |
| `UV_INDEX_URL=https://<HOST>/simple uv sync` | Use a private index (CodeArtifact, Artifactory) |

Inline script dependencies (PEP 723) for single-file tools:

```python
# /// script
# requires-python = ">=3.12"
# dependencies = ["httpx", "rich"]
# ///
import httpx
```

```bash
uv run script.py      # creates an isolated env with those deps
```

---

## 2. `pip` and `venv` Equivalents

| Task | Command |
|---|---|
| Create venv | `python3 -m venv .venv` |
| Activate (bash) / (PowerShell) | `source .venv/bin/activate` / `.venv\Scripts\Activate.ps1` |
| Install from lock | `pip install --require-hashes -r requirements.txt` |
| Install project editable | `pip install -e ".[dev]"` |
| Show what is installed | `pip list --format=columns` / `pip freeze` |
| Show one package | `pip show httpx` |
| Check for conflicts | `pip check` |
| Compile lock (pip-tools) | `pip-compile --generate-hashes pyproject.toml -o requirements.txt` |
| Audit CVEs | `pip-audit -r requirements.txt` |
| Download wheels for offline | `pip download -r requirements.txt -d wheels/` |

---

## 3. `ruff`: Lint and Format

| Command | What it does |
|---|---|
| `ruff check .` | Lint everything per `pyproject.toml` |
| `ruff check --fix .` | Lint and auto-fix safe fixes |
| `ruff check --unsafe-fixes --fix .` | Also apply fixes that may change behaviour (review the diff) |
| `ruff check --select S .` | Run only bandit-style security rules |
| `ruff check --select ASYNC .` | Find blocking calls inside `async def` |
| `ruff check --statistics .` | Count violations by rule |
| `ruff check --output-format=github .` | Annotate PRs in GitHub Actions |
| `ruff check --output-format=sarif . > ruff.sarif` | SARIF for GitHub code scanning / DefectDojo |
| `ruff format .` | Format (Black-compatible) |
| `ruff format --check --diff .` | CI mode: fail if formatting would change |
| `ruff rule B006` | Explain a rule |
| `ruff linter` | List all rule groups |
| `# noqa: E501` | Suppress one rule on one line (must name the rule) |

---

## 4. `mypy`: Static Types

| Command | What it does |
|---|---|
| `mypy` | Check packages listed in `[tool.mypy]` |
| `mypy src/ tests/` | Check paths |
| `mypy --strict src/` | All strictness flags on |
| `mypy --show-error-codes` | Print `[code]` so you can `# type: ignore[code]` precisely (default since 0.990) |
| `mypy --html-report .mypy-report` | Coverage of typed code |
| `mypy --install-types --non-interactive` | Install missing stub packages |
| `reveal_type(x)` (in code) | Ask mypy what it thinks `x` is; remove before commit |
| `mypy --warn-unused-ignores` | Find `# type: ignore` comments that are no longer needed |
| `uvx pyright` | Alternative checker (what VS Code Pylance uses); useful second opinion |

Common strict-mode fixes:

| Error | Fix |
|---|---|
| `Function is missing a return type annotation` | Add `-> None` or the type |
| `Returning Any from function declared to return "str"` | Validate/cast at the boundary (`str(x)`, Pydantic) |
| `Argument 1 has incompatible type "str \| None"; expected "str"` | Narrow: `if x is None: raise ...` |
| `Need type annotation for "items"` | `items: list[str] = []` |
| `Cannot find implementation or library stub` | Install `types-*` / `*-stubs`, or `ignore_missing_imports` override for that module |

---

## 5. `pytest`

| Command | What it does |
|---|---|
| `pytest` | Run everything under `testpaths` |
| `pytest -q` / `-v` / `-vv` | Quiet / verbose / show full diffs |
| `pytest tests/test_k8s.py::test_list_pods` | One test |
| `pytest -k "pods and not slow"` | Select by name expression |
| `pytest -m integration` | Select by marker |
| `pytest -x` / `--maxfail=3` | Stop at first / third failure |
| `pytest --lf` / `--ff` | Only last-failed / failed-first |
| `pytest -ra` | Summary of skips/xfails/errors |
| `pytest --pdb` | Drop into debugger on failure |
| `pytest -s` | Show stdout (disable capture) |
| `pytest --durations=10` | Ten slowest tests |
| `pytest -n auto` | Parallel (needs `pytest-xdist`) |
| `pytest --cov=platform_cli --cov-report=xml` | Coverage XML for SonarQube |
| `pytest --cov-fail-under=80` | Enforce coverage threshold |
| `pytest --junitxml=report.xml` | JUnit XML for CI dashboards |
| `pytest -W error` | Treat warnings as errors |
| `pytest --collect-only -q` | List tests without running |
| `pytest --fixtures` | List available fixtures |

Markers in `pyproject.toml`:

```toml
[tool.pytest.ini_options]
markers = [
  "integration: needs Docker (Testcontainers)",
  "slow: takes more than 5 seconds",
]
```

---

## 6. Interpreter Flags and Env Vars

| Flag / variable | Effect |
|---|---|
| `python -I` | Isolated mode: ignore env vars, user site, and script dir on `sys.path`. Use when running untrusted directories |
| `python -X importtime -c "import x"` | Print import timing tree (find slow startup) |
| `python -X dev` | Dev mode: extra warnings, asyncio debug, faulthandler |
| `python -X faulthandler` | Dump traceback on segfault |
| `python -W error` | Warnings become exceptions |
| `python -m <module>` | Run a module as a script (`python -m http.server 8080`) |
| `python -c "..."` | One-liner |
| `python -u` | Unbuffered stdout/stderr (containers) |
| `PYTHONUNBUFFERED=1` | Same as `-u`; set in every Dockerfile |
| `PYTHONDONTWRITEBYTECODE=1` | Do not write `.pyc` (read-only filesystems) |
| `PYTHONPATH=src` | Extra import paths (avoid; use `uv sync` editable install) |
| `PYTHONASYNCIODEBUG=1` | Log slow callbacks (>100 ms) and un-awaited coroutines |
| `PYTHONHASHSEED=0` | Deterministic hashing (reproducible test ordering) |
| `PYTHONWARNINGS=default` | Show `DeprecationWarning` |
| `PYTHONTRACEMALLOC=25` | Track allocations with 25-frame tracebacks |
| `PYTHONSAFEPATH=1` (3.11+) | Do not prepend script dir to `sys.path` |

---

## 7. Debugging and Profiling

| Command | What it does |
|---|---|
| `breakpoint()` (in code) | Start `pdb` (honours `PYTHONBREAKPOINT=ipdb.set_trace`) |
| `python -m pdb script.py` | Run under debugger |
| `python -m cProfile -o out.prof -m platform_cli pods list` | CPU profile a run |
| `python -c "import pstats; pstats.Stats('out.prof').sort_stats('cumtime').print_stats(20)"` | Read profile |
| `uvx snakeviz out.prof` | Browser flame view of a cProfile output |
| `uvx py-spy top --pid <PID>` | Live `top`-style sampling of a running process (no restart) |
| `uvx py-spy dump --pid <PID>` | Stack of every thread right now (find the blocking call) |
| `uvx py-spy record -o profile.svg --pid <PID> --duration 30` | Flame graph of a live process |
| `kubectl exec <POD> -- py-spy dump --pid 1` | Same inside a container (needs `SYS_PTRACE` capability and py-spy in image) |
| `python -X tracemalloc=10 script.py` then `tracemalloc.take_snapshot()` | Memory allocation tracking |
| `uvx memray run -o out.bin -m platform_cli ...` then `memray flamegraph out.bin` | Memory profiler |
| `python -m timeit -s "import json" "json.dumps({'a': 1})"` | Micro-benchmark |
| `uvx scalene script.py` | CPU + memory + GPU line profiler |

---

## 8. Security Tooling

| Command | What it does |
|---|---|
| `uvx pip-audit` | Scan the current environment against OSV/PyPI advisories |
| `uvx pip-audit -r requirements.txt --strict` | Scan a requirements file; fail on any finding |
| `uv export --no-dev \| uvx pip-audit -r /dev/stdin` | Scan the lock without a requirements file |
| `uvx bandit -r src/ -ll` | SAST for Python (medium+ severity); `ruff --select S` covers most of it faster |
| `uvx bandit -r src/ -f sarif -o bandit.sarif` | SARIF output |
| `uvx semgrep --config p/python src/` | Semgrep Python ruleset (cluster 08) |
| `gitleaks detect --source .` | Secrets in git history |
| `uvx cyclonedx-py environment -o sbom.json` | CycloneDX SBOM of the environment |
| `trivy fs --scanners vuln,secret,misconfig .` | Scan lockfile + Dockerfile |
| `uv lock --check` | Ensure lockfile matches declared deps (supply-chain hygiene) |

---

## 9. Stdlib Snippets by Task

### JSON, YAML, TOML

```python
import json, tomllib
data = json.loads(text)                      # str -> obj
json.dumps(data, indent=2, sort_keys=True, default=str)  # default=str handles datetime/UUID
with open("pyproject.toml", "rb") as f:      # tomllib (3.11+) is read-only, binary mode
    cfg = tomllib.load(f)
import yaml                                   # pip: pyyaml
docs = list(yaml.safe_load_all(open("manifests.yaml")))   # safe_load, never yaml.load
```

### Dates

```python
from datetime import datetime, timedelta, UTC
now = datetime.now(UTC)                       # always timezone-aware
iso = now.isoformat(timespec="milliseconds")  # 2026-10-27T10:15:02.123+00:00
parsed = datetime.fromisoformat("2026-10-27T10:15:02Z")   # 3.11+ accepts Z
cutoff = now - timedelta(days=30)
```

### Environment and secrets

```python
import os
region = os.environ.get("AWS_REGION", "eu-west-1")     # default if absent
token = os.environ["API_TOKEN"]                         # KeyError if absent: fail fast
```

### Retries with `tenacity`

```python
from tenacity import retry, stop_after_attempt, wait_exponential_jitter, retry_if_exception_type
import httpx

@retry(stop=stop_after_attempt(5), wait=wait_exponential_jitter(initial=0.5, max=10),
       retry=retry_if_exception_type((httpx.ConnectError, httpx.ReadTimeout)))
async def get(client: httpx.AsyncClient, url: str) -> httpx.Response:
    r = await client.get(url)
    r.raise_for_status()
    return r
```

### Concurrency one-liners

```python
import asyncio
results = await asyncio.gather(*coros, return_exceptions=True)
async with asyncio.timeout(5): await coro
await asyncio.to_thread(blocking_fn, arg)
from concurrent.futures import ThreadPoolExecutor, ProcessPoolExecutor
with ThreadPoolExecutor(max_workers=8) as ex: list(ex.map(fetch, urls))
```

### Hashing and randomness

```python
import hashlib, secrets
digest = hashlib.sha256(data).hexdigest()
token = secrets.token_urlsafe(32)             # cryptographically secure; never random.random for tokens
```

### Temp files and atomic writes

```python
import tempfile
from pathlib import Path
with tempfile.TemporaryDirectory() as d:
    p = Path(d) / "kubeconfig"
    p.write_text(kubeconfig, encoding="utf-8")
    p.chmod(0o600)
```

---

## 10. Kubernetes Client Snippets

```python
from kubernetes import client, config, watch

config.load_kube_config()                     # local; in-cluster: config.load_incluster_config()
v1 = client.CoreV1Api()
apps = client.AppsV1Api()

pods = v1.list_namespaced_pod("platform", label_selector="app=incident-api")
for p in pods.items:
    print(p.metadata.name, p.status.phase, [c.restart_count for c in p.status.container_statuses or []])

# Patch a deployment image (strategic merge patch)
apps.patch_namespaced_deployment(
    name="incident-api", namespace="platform",
    body={"spec": {"template": {"spec": {"containers": [{"name": "api", "image": "ghcr.io/example/incident-api:v2"}]}}}},
)

# Watch events
w = watch.Watch()
for event in w.stream(v1.list_namespaced_event, namespace="platform", timeout_seconds=60):
    e = event["object"]
    print(event["type"], e.reason, e.message)
```

Auto-detect environment:

```python
def load_k8s() -> None:
    try:
        config.load_incluster_config()
    except config.ConfigException:
        config.load_kube_config()
```

---

## 11. `boto3` Snippets

```python
import boto3
sts = boto3.client("sts"); print(sts.get_caller_identity()["Arn"])      # who am I
s3 = boto3.client("s3")
s3.upload_file("report.json", "<BUCKET_NAME>", "reports/report.json",
               ExtraArgs={"ServerSideEncryption": "aws:kms", "SSEKMSKeyId": "<KMS_KEY_ARN>"})
ssm = boto3.client("ssm")
val = ssm.get_parameter(Name="/platform/db/url", WithDecryption=True)["Parameter"]["Value"]
sm = boto3.client("secretsmanager")
secret = sm.get_secret_value(SecretId="platform/db")["SecretString"]
eks = boto3.client("eks"); eks.describe_cluster(name="<CLUSTER_NAME>")["cluster"]["endpoint"]
logs = boto3.client("logs")
for page in logs.get_paginator("filter_log_events").paginate(logGroupName="/eks/platform", filterPattern="ERROR"):
    for ev in page["events"]: print(ev["message"])
```

Assume a role (cross-account automation):

```python
creds = sts.assume_role(RoleArn="arn:aws:iam::123456789012:role/platform-readonly",
                        RoleSessionName="platform-cli", DurationSeconds=900)["Credentials"]
s3_other = boto3.client("s3", aws_access_key_id=creds["AccessKeyId"],
                        aws_secret_access_key=creds["SecretAccessKey"], aws_session_token=creds["SessionToken"])
```

---

## 12. Packaging and Release

| Command | What it does |
|---|---|
| `uv build` | Build wheel + sdist |
| `uv run python -m zipfile -l dist/*.whl` | Inspect wheel contents (is `py.typed` there?) |
| `uvx twine check dist/*` | Validate metadata |
| `uv version` / `uv version --bump minor` | Show / bump `project.version` |
| `git tag -a v0.2.0 -m "..." && git push --tags` | Release tag; CI builds the image |
| `uvx check-wheel-contents dist/` | Lint the wheel |

---

## 13. Docker One-Liners for Python

| Command | What it does |
|---|---|
| `docker build -t platform-cli:dev .` | Build |
| `docker run --rm platform-cli:dev --help` | Run the entrypoint |
| `docker run --rm -it --entrypoint python platform-cli:dev -c "import sys; print(sys.version)"` | Check interpreter (only works on non-distroless) |
| `docker run --rm --user 65532 --read-only --tmpfs /tmp platform-cli:dev pods list` | Prove it runs non-root on a read-only FS |
| `docker image inspect platform-cli:dev --format '{{.Size}}'` | Image size in bytes |
| `dive platform-cli:dev` | Layer-by-layer size explorer |
| `trivy image --severity HIGH,CRITICAL platform-cli:dev` | CVE scan |
| `syft platform-cli:dev -o cyclonedx-json > sbom.json` | SBOM |
