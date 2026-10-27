# Python: Installation and Setup

| | |
|---|---|
| **Cluster** | 07 Backend & APIs |
| **Role** | Language foundation |
| **Prerequisites** | A terminal, Git, Docker (for the container section) |
| **Target version** | Python 3.12+, uv 0.5+ |
| **Capstone phase** | Phase 1 (Months 1-4) |

> After this file you will have a reproducible Python toolchain on Linux, macOS, Windows and in containers: `uv` managing interpreters and virtualenvs, a `pyproject.toml` that configures `ruff`, `mypy` and `pytest`, and `pre-commit` running them before every commit. You will also know the legacy `pip`/`venv` equivalents for machines where you cannot install `uv`.

---

## 1. Decide How You Will Install Python

| Method | When to use | Verdict |
|---|---|---|
| **`uv python install`** | Everywhere you control | **Primary.** Downloads python-build-standalone builds, no compilation, per-project pinning via `.python-version` |
| OS package (`apt install python3.12`) | Servers where only system Python is permitted | Fine for running, poor for development. Never `sudo pip install` into it (PEP 668 will block you anyway) |
| `pyenv` | Legacy workstations | Works, but compiles from source; `uv` supersedes it |
| Official installer / Microsoft Store | Windows without `uv` | Acceptable; make sure "Add to PATH" is ticked |
| Docker `python:3.12-slim` | Containers | Build stage; see section 7 |
| Conda / Miniconda | Data science with compiled native deps | Avoid for platform work; mixes badly with `pip`/`uv` |

---

## 2. Install `uv`

### Linux and macOS

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
# installs to ~/.local/bin; restart the shell or:
export PATH="$HOME/.local/bin:$PATH"
uv --version
```

Expected output:

```text
uv 0.5.x (...)
```

On macOS you can also `brew install uv`. On Linux distros with pipx: `pipx install uv`.

### Windows (PowerShell)

```powershell
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"
uv --version
```

Or `winget install --id=astral-sh.uv -e`.

### Verify the installer script before piping to a shell (recommended on servers)

```bash
curl -LsSf https://astral.sh/uv/install.sh -o /tmp/uv-install.sh
less /tmp/uv-install.sh        # read it
sh /tmp/uv-install.sh
```

---

## 3. Install an Interpreter

```bash
uv python install 3.12
uv python list
```

Expected output (abridged):

```text
cpython-3.13.1-linux-x86_64-gnu     <download available>
cpython-3.12.8-linux-x86_64-gnu     /home/<USER>/.local/share/uv/python/cpython-3.12.8-linux-x86_64-gnu/bin/python3.12
cpython-3.11.11-linux-x86_64-gnu    <download available>
```

Pin it for a project:

```bash
mkdir platform-cli && cd platform-cli
uv python pin 3.12
cat .python-version
```

```text
3.12
```

`uv` reads `.python-version` and will use (or download) that interpreter for every `uv run` in this directory. Commit the file.

---

## 4. Create a Project

```bash
uv init --package platform-cli        # --package gives you the src/ layout and a console script
cd platform-cli
tree -a -I .git
```

```text
.
├── .python-version
├── README.md
├── pyproject.toml
└── src
    └── platform_cli
        └── __init__.py
```

Add runtime and development dependencies:

```bash
uv add typer httpx "pydantic>=2.7" structlog
uv add --dev pytest pytest-asyncio pytest-cov ruff mypy pre-commit respx
```

`uv add` edits `pyproject.toml`, resolves, writes `uv.lock`, and syncs `.venv/`. You never activate the venv manually; you run through `uv run`.

```bash
uv run python -c "import httpx, pydantic; print(httpx.__version__, pydantic.VERSION)"
```

```text
0.28.1 2.10.4
```

---

## 5. The `pyproject.toml` You Should Start From

Replace the generated file with this. Every tool is configured in one place.

```toml
[project]
name = "platform-cli"
version = "0.1.0"
description = "Platform automation CLI for the DevSecOps capstone"
readme = "README.md"
requires-python = ">=3.12"
license = { text = "Apache-2.0" }
authors = [{ name = "Platform Team" }]
dependencies = [
  "typer>=0.12",
  "httpx>=0.27",
  "pydantic>=2.7",
  "pydantic-settings>=2.3",
  "structlog>=24.1",
]

