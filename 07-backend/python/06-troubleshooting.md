# Python: Troubleshooting

| | |
|---|---|
| **Cluster** | 07 Backend & APIs |
| **Role** | Language foundation |
| **Prerequisites** | [02-core-concepts.md](02-core-concepts.md) |
| **Target version** | Python 3.12+, uv 0.5+ |
| **Capstone phase** | Phase 1 (Months 1-4) |

> Symptom-to-cause-to-fix tables for the failures you will actually hit: environment and import problems, packaging, asyncio, type checking, containers, and cloud/Kubernetes client errors. Each error message is quoted as it appears so you can search this page.

---

## 1. Debugging Workflow

```mermaid
flowchart TD
    A[Failure] --> B{Where?}
    B -->|install / import| C[uv run python -c 'import pkg'<br/>uv tree, which python]
    B -->|runtime exception| D[Read the LAST traceback frame in YOUR code<br/>python -X dev]
    B -->|hang| E[py-spy dump --pid PID<br/>PYTHONASYNCIODEBUG=1]
    B -->|wrong result| F[breakpoint() / pytest --pdb<br/>reveal_type for typing]
    B -->|container only| G[docker run -it --entrypoint sh image<br/>kubectl debug ephemeral container<br/>compare env, user, paths]
    C & D & E & F & G --> H[Write a failing test that reproduces it]
    H --> I[Fix, run full suite, add to this table if novel]
```

Three habits: (1) read tracebacks bottom-up and find the first frame in your own code, (2) reproduce in a test before fixing, (3) when it only fails in the container, diff the environment (`env`, `id`, `python --version`, `pip list`) between local and container.

---

## 2. Environment and Imports

| Symptom | Likely cause | Diagnose | Fix |
|---|---|---|---|
| `ModuleNotFoundError: No module named 'platform_cli'` | Package not installed in the environment being used; running `python` instead of `uv run python` | `uv run python -c "import sys; print(sys.prefix)"`; `which python` | Always `uv run`; ensure `[build-system]` in `pyproject.toml` so `uv sync` installs the project |
| `ModuleNotFoundError` only in Docker | `COPY src ./src` happened before `uv sync` that installs the project, or `--no-install-project` was used in the final sync | `docker run --rm --entrypoint python image -c "import platform_cli"` | Final `uv sync --frozen --no-dev` after copying `src` |
| `ImportError: cannot import name 'X' from partially initialized module` | Circular import | Traceback shows two modules importing each other | Move the import inside the function, or split the module |
| `error: externally-managed-environment` | `pip install` into system Python (PEP 668) | Distro is Debian 12+/Ubuntu 23.04+ | Use `uv` or a venv; never `--break-system-packages` on a server |
| `uv sync` fails: `No solution found when resolving dependencies` | Conflicting version constraints | Read the resolver's explanation; `uv tree --invert --package <pkg>` | Relax a constraint or upgrade the pinned sibling |
| `error: Failed to build ... Could not find a version that satisfies` with `musl` in path | Alpine image, no wheel for musl | Base image is `python:*-alpine` | Switch to `-slim-bookworm` |
| `warning: The lockfile is out of date` / `uv sync --frozen` refuses | `pyproject.toml` changed without `uv lock` | `uv lock --check` | Run `uv lock`, commit `uv.lock` |
| `SyntaxError: invalid syntax` on `match`, `type X = ...`, `def f[T]()` | Running an older interpreter than 3.12 | `python --version` | `uv python pin 3.12`; check the Docker base tag |
| `AttributeError: module 'typing' has no attribute 'Self'` | Python < 3.11 | as above | Upgrade or `from typing_extensions import Self` |
| Script works in shell, fails in cron/systemd | Different `PATH`/env; no venv activation | Log `sys.executable`, `os.environ` keys | Call `/path/to/.venv/bin/python -m pkg`, or `uv run --project /path` |
| `UnicodeDecodeError` reading files in container | Locale not set, ASCII default | `python -c "import locale; print(locale.getpreferredencoding())"` | Always `open(..., encoding="utf-8")`; set `PYTHONUTF8=1` |

