# Python: Hands-On Labs

| | |
|---|---|
| **Cluster** | 07 Backend & APIs |
| **Role** | Language foundation |
| **Prerequisites** | [01-installation-and-setup.md](01-installation-and-setup.md), [02-core-concepts.md](02-core-concepts.md), Docker running, a kind cluster for Lab 5 (`kind create cluster --name lab`) |
| **Target version** | Python 3.12+, uv 0.5+ |
| **Capstone phase** | Phase 1 (Months 1-4) |

> Six progressive labs that build `tools/platform-cli`, the capstone's Python automation package. Lab 1 produces a typed, tested CLI; Lab 6 ships it as a non-root distroless container that lists Kubernetes pods and S3 buckets concurrently. Each lab is self-contained but the code accumulates, so do them in order.

| Lab | Topic | Time |
|---|---|---|
| 1 | Project skeleton, typed CLI, first tests | 40 min |
| 2 | Pydantic models and settings; parametrised tests | 40 min |
| 3 | asyncio fan-out with httpx, semaphores, timeouts; mocking with respx | 60 min |
| 4 | boto3 with moto; subprocess done safely | 50 min |
| 5 | Kubernetes client against kind; structured logging | 60 min |
| 6 | Profiling, security scan, multi-stage distroless Docker image | 60 min |

---

## Lab 1: Project Skeleton and a Typed CLI

**Objective.** Create `platform-cli` with the `src/` layout, a `typer` CLI with one command, `ruff`/`mypy`/`pytest` passing, and `pre-commit` installed.

**Prerequisites.** `uv` installed.

**Time.** 40 minutes.

### Steps

1. Scaffold and add dependencies.

   ```bash
   uv init --package platform-cli && cd platform-cli
   uv python pin 3.12
   uv add typer pydantic pydantic-settings structlog httpx
   uv add --dev pytest pytest-asyncio pytest-cov ruff mypy pre-commit respx "moto[s3]" "boto3-stubs[s3,sts]"
   ```

2. Replace `pyproject.toml`'s tool sections with the configuration from [01-installation-and-setup.md](01-installation-and-setup.md) section 5 (keep the generated `[project]` dependencies).

3. Write the CLI. `src/platform_cli/cli.py`:

   ```python
   from typing import Annotated

   import typer

   from platform_cli import __version__

   app = typer.Typer(no_args_is_help=True, help="Platform automation CLI for the capstone.")


   def _version_callback(value: bool) -> None:
       if value:
           typer.echo(f"platform-cli {__version__}")
           raise typer.Exit()


   @app.callback()
   def main(
       version: Annotated[
           bool, typer.Option("--version", callback=_version_callback, is_eager=True, help="Show version")
       ] = False,
   ) -> None:
       """Platform automation CLI."""


   @app.command()
   def greet(name: Annotated[str, typer.Argument(help="Who to greet")] = "platform") -> None:
       """Print a greeting (smoke test command)."""
       typer.echo(f"hello, {name}")
   ```

   `src/platform_cli/__init__.py`:

   ```python
   __version__ = "0.1.0"
   ```

   `src/platform_cli/__main__.py`:

   ```python
   from platform_cli.cli import app

   app()
   ```

4. Write the first test. `tests/test_cli.py`:

   ```python
   from typer.testing import CliRunner

   from platform_cli.cli import app

   runner = CliRunner()


   def test_version() -> None:
       result = runner.invoke(app, ["--version"])
       assert result.exit_code == 0
       assert result.stdout.startswith("platform-cli ")


   def test_greet_default() -> None:
       result = runner.invoke(app, ["greet"])
       assert result.exit_code == 0
       assert result.stdout.strip() == "hello, platform"
   ```

5. Run the toolchain.

   ```bash
   uv run ruff check . && uv run ruff format . && uv run mypy && uv run pytest -q
   ```

   Expected output (tail):

   ```text
   All checks passed!
   3 files reformatted, 2 files left unchanged
   Success: no issues found in 4 source files
   ..                                                                   [100%]
   2 passed in 0.21s
   ```

6. Install pre-commit using the config from the setup file and run it once.

   ```bash
   uv run pre-commit install && uv run pre-commit run --all-files
   ```

