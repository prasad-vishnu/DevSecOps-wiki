# Python: Interview Questions

| | |
|---|---|
| **Cluster** | 07 Backend & APIs |
| **Role** | Language foundation |
| **Prerequisites** | All files in this folder |
| **Target version** | Python 3.12+ |
| **Capstone phase** | Phase 1 (Months 1-4) |

> Thirty questions from junior to principal level, each with a model answer. These are the questions platform, SRE and AI-infrastructure interviews actually ask about Python: environments, typing, asyncio, security, containers and judgement about when not to use Python.

---

## Junior

### Q1. What is a virtual environment and why does every project need one? (Junior)

**Answer.** A virtual environment is an isolated directory containing a Python interpreter link and its own `site-packages`. It prevents dependency conflicts between projects (project A needs `httpx 0.27`, project B needs `0.25`) and protects the system interpreter, which the OS itself depends on. Modern Linux enforces this via PEP 668 (`externally-managed-environment`). With `uv`, the venv is created and synchronised automatically from `uv.lock`; you never activate it, you `uv run`.

### Q2. What is the difference between a list, a tuple, a set and a dict, and when would you pick each in ops code? (Junior)

**Answer.** List: ordered, mutable, allows duplicates; use for sequences you build up. Tuple: ordered, immutable, hashable if elements are; use for fixed records and dict keys. Set: unordered, unique, O(1) membership; use for "is this pod name in the allow-list" checks and set algebra (`desired - actual`). Dict: key-value, insertion-ordered since 3.7, O(1) lookup; use for indexing by ID. For structured records prefer a `dataclass` or Pydantic model over a dict of strings.

### Q3. Why must you never use a mutable default argument? (Junior)

**Answer.** Defaults are evaluated once at function definition, so `def f(items=[])` shares one list across all calls; items appended in one call appear in the next. Use `None` and create inside, or `field(default_factory=list)` in dataclasses. Ruff rule `B006` catches it.

### Q4. What does `if __name__ == "__main__":` do and why is it relevant to packaging? (Junior)

**Answer.** When a file is run directly, `__name__` is `"__main__"`; when imported, it is the module path. The guard lets a module be both importable (tests) and executable. In a packaged tool, prefer a `__main__.py` and a `[project.scripts]` entry point over the guard, so the code runs identically via `python -m pkg` and the installed console script.

### Q5. What is the GIL in one sentence, and what does it mean for a script that downloads 500 files? (Junior)

**Answer.** The Global Interpreter Lock lets only one thread execute Python bytecode at a time per process. Downloading is I/O-bound and the GIL is released during socket waits, so threads or asyncio give near-linear speedup; the GIL only limits CPU-bound parallelism, which needs processes.

### Q6. How do you read an environment variable safely? (Junior)

**Answer.** `os.environ["VAR"]` raises `KeyError` when missing (fail fast, correct for required config); `os.environ.get("VAR", "default")` for optional values. In real code, centralise this in a `pydantic-settings` `BaseSettings` class so types are validated once at startup and the rest of the code receives a typed object.

### Q7. What is the difference between `pip freeze > requirements.txt` and a lockfile? (Junior)

**Answer.** `pip freeze` dumps whatever happens to be installed, mixing direct and transitive dependencies with no record of what you asked for, and no hashes. A lockfile (`uv.lock`) records the declared dependencies separately (`pyproject.toml`) from the resolved exact versions and hashes, is cross-platform, and can be checked for staleness (`uv lock --check`).

---

## Mid-level

### Q8. Explain `async`/`await` and what happens when you call `time.sleep(5)` inside an `async def`. (Mid-level)

**Answer.** `async def` defines a coroutine that can suspend at `await` points, handing control back to the single-threaded event loop, which runs other ready coroutines or waits on sockets via epoll. `time.sleep(5)` is a blocking syscall that never yields, so the loop and every other coroutine freeze for 5 seconds. Use `await asyncio.sleep(5)`; for unavoidable blocking calls use `await asyncio.to_thread(fn)`. Ruff's `ASYNC` rules and `PYTHONASYNCIODEBUG=1` ("Executing ... took 5.0 seconds") expose this.

### Q9. How do you limit concurrency when fanning out 10,000 HTTP calls with asyncio? (Mid-level)

**Answer.** Wrap each call in `async with semaphore:` where `semaphore = asyncio.Semaphore(N)`, and schedule them under an `asyncio.TaskGroup` (or `gather`). The semaphore bounds in-flight requests; the TaskGroup gives structured cancellation if one fails. For very large inputs, prefer an `asyncio.Queue` with N worker tasks so you never create 10,000 coroutine objects at once. Add `asyncio.timeout` per call and retries with jitter only for idempotent requests.