[project.optional-dependencies]
aws = ["boto3>=1.34"]
k8s = ["kubernetes>=30.1"]

[project.scripts]
platform-cli = "platform_cli.cli:app"

[dependency-groups]
dev = [
  "pytest>=8.2",
  "pytest-asyncio>=0.23",
  "pytest-cov>=5.0",
  "ruff>=0.6",
  "mypy>=1.11",
  "pre-commit>=3.7",
  "respx>=0.21",
  "moto[s3]>=5.0",
  "boto3-stubs[s3,sts]>=1.34",
  "types-requests",
]

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

# ---------------------------------------------------------------- ruff
[tool.ruff]
target-version = "py312"
line-length = 100
src = ["src", "tests"]

[tool.ruff.lint]
select = [
  "E", "W",    # pycodestyle
  "F",         # pyflakes
  "I",         # isort
  "B",         # bugbear (common bugs)
  "UP",        # pyupgrade (modern syntax)
  "S",         # bandit security rules
  "ASYNC",     # async pitfalls (blocking calls in async code)
  "PTH",       # prefer pathlib
  "RUF",       # ruff-specific
  "T20",       # no print()
]
ignore = ["S101"]   # allow assert in tests (see per-file-ignores) and code

[tool.ruff.lint.per-file-ignores]
"tests/**" = ["S101", "S105", "S106"]

[tool.ruff.format]
quote-style = "double"

# ---------------------------------------------------------------- mypy
[tool.mypy]
python_version = "3.12"
strict = true
warn_unreachable = true
plugins = ["pydantic.mypy"]
mypy_path = "src"
packages = ["platform_cli"]

[[tool.mypy.overrides]]
module = ["kubernetes.*", "moto.*"]
ignore_missing_imports = true

# ---------------------------------------------------------------- pytest
[tool.pytest.ini_options]
testpaths = ["tests"]
asyncio_mode = "auto"
addopts = "-ra --strict-markers --cov=platform_cli --cov-report=term-missing"
filterwarnings = ["error", "ignore::DeprecationWarning:botocore.*"]

[tool.coverage.run]
branch = true
source = ["src"]
```

Sync after editing:

```bash
uv sync --all-extras
uv lock --check          # fails if uv.lock is out of date (use this in CI)
```

---

## 6. Wire Up `pre-commit`

`.pre-commit-config.yaml` at the repo root:

```yaml
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v5.0.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-added-large-files
      - id: detect-private-key
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.6.9
    hooks:
      - id: ruff
        args: [--fix]
      - id: ruff-format
  - repo: https://github.com/pre-commit/mirrors-mypy
    rev: v1.11.2
    hooks:
      - id: mypy
        additional_dependencies: ["pydantic>=2.7", "types-requests"]
        args: [--strict]
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.21.2
    hooks:
      - id: gitleaks