### Verify

```bash
uv run platform-cli --version
uv run python -m platform_cli greet sre
```

```text
platform-cli 0.1.0
hello, sre
```

### Cleanup

None; keep the project for the next labs.

### What you learned

The `src/` layout, how `uv run` guarantees the environment, how `typer` builds a CLI from type hints, and how `CliRunner` tests a CLI without subprocesses.

---

## Lab 2: Pydantic Models, Settings and Parametrised Tests

**Objective.** Add a validated `Settings` class and an `Incident` model (the same shape the FastAPI service will use), and test validation rules with `parametrize`.

**Prerequisites.** Lab 1.

**Time.** 40 minutes.

### Steps

1. `src/platform_cli/config.py`:

   ```python
   from functools import lru_cache

   from pydantic import Field, SecretStr
   from pydantic_settings import BaseSettings, SettingsConfigDict


   class Settings(BaseSettings):
       model_config = SettingsConfigDict(env_prefix="PLATFORM_", env_file=".env", extra="ignore")

       env: str = "dev"
       log_level: str = "INFO"
       log_json: bool = True
       aws_region: str = "eu-west-1"
       k8s_namespace: str = "default"
       http_timeout_s: float = Field(default=5.0, gt=0, le=60)
       concurrency: int = Field(default=10, ge=1, le=100)
       api_token: SecretStr | None = None


   @lru_cache
   def get_settings() -> Settings:
       return Settings()
   ```

2. `src/platform_cli/models.py`:

   ```python
   from datetime import datetime
   from enum import StrEnum
   from typing import Annotated
   from uuid import UUID

   from pydantic import BaseModel, ConfigDict, Field, model_validator


   class Severity(StrEnum):
       SEV1 = "SEV1"
       SEV2 = "SEV2"
       SEV3 = "SEV3"
       SEV4 = "SEV4"


   class IncidentCreate(BaseModel):
       model_config = ConfigDict(str_strip_whitespace=True, extra="forbid")

       title: Annotated[str, Field(min_length=1, max_length=200)]
       description: Annotated[str | None, Field(max_length=4000)] = None
       severity: Severity
       service: Annotated[str, Field(min_length=1, max_length=100, pattern=r"^[a-z0-9-]+$")]

       @model_validator(mode="after")
       def sev1_requires_description(self) -> "IncidentCreate":
           if self.severity is Severity.SEV1 and not self.description:
               raise ValueError("SEV1 incidents require a description")
           return self


   class Incident(IncidentCreate):
       id: UUID
       status: str
       created_at: datetime
       updated_at: datetime
   ```

3. `tests/test_models.py`:

   ```python
   import pytest
   from pydantic import ValidationError

   from platform_cli.models import IncidentCreate, Severity

   VALID = {"title": "API latency", "severity": "SEV2", "service": "incident-api"}


   def test_valid_incident_strips_whitespace() -> None:
       inc = IncidentCreate.model_validate({**VALID, "title": "  API latency  "})
       assert inc.title == "API latency"
       assert inc.severity is Severity.SEV2


   @pytest.mark.parametrize(
       ("field", "value", "msg"),
       [
           ("title", "", "at least 1 character"),
           ("title", "x" * 201, "at most 200 characters"),
           ("severity", "SEV5", "Input should be 'SEV1', 'SEV2', 'SEV3' or 'SEV4'"),
           ("service", "Incident API", "String should match pattern"),
       ],
   )
   def test_invalid_fields(field: str, value: str, msg: str) -> None:
       with pytest.raises(ValidationError) as exc:
           IncidentCreate.model_validate({**VALID, field: value})
       assert msg in str(exc.value)


   def test_sev1_requires_description() -> None:
       with pytest.raises(ValidationError, match="SEV1 incidents require a description"):
           IncidentCreate.model_validate({**VALID, "severity": "SEV1"})


   def test_unknown_field_rejected() -> None:
       with pytest.raises(ValidationError, match="Extra inputs are not permitted"):
           IncidentCreate.model_validate({**VALID, "priority": "high"})
   ```