---

## 3. Packaging and Build

| Symptom | Likely cause | Diagnose | Fix |
|---|---|---|---|
| Wheel builds but `platform-cli` command missing | `[project.scripts]` wrong module path | `uv run python -m zipfile -l dist/*.whl` and check `entry_points.txt` | `platform-cli = "platform_cli.cli:app"` |
| Wheel missing `py.typed` or data files | Build backend did not include them | List wheel contents | For hatchling, files under the package dir are included; check `.gitignore` is not excluding them (hatch respects VCS ignore) |
| `uv build` includes `tests/` in the sdist | Default sdist includes everything tracked | Inspect `.tar.gz` | `[tool.hatch.build.targets.sdist] exclude = ["tests/"]` |
| Version mismatch between `__version__` and `pyproject.toml` | Two sources of truth | grep | Use `importlib.metadata.version("platform-cli")` or `uv version` with dynamic versioning |
| Image builds every time from scratch (slow) | `COPY . .` before dependency install busts the cache | `docker build --progress=plain` shows which layer rebuilt | Copy `pyproject.toml uv.lock` first; `--mount=type=cache` |
| Image is 1 GB | Full `python:3.12` base, dev deps installed, build tools left in | `dive image` | Multi-stage; `--no-dev`; slim or distroless base |

---

## 4. asyncio

| Symptom | Likely cause | Diagnose | Fix |
|---|---|---|---|
| `RuntimeWarning: coroutine 'x' was never awaited` | Called `async def` without `await`/`create_task` | Warning names the coroutine | Add `await` |
| Service slow; one request blocks others | Blocking call in `async def` (`requests`, `time.sleep`, boto3, CPU loop) | `PYTHONASYNCIODEBUG=1` logs `Executing <Task...> took 2.1 seconds`; `py-spy dump` | `httpx.AsyncClient`, `asyncio.sleep`, `asyncio.to_thread`, async drivers |
| `RuntimeError: Event loop is closed` | Object (httpx client, DB engine) created in one loop, used after `asyncio.run` ended, or across pytest tests | Appears in teardown or second test | Create resources inside the running loop (lifespan / fixture); set pytest-asyncio `loop_scope="session"` if sharing |
| `RuntimeError: asyncio.run() cannot be called from a running event loop` | Nested `asyncio.run` (e.g., inside Jupyter or inside a FastAPI handler) | Traceback | `await` directly; in notebooks `nest_asyncio` or top-level `await` |
| `RuntimeError: There is no current event loop` | `asyncio.get_event_loop()` on a non-main thread or after loop close (deprecated) | Traceback | `asyncio.get_running_loop()` inside coroutines; `asyncio.run()` at entry |
| Task silently disappears / exception never shown | `create_task` result not stored; GC'd; or exception never retrieved | `Task exception was never retrieved` logged at exit | Keep references; use `TaskGroup`; await tasks |
| Shutdown hangs until SIGKILL | Task swallowing `CancelledError`, or infinite loop without `await` | `py-spy dump` shows the stuck frame | Re-raise `CancelledError`; add `await asyncio.sleep(0)` in tight loops; set `terminationGracePeriodSeconds` |
| `TimeoutError` with no useful message | `asyncio.timeout` fired | Wrap with context (`except TimeoutError: raise TimeoutError(f"fetch {url}")`) | Increase timeout or fix the slow dependency |
| Memory climbs with `gather` over large inputs | All coroutines created up front | Count tasks | Semaphore + chunking, or `asyncio.Queue` with N workers |
| `ExceptionGroup` raised from `TaskGroup` | One child failed; group wraps all | `except* ValueError as eg:` | Handle with `except*` or inspect `eg.exceptions` |

---

## 5. Typing and Tooling