```

```bash
uv run pre-commit install
uv run pre-commit run --all-files
```

Expected output:

```text
trim trailing whitespace.................................................Passed
fix end of files.........................................................Passed
check yaml...............................................................Passed
check for added large files..............................................Passed
detect private key.......................................................Passed
ruff.....................................................................Passed
ruff-format..............................................................Passed
mypy.....................................................................Passed
Detect hardcoded secrets.................................................Passed
```

Gitleaks here is the first of the DevSecOps gates from cluster 08; wiring it into Python projects from day one is deliberate.

---

## 7. Legacy Equivalents: `pip` and `venv`

You will meet servers and CI images without `uv`. Know the equivalents.

| Task | `uv` | `pip` / `venv` |
|---|---|---|
| Create venv | implicit (`uv sync`) | `python3 -m venv .venv && source .venv/bin/activate` |
| Install deps | `uv sync` | `pip install -r requirements.txt` |
| Add dep | `uv add httpx` | `pip install httpx && pip freeze > requirements.txt` (loses intent) |
| Lock | `uv lock` | `pip-compile pyproject.toml -o requirements.txt` (pip-tools) |
| Export for pip | `uv export --format requirements-txt --no-dev > requirements.txt` | n/a |
| Run a tool once | `uvx ruff check .` | `pipx run ruff check .` |
| Install a CLI globally | `uv tool install ruff` | `pipx install ruff` |
| Install Python | `uv python install 3.12` | `pyenv install 3.12` or apt |

The `uv export` command is what you use when a downstream system (Snyk, Dependabot, an old Dockerfile) insists on `requirements.txt`. Keep `uv.lock` as the source of truth and generate the text file in CI.

PEP 668 note: on Debian 12+, Ubuntu 23.04+, Fedora 38+, `pip install` into system Python fails with `externally-managed-environment`. This is correct behaviour. Use a venv or `uv`.

---

## 8. Editor Setup

VS Code `.vscode/settings.json`:

```json
{
  "python.defaultInterpreterPath": "${workspaceFolder}/.venv/bin/python",
  "python.testing.pytestEnabled": true,
  "[python]": {
    "editor.defaultFormatter": "charliermarsh.ruff",
    "editor.formatOnSave": true,
    "editor.codeActionsOnSave": { "source.organizeImports": "explicit", "source.fixAll": "explicit" }
  },
  "mypy-type-checker.args": ["--strict"]
}
```

Install the "Ruff", "Mypy Type Checker" and "Python" extensions. PyCharm picks up `.venv` automatically.

---

## 9. Containers: The Build-Stage Baseline

The full production Dockerfile is in [05-production-patterns.md](05-production-patterns.md). The minimal reproducible build stage looks like this:

```dockerfile
# syntax=docker/dockerfile:1.7
FROM ghcr.io/astral-sh/uv:0.5.11-python3.12-bookworm-slim AS build
ENV UV_COMPILE_BYTECODE=1 UV_LINK_MODE=copy UV_PYTHON_DOWNLOADS=never
WORKDIR /app
COPY pyproject.toml uv.lock ./
RUN --mount=type=cache,target=/root/.cache/uv \
    uv sync --frozen --no-dev --no-install-project
COPY src ./src
RUN --mount=type=cache,target=/root/.cache/uv \
    uv sync --frozen --no-dev
```

`--frozen` refuses to run if `uv.lock` does not match `pyproject.toml`, which is the container equivalent of `uv lock --check`.

---

## 10. Verification Checklist

```bash
uv --version                       # uv 0.5+
uv run python --version            # Python 3.12.x
uv run ruff check . && uv run ruff format --check .
uv run mypy
uv run pytest -q
uv lock --check
uv run pre-commit run --all-files
```

All green means your toolchain is reproducible. Commit `pyproject.toml`, `uv.lock`, `.python-version`, `.pre-commit-config.yaml`. Do **not** commit `.venv/` (add it to `.gitignore`).

---

## 11. Common Setup Failures

| Symptom | Cause | Fix |
|---|---|---|
| `uv: command not found` after install | `~/.local/bin` not on PATH | `export PATH="$HOME/.local/bin:$PATH"` in your shell rc |
| `error: externally-managed-environment` | Using system pip on PEP 668 distro | Use `uv` or a venv |
| `uv sync` downloads Python even though one is installed | System Python version differs from `.python-version` | `uv python pin <installed version>` or let it download (fine) |
| `ModuleNotFoundError: platform_cli` in tests | Package not installed in editable mode | `uv sync` installs the project itself; check `[build-system]` exists |
| mypy: `Cannot find implementation or library stub for module named "boto3"` | Missing stubs | `uv add --dev "boto3-stubs[s3]"` |
| Windows: `ruff` works but `pre-commit` fails on line endings | CRLF | `git config core.autocrlf input`; add `end-of-file-fixer` |