4. `tests/test_config.py` using `monkeypatch`:

   ```python
   import pytest

   from platform_cli.config import Settings


   def test_settings_from_env(monkeypatch: pytest.MonkeyPatch) -> None:
       monkeypatch.setenv("PLATFORM_ENV", "prod")
       monkeypatch.setenv("PLATFORM_CONCURRENCY", "25")
       monkeypatch.setenv("PLATFORM_API_TOKEN", "not-a-real-token")
       s = Settings(_env_file=None)
       assert s.env == "prod"
       assert s.concurrency == 25
       assert "not-a-real-token" not in repr(s)          # SecretStr is masked


   def test_settings_rejects_out_of_range(monkeypatch: pytest.MonkeyPatch) -> None:
       monkeypatch.setenv("PLATFORM_CONCURRENCY", "0")
       with pytest.raises(ValueError):
           Settings(_env_file=None)
   ```

5. Run.

   ```bash
   uv run pytest -q
   ```

   ```text
   .........                                                            [100%]
   9 passed in 0.34s
   ```

### Verify

```bash
uv run python -c "from platform_cli.models import IncidentCreate; print(IncidentCreate.model_json_schema()['properties']['severity'])"
```

```text
{'$ref': '#/$defs/Severity'}
```

### What you learned

Pydantic v2 validation at the boundary, `SecretStr` masking, `env_prefix` settings, `parametrize` for table-driven tests, and `monkeypatch` for env-driven tests.

---

## Lab 3: asyncio Fan-Out with httpx, Semaphores and Timeouts

**Objective.** Implement `platform-cli http check` that probes N URLs concurrently with bounded concurrency and per-request timeouts, and test it without the network using `respx`.

**Prerequisites.** Lab 2.

**Time.** 60 minutes.

### Steps

1. `src/platform_cli/http.py`:

   ```python
   import asyncio
   import time
   from dataclasses import dataclass

   import httpx


   @dataclass(frozen=True, slots=True)
   class ProbeResult:
       url: str
       status: int | None
       latency_ms: float
       error: str | None = None

       @property
       def ok(self) -> bool:
           return self.status is not None and 200 <= self.status < 400


   async def probe(client: httpx.AsyncClient, url: str, sem: asyncio.Semaphore, timeout_s: float) -> ProbeResult:
       start = time.perf_counter()
       async with sem:
           try:
               async with asyncio.timeout(timeout_s):
                   r = await client.get(url)
               return ProbeResult(url, r.status_code, (time.perf_counter() - start) * 1000)
           except TimeoutError:
               return ProbeResult(url, None, (time.perf_counter() - start) * 1000, "timeout")
           except httpx.HTTPError as exc:
               return ProbeResult(url, None, (time.perf_counter() - start) * 1000, type(exc).__name__)


   async def probe_all(urls: list[str], *, concurrency: int = 10, timeout_s: float = 5.0) -> list[ProbeResult]:
       sem = asyncio.Semaphore(concurrency)
       async with httpx.AsyncClient(follow_redirects=False) as client:
           async with asyncio.TaskGroup() as tg:
               tasks = [tg.create_task(probe(client, u, sem, timeout_s)) for u in urls]
       return [t.result() for t in tasks]
   ```

2. Add the command to `cli.py`:

   ```python
   import asyncio
   from pathlib import Path

   from platform_cli.config import get_settings
   from platform_cli.http import probe_all

   http_app = typer.Typer(help="HTTP probes")
   app.add_typer(http_app, name="http")


   @http_app.command("check")
   def http_check(
       urls: Annotated[list[str] | None, typer.Argument(help="URLs to probe")] = None,
       file: Annotated[Path | None, typer.Option("--file", "-f", help="File with one URL per line")] = None,
       concurrency: Annotated[int | None, typer.Option("--concurrency", "-c")] = None,
   ) -> None:
       """Probe URLs concurrently and report status and latency."""
       settings = get_settings()
       targets = list(urls or [])
       if file:
           targets += [l.strip() for l in file.read_text().splitlines() if l.strip() and not l.startswith("#")]
       if not targets:
           raise typer.BadParameter("provide URLs or --file")
       results = asyncio.run(
           probe_all(targets, concurrency=concurrency or settings.concurrency, timeout_s=settings.http_timeout_s)
       )
       failures = 0
       for r in results:
           mark = "OK " if r.ok else "ERR"
           failures += not r.ok
           typer.echo(f"{mark} {r.status or '-':>4} {r.latency_ms:8.1f}ms {r.url} {r.error or ''}")
       raise typer.Exit(code=1 if failures else 0)
   ```