| Symptom | Likely cause | Fix |
|---|---|---|
| `mypy: error: Skipping analyzing "kubernetes": module is installed, but missing library stubs or py.typed marker` | Untyped library | `[[tool.mypy.overrides]] module = ["kubernetes.*"] ignore_missing_imports = true` |
| `Incompatible types in assignment (expression has type "str \| None", variable has type "str")` | Missing None check | `if x is None: raise ...` before use |
| `Returning Any from function declared to return "dict[str, Any]"` | `json.loads` returns `Any` | Validate with Pydantic, or `cast(dict[str, Any], ...)` when sure |
| mypy passes locally, fails in CI | Different mypy version or missing stubs in CI | Pin `mypy` in dev group; `uv run mypy` in CI (same lock) |
| `ruff` and `ruff format` disagree (lint says line too long after format) | `E501` enabled with `line-length` lower than formatter | Set one `line-length`; formatter does not wrap strings, so `# noqa: E501` for long URLs |
| `pytest` collects 0 tests | Files not named `test_*.py`, or `testpaths` wrong | `pytest --collect-only -q` |
| `pytest-asyncio` tests skipped / `async def functions are not natively supported` | `asyncio_mode` not set | `asyncio_mode = "auto"` in `pyproject.toml` |
| `fixture 'x' not found` | Fixture defined in a module not imported; conftest missing | Put shared fixtures in `tests/conftest.py` |
| pre-commit `mypy` passes but CLI `mypy` fails | pre-commit runs mypy in its own env without your deps | `additional_dependencies` in the hook, or run `uv run mypy` via a `local` hook |

---

## 6. Containers and Kubernetes Runtime

| Symptom | Likely cause | Diagnose | Fix |
|---|---|---|---|
| No logs until the process exits | stdout buffered | Logs appear in bursts | `PYTHONUNBUFFERED=1` or `python -u` |
| Pod ignores SIGTERM, killed after grace period | Shell-form `ENTRYPOINT python app.py` (PID 1 is `sh`) or no signal handler | `kubectl get pod -o yaml` shows `reason: Error` after 30 s; `ps` shows `sh -c` | Exec-form `ENTRYPOINT ["python", "-m", "pkg"]`; handle SIGTERM |
| `PermissionError: [Errno 13] Permission denied: '/app/__pycache__'` | Read-only FS or non-root cannot write bytecode | Traceback | `PYTHONDONTWRITEBYTECODE=1` and pre-compile at build |
| `OSError: [Errno 30] Read-only file system: '/tmp/...'` | `readOnlyRootFilesystem` without a writable `/tmp` | Pod spec | Mount `emptyDir` at `/tmp` |
| Container runs as root despite `USER` | `USER` set before a later `RUN` that reset it; or `securityContext.runAsUser: 0` | `docker inspect --format '{{.Config.User}}'` | Put `USER` last; set `runAsNonRoot: true` in Pod |
| `OOMKilled`, no Python traceback | Memory limit hit; kernel kills before Python can react | `kubectl describe pod` shows `OOMKilled`, exit code 137 | Stream instead of materialising; raise limit from load test; `MALLOC_ARENA_MAX=2` |
| `exec format error` | Image built for arm64 (Apple Silicon) run on amd64 nodes | `docker image inspect --format '{{.Architecture}}'` | `docker buildx build --platform linux/amd64` or multi-arch |
| Works with `docker run`, fails in Kubernetes with `CrashLoopBackOff` | Missing env var; settings validation raises at import | `kubectl logs --previous` shows `pydantic ValidationError` | Provide the variable via ConfigMap/Secret; the fail-fast is correct |
| `distroless` image: `kubectl exec` fails `exec: "sh": not found` | No shell by design | - | `kubectl debug` with an ephemeral container |
| Slow start, readiness fails | Heavy imports + connection setup under tight `initialDelaySeconds` | `python -X importtime` | Lazy imports; `startupProbe` with `failureThreshold` |

---

## 7. Cloud and Kubernetes Client Errors