### Q10. Dataclass or Pydantic model: how do you choose? (Mid-level)

**Answer.** Dataclasses are for internal, trusted data: zero validation cost, `frozen=True` for hashability, `slots=True` for memory. Pydantic models are for data crossing a trust boundary: HTTP bodies, env vars, YAML, LLM output. Pydantic v2 validates and coerces, generates JSON Schema (which FastAPI turns into OpenAPI), and `extra="forbid"` rejects unknown fields. Using Pydantic everywhere costs little but muddles the boundary; using dataclasses at the boundary is a security bug waiting to happen.

### Q11. What changed in Pydantic v2 and why does it matter? (Mid-level)

**Answer.** The core moved to Rust (`pydantic-core`), giving 5-50x faster validation. API renames: `@validator` to `@field_validator`, `.dict()` to `.model_dump()`, `parse_obj` to `model_validate`, `class Config` to `model_config = ConfigDict(...)`, `orm_mode` to `from_attributes`. Strict mode and `Annotated` constraints became first-class. It matters because FastAPI, LangChain and most AI tooling now require v2, and copying v1 snippets from old blog posts produces deprecation errors.

### Q12. Why is `subprocess.run(cmd, shell=True)` dangerous, and when is `shell=True` acceptable? (Mid-level)

**Answer.** With `shell=True` the string is interpreted by `/bin/sh`, so any interpolated value containing `;`, `|`, `$()` or backticks executes arbitrary commands: classic command injection. Pass a list (`["kubectl", "get", name]`), which `execve`s directly with each element as one literal argument. `shell=True` is acceptable only for a fixed, literal string with no interpolation that genuinely needs shell features (a pipe), and even then prefer composing two `Popen`s or doing the work in Python.

### Q13. How does a `with` statement work and how is it related to generators? (Mid-level)

**Answer.** `with` calls `__enter__` on entry and `__exit__` on exit, including on exceptions, guaranteeing cleanup. `contextlib.contextmanager` turns a generator into a context manager: code before `yield` is `__enter__`, code after is `__exit__`, wrapped in `try/finally`. This same shape is used by pytest fixtures and FastAPI `yield` dependencies, which is why understanding generators is a prerequisite for both.

### Q14. How would you structure a Python package for a platform CLI, and why the `src/` layout? (Mid-level)

**Answer.** `pyproject.toml` with `[project]`, `[project.scripts]`, tool config; `src/<pkg>/` containing `cli.py` (thin typer commands), `config.py` (Settings), domain modules (`k8s.py`, `aws.py`), `logging.py`, `py.typed`; `tests/` with `conftest.py`. The `src/` layout forces tests to import the *installed* package, which catches files missing from the wheel and accidental reliance on the current directory, and it prevents the project root from shadowing installed packages.

### Q15. Describe your approach to testing code that calls AWS and Kubernetes. (Mid-level)