3. Test with `respx`. `tests/test_http.py`:

   ```python
   import asyncio

   import httpx
   import pytest
   import respx

   from platform_cli.http import probe_all


   @respx.mock
   async def test_probe_all_mixed_results() -> None:
       respx.get("https://svc-a.example.com/healthz").mock(return_value=httpx.Response(200))
       respx.get("https://svc-b.example.com/healthz").mock(return_value=httpx.Response(503))
       respx.get("https://svc-c.example.com/healthz").mock(side_effect=httpx.ConnectError("refused"))

       results = await probe_all(
           ["https://svc-a.example.com/healthz", "https://svc-b.example.com/healthz", "https://svc-c.example.com/healthz"],
           concurrency=2,
       )
       by_url = {r.url: r for r in results}
       assert by_url["https://svc-a.example.com/healthz"].ok
       assert by_url["https://svc-b.example.com/healthz"].status == 503
       assert by_url["https://svc-c.example.com/healthz"].error == "ConnectError"


   @respx.mock
   async def test_probe_times_out() -> None:
       async def slow(request: httpx.Request) -> httpx.Response:
           await asyncio.sleep(0.5)
           return httpx.Response(200)

       respx.get("https://slow.example.com/").mock(side_effect=slow)
       [r] = await probe_all(["https://slow.example.com/"], timeout_s=0.05)
       assert r.error == "timeout"
       assert r.status is None


   @respx.mock
   async def test_concurrency_is_bounded() -> None:
       in_flight = 0
       peak = 0

       async def handler(request: httpx.Request) -> httpx.Response:
           nonlocal in_flight, peak
           in_flight += 1
           peak = max(peak, in_flight)
           await asyncio.sleep(0.01)
           in_flight -= 1
           return httpx.Response(200)

       respx.get(url__regex=r"https://svc-\d+\.example\.com/").mock(side_effect=handler)
       await probe_all([f"https://svc-{i}.example.com/" for i in range(20)], concurrency=3)
       assert peak <= 3
   ```

4. Run.

   ```bash
   uv run pytest -q tests/test_http.py
   ```

   ```text
   ...                                                                  [100%]
   3 passed in 0.41s
   ```

5. Try it for real against a local server.

   ```bash
   uv run python -m http.server 8080 --directory . &
   printf 'http://127.0.0.1:8080/\nhttp://127.0.0.1:8080/missing\nhttp://127.0.0.1:9/\n' > urls.txt
   uv run platform-cli http check --file urls.txt --concurrency 3; echo "exit=$?"
   kill %1
   ```

   Expected output:

   ```text
   OK   200      3.1ms http://127.0.0.1:8080/
   ERR  404      2.4ms http://127.0.0.1:8080/missing
   ERR     -      1.2ms http://127.0.0.1:9/ ConnectError
   exit=1
   ```

### Verify

The third test proves the semaphore bounds concurrency; `pytest -v` shows all three tests as `async` tests picked up by `asyncio_mode = "auto"`.

### What you learned

`TaskGroup` structured concurrency, `Semaphore` back-pressure, `asyncio.timeout`, turning exceptions into data at the boundary, and route-level HTTP mocking with `respx`.

---

## Lab 4: boto3 with moto, and Safe subprocess

**Objective.** Add `platform-cli s3 audit` listing buckets lacking default encryption, tested with `moto`; add a safe `kubectl` wrapper tested with a fake binary.

**Prerequisites.** Lab 3. No AWS account needed.

**Time.** 50 minutes.

### Steps