| Error message | Cause | Fix |
|---|---|---|
| `botocore.exceptions.NoCredentialsError: Unable to locate credentials` | No credential source in the chain | `aws sso login`, set `AWS_PROFILE`; in K8s, verify IRSA/Pod Identity annotation and `AWS_WEB_IDENTITY_TOKEN_FILE` |
| `botocore.exceptions.ClientError: An error occurred (AccessDenied) when calling the GetBucketEncryption operation` | Role lacks the permission | `aws sts get-caller-identity`; add `s3:GetEncryptionConfiguration` to the policy |
| `An error occurred (ExpiredToken)` / `ExpiredTokenException` | Session credentials expired | Re-login; for long jobs refresh via assume-role with a `RefreshableCredentials` or shorter tasks |
| `botocore.exceptions.ReadTimeoutError` | Default 60 s read timeout exceeded | `Config(read_timeout=...)`; check VPC endpoint/NAT path |
| `EndpointConnectionError: Could not connect to the endpoint URL` | Wrong region, no route (private subnet without endpoint) | Check `AWS_REGION`; add VPC endpoint |
| `Throttling` / `TooManyRequestsException` | API rate limit | `retries={"mode": "adaptive"}`; reduce concurrency |
| `kubernetes.config.config_exception.ConfigException: Invalid kube-config file. No configuration found.` | No kubeconfig and not in-cluster | Set `KUBECONFIG`; or mount a ServiceAccount token |
| `kubernetes.client.exceptions.ApiException: (403) Reason: Forbidden ... cannot list resource "pods"` | RBAC | `kubectl auth can-i list pods --as system:serviceaccount:<ns>:<sa>`; add Role/RoleBinding |
| `ApiException: (401) Unauthorized` | Expired token / wrong cluster | Refresh kubeconfig (`aws eks update-kubeconfig`) |
| `urllib3.exceptions.MaxRetryError ... Connection refused` | API server address wrong or network policy | Check `kubectl cluster-info`; in-cluster DNS `kubernetes.default.svc` |
| `ApiException: (409) Conflict ... the object has been modified` | Optimistic concurrency on update | Re-read then update, or use `patch` |
| `ssl.SSLCertVerificationError: certificate verify failed` | Corporate proxy / private CA not trusted | Set `SSL_CERT_FILE`/`REQUESTS_CA_BUNDLE` to the CA bundle; never `verify=False` |
| `httpx.ConnectTimeout` | DNS/route/firewall | `curl -v`; check NetworkPolicy; increase `connect` timeout only if the path is legitimately slow |

---

## 8. Pydantic

| Error | Cause | Fix |
|---|---|---|
| `pydantic_core._pydantic_core.ValidationError: 1 validation error for Settings database_url Field required` | Env var missing | Set `DATABASE_URL`; this is the desired fail-fast |
| `Extra inputs are not permitted` | `extra="forbid"` and a client sent an unknown field | Fix client, or relax to `extra="ignore"` for inbound webhooks you do not control |
| `Input should be a valid string` for an int | Type mismatch in strict context | Coerce explicitly or fix source |
| `PydanticUserError: ... @validator is deprecated` | v1 API on v2 | Migrate to `@field_validator` (see concepts file) |
| `AttributeError: 'IncidentCreate' object has no attribute 'dict'` | v1 method on v2 | `.model_dump()` |
| Settings read `.env` in tests unexpectedly | `env_file` default | `Settings(_env_file=None)` in tests |

---

## 9. Diagnostic Command Reference

```bash
# Which interpreter and environment?
uv run python -c "import sys, platform; print(sys.executable, sys.version, platform.platform())"
uv tree --depth 1

# Why is startup slow?
uv run python -X importtime -c "import platform_cli.cli" 2>&1 | sort -t'|' -k2 -n | tail

# What is a running process doing? (local or in a debug container)
uvx py-spy dump --pid <PID>
uvx py-spy top --pid <PID>

# Async debug
PYTHONASYNCIODEBUG=1 PYTHONTRACEMALLOC=1 uv run python -X dev -m platform_cli ...

# Container inspection
docker run --rm -it --entrypoint sh platform-cli:dev -c 'id; env | sort; python --version; ls -la /opt/venv/bin | head'
kubectl debug -it <POD> --image=python:3.12-slim --target=api -- sh
kubectl logs <POD> --previous --tail=200

# Dependency conflicts
uv tree --invert --package urllib3
uv lock --check
```