**Answer.** Three layers: (1) unit tests with the SDK client injected and replaced by a fake (`moto`'s `@mock_aws` for AWS, which emulates services in-process; `unittest.mock.patch` at the point of use for the Kubernetes client); (2) HTTP-level mocks (`respx`) for services you call over httpx; (3) a small number of integration tests marked `integration` using Testcontainers (LocalStack, a kind cluster) run in CI but not on every save. Never let unit tests touch real accounts; make the client a parameter so it can be substituted without monkeypatching globals.

### Q16. What is structured logging and why does `print()` fail in production? (Mid-level)

**Answer.** Structured logging emits one JSON object per event with typed fields (`event`, `level`, `timestamp`, `request_id`, `trace_id`, domain fields) so that Loki/Elasticsearch can filter and aggregate without regex. `print()` has no level, no timestamp, no context, interleaves across threads and workers, and may be buffered. `structlog` with `contextvars` binds request-scoped fields once so that every line in a request carries them without passing a logger through every function.

### Q17. What does `uv` give you over pip, venv, pyenv and pipx? (Mid-level)

**Answer.** One binary that installs interpreters (`uv python install`), creates and syncs venvs (`uv sync`), resolves and locks with hashes (`uv lock`) 10-100x faster than pip thanks to a Rust resolver and global cache, runs commands in the environment (`uv run`), installs global tools (`uv tool install` replacing pipx), and supports PEP 723 inline script metadata. The operational wins are reproducibility (`--frozen`), speed in CI and Docker, and one tool for the whole team.

---

## Senior

### Q18. Walk through a production Dockerfile for a Python service and justify each decision. (Senior)

**Answer.** Build stage from `ghcr.io/astral-sh/uv:<ver>-python3.12-bookworm-slim` pinned by digest; `UV_COMPILE_BYTECODE=1` to pre-compile; copy `pyproject.toml` and `uv.lock` first and `uv sync --frozen --no-dev --no-install-project` so the dependency layer caches; then copy `src` and sync again. Runtime stage from `python:3.12-slim-bookworm` (or distroless) with the venv copied; `PYTHONUNBUFFERED=1`, `PYTHONDONTWRITEBYTECODE=1`; create a UID 65532 user and `USER 65532`; exec-form `ENTRYPOINT ["python", "-m", "pkg"]` so PID 1 receives SIGTERM; no `HEALTHCHECK` (Kubernetes probes own that); OCI labels for provenance. No Alpine (musl wheel problems), no `pip install` in the runtime stage, no root.

### Q19. A FastAPI pod's p99 latency jumps from 50 ms to 3 s under load while CPU stays at 20%. How do you diagnose it? (Senior)

**Answer.** Low CPU with high latency means waiting, not computing: the event loop is blocked or a pool is exhausted. `kubectl debug` with `py-spy dump --pid 1` shows every thread's stack at that instant; a frame in `requests`/`time.sleep`/`boto3`/sync DB driver inside a coroutine is the smoking gun. `PYTHONASYNCIODEBUG=1` logs slow callbacks. Check DB pool metrics (waiting for a connection), `httpx` pool limits, and threadpool saturation (default 40 threads for sync handlers). Fix: async drivers, `run_in_threadpool`, larger pools, more workers. Confirm with a load test and OTel spans showing where the time went.

### Q20. How do you make a long-running Python worker shut down gracefully in Kubernetes? (Senior)

**Answer.** Kubernetes sends SIGTERM, waits `terminationGracePeriodSeconds` (default 30 s), then SIGKILL. The process must be PID 1 or forward signals (exec-form entrypoint). Register a handler via `loop.add_signal_handler(SIGTERM, stop_event.set)`; the main loop checks the event, stops taking new work, awaits in-flight tasks with a deadline (`asyncio.timeout`), closes pools in `finally`, and exits 0. For HTTP services also fail readiness immediately so the endpoint is removed from the Service before draining. Never swallow `CancelledError`. Test it: `docker kill --signal=TERM` and watch the exit code and the time to exit.

### Q21. Explain Python's memory behaviour in containers and how you size limits. (Senior)

**Answer.** CPython uses reference counting plus a cycle collector; freed small objects go back to pymalloc arenas, which are rarely returned to the OS, so RSS tracks peak usage. Materialising large lists (paginated API results, file contents) sets the peak. Size `requests` to steady-state and `limits` to measured peak times 1.2-1.3 from a load test; an `OOMKilled` Python gives no traceback. Reduce peaks with generators and streaming, `MALLOC_ARENA_MAX=2` for multi-threaded glibc fragmentation, and fewer uvicorn workers with higher per-worker limits rather than many small ones. Diagnose with `tracemalloc` or `memray` in staging.

### Q22. What is your dependency security posture for Python services? (Senior)

**Answer.** Declared deps in `pyproject.toml`, exact resolution in `uv.lock` with hashes, `uv lock --check` in CI, `uv sync --frozen` in Docker. A private index proxy (CodeArtifact/Artifactory) to quarantine new packages. `pip-audit` or Snyk on every PR against `uv export` output; Renovate/Dependabot for upgrades. Syft SBOM per image into Dependency-Track for continuous monitoring. Trivy on the image for OS CVEs. Code rules via `ruff -S`/bandit/Semgrep: no `eval`, `pickle`, `yaml.load`, `shell=True`. Minimal runtime deps (`--no-dev`). Base images pinned by digest and rebuilt weekly.

### Q23. Compare threads, asyncio and multiprocessing for a tool that must (a) call 2,000 cloud APIs, (b) parse 50 GB of logs. (Senior)

**Answer.** (a) I/O-bound: asyncio with `httpx.AsyncClient`/`aioboto3`, a semaphore of 20-50, and per-call timeouts; threads via `ThreadPoolExecutor` are acceptable with sync SDKs (`boto3`) and simpler to retrofit. (b) CPU-bound: the GIL makes threads useless; use `ProcessPoolExecutor` with one process per core, chunk the input by file or byte range, and consider whether the parsing belongs in Go, DuckDB or `polars` instead. Mixing: an asyncio coordinator can offload CPU chunks via `loop.run_in_executor(process_pool, fn)`.

### Q24. How do you make an automation script safe to run against production accounts? (Senior)

**Answer.** Dry-run by default with an explicit `--apply`; print and require confirmation of the target context (`sts get-caller-identity`, `kubectl config current-context`) with a guard for production; mandatory selectors so an empty filter cannot match everything; idempotent operations (apply/patch, check-then-act); timeouts and bounded retries only on idempotent calls; structured logs with a run ID; least-privilege roles separated into read-only and write; exit codes for CI; no interactive prompts when not a TTY; audit log to an immutable store. Tests include a "wrong context" test proving the guard works.

---

## Principal

### Q25. When should a platform team *not* write something in Python? (Principal)

**Answer.** When the deliverable must be a single static binary distributed to many hosts or embedded in minimal images (Go); when it is a Kubernetes controller or operator where the ecosystem (controller-runtime, kubebuilder, client-go informers) is Go-native; when it is CPU-bound at scale (log processing, proxies, high-throughput data planes); when it is under 50 lines of glue calling existing CLIs inside CI or a Dockerfile (Bash with shellcheck). Python wins for anything with SDK-heavy cloud logic, JSON/YAML manipulation, tests, LLM/agent code and evals. A principal's job is to set the budget: Python for tools and services, Go for controllers, Bash for glue, and to say no to a fourth.

### Q26. Design the Python toolchain standards for a 40-engineer platform org. (Principal)

**Answer.** Single interpreter line (3.12) pinned in `.python-version` and base images; `uv` everywhere with `uv.lock` committed; a shared `pyproject.toml` template with `ruff` (E,W,F,I,B,UP,S,ASYNC,PTH,RUF), `mypy --strict`, `pytest` with coverage threshold; pre-commit with ruff, mypy, gitleaks; a cookiecutter/Backstage template producing the `src/` layout, Dockerfile, CI workflow and OTel wiring; a private index proxy; a reusable GitHub Actions workflow (`python-ci.yaml`) that every repo calls so gates are upgraded centrally; an internal `platform-core` library for logging, settings, HTTP client, AWS/K8s helpers with `py.typed`; and ADRs recording each choice. Enforce via the template and CI rather than wiki pages.

### Q27. Explain how FastAPI's dependency injection and pytest fixtures are "the same idea". (Principal)

**Answer.** Both are declarative resource graphs resolved by a framework: you declare what a function needs by name/type, the framework builds the dependency tree, instantiates each node once per scope (per request in FastAPI, per function/module/session in pytest), supports setup/teardown via generator `yield`, caches results within the scope, and allows substitution (`app.dependency_overrides`, fixture overriding in a nearer `conftest.py`). Recognising this lets you design services whose handlers are trivially testable: the dependency boundary is the test boundary. The same shape appears in Spring's IoC container, with scopes and `@MockBean`.

### Q28. How would you detect and prevent event-loop blocking across dozens of async Python services? (Principal)

**Answer.** Prevent: `ruff --select ASYNC` in the shared pre-commit, a platform HTTP client library that only exposes async methods, code review guidance, and a lint forbidding `requests`/`time.sleep`/sync DB drivers in service repos. Detect: enable `PYTHONASYNCIODEBUG` in staging to log slow callbacks; export an "event loop lag" metric (schedule a `call_later` every 100 ms and measure drift) to Prometheus with an alert at >200 ms; OTel spans reveal handlers whose duration is spent outside awaited children. Respond: `py-spy dump` runbook in the on-call docs. Educate with a lab reproducing the stall.

### Q29. Argue for or against GraalVM-style native compilation for Python (Nuitka, PyInstaller, Cython) in a platform context. (Principal)

**Answer.** Mostly against for services: containers already solve distribution, startup is sub-second, and the tools complicate debugging, profiling and dependency scanning (SBOM tools see a blob). Cython/`mypyc` are justified for a hot CPU kernel inside a library. PyInstaller/Nuitka are justified for a developer-facing CLI that must run on laptops without a Python installation, though `uv tool install` and `uvx` have largely removed that need. Prefer spending the effort on lazy imports, `UV_COMPILE_BYTECODE`, and moving CPU-heavy paths to Go.

### Q30. A team wants to run the FastAPI service with 32 uvicorn workers on a 4-core node "for concurrency". Respond. (Principal)

**Answer.** Workers are processes; 32 on 4 cores means 8x oversubscription, context switching, 32 DB pools (connection explosion at Postgres), 32x memory, and worse tail latency. Async concurrency comes from the event loop, not from processes; one worker handles thousands of concurrent I/O-bound requests. Guidance: workers = CPU cores (or cores + 1), scale horizontally with the HPA, size the DB pool per worker so total connections stay under the DB limit (use PgBouncer if needed), and if a handler is CPU-bound, fix the handler or move the work. Show them `py-spy top` and the connection count before and after.