1. `uv add boto3` then `src/platform_cli/aws.py`:

   ```python
   from dataclasses import dataclass

   import boto3
   from botocore.config import Config
   from botocore.exceptions import ClientError
   from mypy_boto3_s3 import S3Client


   @dataclass(frozen=True, slots=True)
   class BucketAudit:
       name: str
       encrypted: bool
       versioned: bool


   def s3_client(region: str) -> S3Client:
       return boto3.client("s3", config=Config(region_name=region, retries={"max_attempts": 5, "mode": "adaptive"}))


   def audit_buckets(s3: S3Client) -> list[BucketAudit]:
       out: list[BucketAudit] = []
       for b in s3.list_buckets()["Buckets"]:
           name = b["Name"]
           try:
               s3.get_bucket_encryption(Bucket=name)
               encrypted = True
           except ClientError as exc:
               if exc.response["Error"]["Code"] != "ServerSideEncryptionConfigurationNotFoundError":
                   raise
               encrypted = False
           versioned = s3.get_bucket_versioning(Bucket=name).get("Status") == "Enabled"
           out.append(BucketAudit(name, encrypted, versioned))
       return out
   ```

2. `tests/test_aws.py`:

   ```python
   import boto3
   import pytest
   from moto import mock_aws

   from platform_cli.aws import audit_buckets


   @pytest.fixture
   def aws_env(monkeypatch: pytest.MonkeyPatch) -> None:
       # moto needs some credentials present; these are dummies, never real.
       monkeypatch.setenv("AWS_ACCESS_KEY_ID", "testing")
       monkeypatch.setenv("AWS_SECRET_ACCESS_KEY", "testing")
       monkeypatch.setenv("AWS_DEFAULT_REGION", "eu-west-1")


   @mock_aws
   def test_audit_reports_encryption_and_versioning(aws_env: None) -> None:
       s3 = boto3.client("s3", region_name="eu-west-1")
       loc = {"LocationConstraint": "eu-west-1"}
       s3.create_bucket(Bucket="good-bucket", CreateBucketConfiguration=loc)
       s3.put_bucket_encryption(
           Bucket="good-bucket",
           ServerSideEncryptionConfiguration={"Rules": [{"ApplyServerSideEncryptionByDefault": {"SSEAlgorithm": "aws:kms"}}]},
       )
       s3.put_bucket_versioning(Bucket="good-bucket", VersioningConfiguration={"Status": "Enabled"})
       s3.create_bucket(Bucket="bad-bucket", CreateBucketConfiguration=loc)

       result = {a.name: a for a in audit_buckets(s3)}
       assert result["good-bucket"].encrypted and result["good-bucket"].versioned
       assert not result["bad-bucket"].encrypted and not result["bad-bucket"].versioned
   ```

3. Safe subprocess wrapper. `src/platform_cli/shell.py`:

   ```python
   import shlex
   import subprocess
   from pathlib import Path


   class CommandError(RuntimeError):
       def __init__(self, cmd: list[str], code: int, stderr: str) -> None:
           super().__init__(f"{shlex.join(cmd)} exited {code}: {stderr.strip()}")
           self.cmd, self.code, self.stderr = cmd, code, stderr


   def run(cmd: list[str], *, timeout: float = 60, cwd: Path | None = None, env: dict[str, str] | None = None) -> str:
       proc = subprocess.run(cmd, capture_output=True, text=True, timeout=timeout, cwd=cwd, env=env, check=False)
       if proc.returncode != 0:
           raise CommandError(cmd, proc.returncode, proc.stderr)
       return proc.stdout


   def kubectl(*args: str, namespace: str | None = None, timeout: float = 30) -> str:
       cmd = ["kubectl", *args]
       if namespace:
           cmd += ["--namespace", namespace]
       return run(cmd, timeout=timeout)
   ```

4. Test with a fake `kubectl` on `PATH`. `tests/test_shell.py`:

   ```python
   import os
   import stat
   from pathlib import Path

   import pytest

   from platform_cli.shell import CommandError, kubectl, run


   @pytest.fixture
   def fake_kubectl(tmp_path: Path, monkeypatch: pytest.MonkeyPatch) -> Path:
       script = tmp_path / "kubectl"
       script.write_text('#!/bin/sh\nif [ "$1" = "fail" ]; then echo "boom" >&2; exit 3; fi\necho "args: $*"\n')
       script.chmod(script.stat().st_mode | stat.S_IEXEC)
       monkeypatch.setenv("PATH", f"{tmp_path}{os.pathsep}{os.environ['PATH']}")
       return script


   def test_kubectl_passes_namespace(fake_kubectl: Path) -> None:
       out = kubectl("get", "pods", namespace="platform")
       assert out.strip() == "args: get pods --namespace platform"


   def test_nonzero_exit_raises(fake_kubectl: Path) -> None:
       with pytest.raises(CommandError, match="exited 3: boom"):
           kubectl("fail")


   def test_no_shell_injection(fake_kubectl: Path) -> None:
       out = run(["kubectl", "get", "pod", "x; rm -rf /"])
       assert "rm -rf" in out            # passed as one literal argument, never interpreted
   ```

   On Windows this fixture needs a `.cmd` shim; run these tests in WSL or CI.

5. Run everything.

   ```bash
   uv run pytest -q
   ```

   ```text
   ...............                                                      [100%]
   15 passed in 1.92s
   ```

### What you learned

`moto` gives you a free AWS for tests; typed boto3 via `boto3-stubs`; how to test subprocess wrappers without mocking `subprocess` itself; and a concrete demonstration that list-form commands cannot be injected.

---

## Lab 5: Kubernetes Client Against kind, with Structured Logging

**Objective.** Add `platform-cli pods list --namespace <NS>` using the official `kubernetes` client, output JSON logs with `structlog`, and run it against a local kind cluster.

**Prerequisites.** Lab 4; `kind create cluster --name lab`; `kubectl get nodes` works.

**Time.** 60 minutes.

### Steps

1. `uv add kubernetes` and create `src/platform_cli/logging.py` with `configure_logging` from [02-core-concepts.md](02-core-concepts.md) section 7.

2. `src/platform_cli/k8s.py`:

   ```python
   from dataclasses import dataclass

   import structlog
   from kubernetes import client, config

   log = structlog.get_logger(__name__)


   @dataclass(frozen=True, slots=True)
   class PodSummary:
       namespace: str
       name: str
       phase: str
       restarts: int
       node: str | None


   def load_config() -> None:
       try:
           config.load_incluster_config()
           log.debug("k8s_config", source="in-cluster")
       except config.ConfigException:
           config.load_kube_config()
           log.debug("k8s_config", source="kubeconfig")


   def list_pods(namespace: str, selector: str | None = None) -> list[PodSummary]:
       v1 = client.CoreV1Api()
       resp = v1.list_namespaced_pod(namespace, label_selector=selector or "", timeout_seconds=10)
       pods = [
           PodSummary(
               namespace=p.metadata.namespace,
               name=p.metadata.name,
               phase=p.status.phase or "Unknown",
               restarts=sum(cs.restart_count for cs in (p.status.container_statuses or [])),
               node=p.spec.node_name,
           )
           for p in resp.items
       ]
       log.info("pods_listed", namespace=namespace, count=len(pods))
       return pods
   ```

3. Add the command in `cli.py`:

   ```python
   import json
   from dataclasses import asdict

   from platform_cli import k8s
   from platform_cli.logging import configure_logging

   pods_app = typer.Typer(help="Kubernetes pods")
   app.add_typer(pods_app, name="pods")


   @pods_app.command("list")
   def pods_list(
       namespace: Annotated[str | None, typer.Option("--namespace", "-n", envvar="PLATFORM_K8S_NAMESPACE")] = None,
       selector: Annotated[str | None, typer.Option("--selector", "-l")] = None,
       output: Annotated[str, typer.Option("--output", "-o")] = "table",
   ) -> None:
       """List pods with phase and restart counts."""
       settings = get_settings()
       configure_logging(settings.log_level, settings.log_json)
       k8s.load_config()
       pods = k8s.list_pods(namespace or settings.k8s_namespace, selector)
       if output == "json":
           typer.echo(json.dumps([asdict(p) for p in pods], indent=2))
           return
       for p in pods:
           typer.echo(f"{p.name:<50} {p.phase:<10} restarts={p.restarts:<3} node={p.node}")
   ```

4. Unit test with a fake API (no cluster). `tests/test_k8s.py`:

   ```python
   from types import SimpleNamespace
   from unittest.mock import patch

   from platform_cli.k8s import list_pods


   def _pod(name: str, phase: str, restarts: int) -> SimpleNamespace:
       return SimpleNamespace(
           metadata=SimpleNamespace(namespace="platform", name=name),
           status=SimpleNamespace(phase=phase, container_statuses=[SimpleNamespace(restart_count=restarts)]),
           spec=SimpleNamespace(node_name="node-1"),
       )


   def test_list_pods_summarises() -> None:
       fake = SimpleNamespace(items=[_pod("api-1", "Running", 0), _pod("api-2", "CrashLoopBackOff", 7)])
       with patch("platform_cli.k8s.client.CoreV1Api") as api:
           api.return_value.list_namespaced_pod.return_value = fake
           pods = list_pods("platform")
       assert [p.name for p in pods] == ["api-1", "api-2"]
       assert pods[1].restarts == 7
       api.return_value.list_namespaced_pod.assert_called_once_with("platform", label_selector="", timeout_seconds=10)
   ```

5. Run it against kind.

   ```bash
   kubectl create namespace platform
   kubectl --namespace platform create deployment echo --image=registry.k8s.io/e2e-test-images/agnhost:2.47 -- /agnhost netexec --http-port=8080
   kubectl --namespace platform rollout status deployment/echo
   PLATFORM_LOG_JSON=true uv run platform-cli pods list --namespace platform
   ```

   Expected output:

   ```text
   {"source": "kubeconfig", "event": "k8s_config", "level": "debug", "timestamp": "2026-10-27T10:21:44.120Z"}
   {"namespace": "platform", "count": 1, "event": "pods_listed", "level": "info", "timestamp": "2026-10-27T10:21:44.231Z"}
   echo-6d4f8b7c9d-x2k4p                              Running    restarts=0   node=lab-control-plane
   ```

   The `debug` line only appears if `PLATFORM_LOG_LEVEL=DEBUG`.

### Verify

```bash
uv run platform-cli pods list -n platform -o json | python -c "import json,sys; d=json.load(sys.stdin); print(len(d), d[0]['phase'])"
```

```text
1 Running
```

### Cleanup

```bash
kubectl delete namespace platform
```

(Keep the kind cluster for the FastAPI labs.)

### What you learned

The official client's config loading order, `SimpleNamespace` as a lightweight fake for SDK objects, `patch` at the point of use (`platform_cli.k8s.client.CoreV1Api`, not `kubernetes.client.CoreV1Api`), and JSON logs that a collector can parse.

---

## Lab 6: Profile, Scan, and Ship a Distroless Image

**Objective.** Profile the CLI's startup, run security scans, and build a multi-stage non-root distroless image under 80 MB that runs `platform-cli`.

**Prerequisites.** Lab 5; Docker; `trivy` (optional).

**Time.** 60 minutes.

### Steps

1. Measure import time.

   ```bash
   uv run python -X importtime -c "import platform_cli.cli" 2>&1 | sort -t'|' -k2 -n | tail -5
   ```

   Expected output (your numbers will vary):

   ```text
   import time:      9812 |      41233 | kubernetes.client
   import time:      1320 |      62011 | platform_cli.k8s
   import time:      2104 |      88420 | boto3
   import time:      1100 |     101230 | platform_cli.cli
   ```

   `kubernetes.client` and `boto3` dominate. Fix: import them lazily inside the functions that need them so `--help` and `http check` do not pay for them. Move `import boto3` into `s3_client()` and `from kubernetes import client, config` into `load_config()`/`list_pods()`. Re-measure; `platform_cli.cli` should drop below 30 ms.

2. CPU-profile a command.

   ```bash
   uv run python -m cProfile -o cli.prof -m platform_cli http check http://127.0.0.1:9/ || true
   uv run python -c "import pstats; pstats.Stats('cli.prof').sort_stats('cumtime').print_stats(8)"
   ```

3. Security scans.

   ```bash
   uv run ruff check --select S .
   uvx pip-audit
   uv export --no-dev --format requirements-txt -o requirements.txt && uvx pip-audit -r requirements.txt
   ```

   Expected: `No known vulnerabilities found`. If a finding appears, `uv lock --upgrade-package <name>` and re-run.

4. Write the Dockerfile.

   ```dockerfile
   # syntax=docker/dockerfile:1.7
   ############################ build stage ############################
   FROM ghcr.io/astral-sh/uv:0.5.11-python3.12-bookworm-slim AS build
   ENV UV_COMPILE_BYTECODE=1 UV_LINK_MODE=copy UV_PYTHON_DOWNLOADS=never \
       UV_PROJECT_ENVIRONMENT=/opt/venv
   WORKDIR /src
   COPY pyproject.toml uv.lock ./
   RUN --mount=type=cache,target=/root/.cache/uv \
       uv sync --frozen --no-dev --no-install-project --no-editable
   COPY src ./src
   COPY README.md ./
   RUN --mount=type=cache,target=/root/.cache/uv \
       uv sync --frozen --no-dev --no-editable

   ############################ runtime stage ##########################
   FROM gcr.io/distroless/python3-debian12:nonroot
   # distroless python3-debian12 ships Python 3.11; we bring our own 3.12 interpreter below.
   COPY --from=build /usr/local/bin/python3.12 /usr/local/bin/python3.12
   COPY --from=build /usr/local/lib/python3.12 /usr/local/lib/python3.12
   COPY --from=build /usr/local/lib/libpython3.12.so.1.0 /usr/local/lib/libpython3.12.so.1.0
   COPY --from=build --chown=nonroot:nonroot /opt/venv /opt/venv
   ENV PATH="/opt/venv/bin:$PATH" PYTHONUNBUFFERED=1 PYTHONDONTWRITEBYTECODE=1 \
       LD_LIBRARY_PATH=/usr/local/lib
   USER nonroot:nonroot
   WORKDIR /home/nonroot
   ENTRYPOINT ["/usr/local/bin/python3.12", "-m", "platform_cli"]
   CMD ["--help"]
   ```

   Simpler alternative when distroless interpreter juggling is not worth it (slightly larger, still non-root, no package manager removed): use `python:3.12-slim-bookworm` as the runtime and add `RUN useradd --uid 65532 --no-create-home app` + `USER 65532`. The FastAPI folder uses that variant and explains the trade-off.

5. Build, size, run.

   ```bash
   docker build -t platform-cli:lab6 .
   docker image inspect platform-cli:lab6 --format '{{.Size}}' | awk '{printf "%.1f MB\n", $1/1024/1024}'
   docker run --rm platform-cli:lab6 --version
   docker run --rm --read-only --tmpfs /tmp platform-cli:lab6 http check http://host.docker.internal:9/ ; echo "exit=$?"
   ```

   Expected output:

   ```text
   74.3 MB
   platform-cli 0.1.0
   ERR     -      1.4ms http://host.docker.internal:9/ ConnectError
   exit=1
   ```

6. Scan the image.

   ```bash
   trivy image --severity HIGH,CRITICAL --ignore-unfixed platform-cli:lab6
   docker run --rm --entrypoint sh platform-cli:lab6 -c "id" ; echo "no shell is the point: exit=$?"
   ```

   The second command fails because distroless has no `sh`: an attacker who gains code execution has no shell either.

### Verify

```bash
docker run --rm platform-cli:lab6 --help | head -3
docker inspect platform-cli:lab6 --format '{{.Config.User}}'
```

```text
Usage: python -m platform_cli [OPTIONS] COMMAND [ARGS]...
...
nonroot:nonroot
```

### Cleanup

```bash
docker rmi platform-cli:lab6
rm -f cli.prof requirements.txt urls.txt
```

### What you learned

Lazy imports for CLI startup, cProfile/`-X importtime`, `pip-audit` and `ruff -S` as pre-merge gates, and a BuildKit multi-stage Dockerfile that produces a non-root, shell-less, cache-friendly image. This is the pattern the FastAPI service's Dockerfile builds on.
