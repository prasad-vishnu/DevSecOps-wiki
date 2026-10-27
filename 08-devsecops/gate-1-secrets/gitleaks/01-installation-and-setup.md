# Gitleaks: Installation and Setup

| | |
|---|---|
| **Cluster** | 08 DevSecOps |
| **Role** | Gate 1 anchor |
| **Prerequisites** | Git 2.30+, a shell, optionally Docker and Go 1.22+ |
| **Target version** | Gitleaks 8.18+ |
| **Capstone phase** | Phase 5 (Months 17-20) |

> After this file Gitleaks is installed on your workstation and in a container, runs as a pre-commit hook in the capstone repository, and you have verified it by catching a clearly fake key.

---

## 1. Install the Binary

### macOS (Homebrew)

```bash
brew install gitleaks
gitleaks version
```

Expected output:

```text
8.21.2
```

### Linux (release tarball)

```bash
GITLEAKS_VERSION=8.21.2
ARCH=$(uname -m | sed 's/x86_64/x64/;s/aarch64/arm64/')
curl -sSL -o /tmp/gitleaks.tar.gz \
  "https://github.com/gitleaks/gitleaks/releases/download/v${GITLEAKS_VERSION}/gitleaks_${GITLEAKS_VERSION}_linux_${ARCH}.tar.gz"
# Verify the checksum published with the release before trusting the binary
curl -sSL -o /tmp/checksums.txt \
  "https://github.com/gitleaks/gitleaks/releases/download/v${GITLEAKS_VERSION}/gitleaks_${GITLEAKS_VERSION}_checksums.txt"
(cd /tmp && grep "linux_${ARCH}.tar.gz" checksums.txt | sha256sum --check)
sudo tar -xzf /tmp/gitleaks.tar.gz -C /usr/local/bin gitleaks
gitleaks version
```

### Windows

```powershell
# winget
winget install --id Gitleaks.Gitleaks
# or Chocolatey
choco install gitleaks
# or Scoop
scoop install gitleaks
gitleaks version
```

### Go toolchain

```bash
go install github.com/gitleaks/gitleaks/v8@latest
# $GOPATH/bin must be on PATH
gitleaks version
```

### Docker

```bash
docker pull ghcr.io/gitleaks/gitleaks:latest
# Scan the current directory's git history. Mount read-only; run as your UID so reports are owned by you.
docker run --rm --user "$(id -u):$(id -g)" \
  --volume "$PWD:/repo:ro" \
  ghcr.io/gitleaks/gitleaks:latest git /repo --verbose
```

Pin a tag (`ghcr.io/gitleaks/gitleaks:v8.21.2`) in CI and lab manifests; `latest` is fine for interactive use.

---

## 2. First Run

Inside any Git repository:

```bash
cd ~/src/capstone
gitleaks git . --verbose
```

Expected output on a clean repository:

```text
    ○
    │╲
    │ ○
    ○ ░
    ░    gitleaks

12:01:14 INF 1 commits scanned.
12:01:14 INF scan completed in 62.3ms
12:01:14 INF no leaks found
```

Exit code `0`. If leaks are found you get a block per finding and exit code `1`.

### Legacy verbs still work

| Legacy (v8.0-8.18) | Modern (v8.19+) |
|---|---|
| `gitleaks detect` | `gitleaks git .` |
| `gitleaks detect --no-git` | `gitleaks dir .` |
| `gitleaks protect` | `gitleaks git --pre-commit` |
| `gitleaks protect --staged` | `gitleaks git --pre-commit --staged` |

Both forms are documented in this folder because many CI templates still use `detect`/`protect`.

---

## 3. Seed a Fake Secret and Catch It

Create a throwaway repository with a clearly fake AWS key. (This key is invalid by construction and is used throughout the wiki as the standard fake.)

```bash
mkdir -p /tmp/gl-demo && cd /tmp/gl-demo && git init -q
cat > config.py <<'EOF'
# FAKE credential for scanner testing only
AWS_ACCESS_KEY_ID = "AKIAEXAMPLEEXAMPLE00"
AWS_SECRET_ACCESS_KEY = "EXAMPLEexampleEXAMPLEexampleEXAMPLE00000"
EOF
git add config.py && git -c user.name=demo -c user.email=demo@example.com commit -qm "add config"
gitleaks git . --verbose
```

Expected output:

```text
Finding:     AWS_ACCESS_KEY_ID = "AKIAEXAMPLEEXAMPLE00"
Secret:      AKIAEXAMPLEEXAMPLE00
RuleID:      aws-access-token
Entropy:     3.121928
File:        config.py
Line:        2
Commit:      3f1c2b9e...
Author:      demo
Email:       demo@example.com
Date:        2026-10-27T06:31:14Z
Fingerprint: 3f1c2b9e...:config.py:aws-access-token:2

12:03:02 INF 1 commits scanned.
12:03:02 INF scan completed in 48.1ms
12:03:02 WRN leaks found: 1
```

Exit code `1`. Note the `Fingerprint` line; it is the identity used by `.gitleaksignore`.

The secret-key line may or may not fire depending on version because the generic rule requires surrounding keywords and entropy; the access-key line always fires because `AKIA` + 16 uppercase alphanumerics is a precise rule.

---

## 4. Repository Configuration File

Create `.gitleaks.toml` at the repository root. Start by extending the defaults rather than replacing them:

```toml
# .gitleaks.toml
title = "capstone gitleaks config"

[extend]
# Pull in all ~170 built-in rules; we only add and allowlist on top.
useDefault = true

[allowlist]
description = "Global allowlist"
paths = [
  '''(^|/)tests/fixtures/''',
  '''\.lock$''',
  '''(^|/)node_modules/''',
  '''(^|/)\.git/''',
]
regexes = [
  '''AKIAEXAMPLEEXAMPLE00''',        # documented fake used in labs
  '''ghp_EXAMPLE0{32}''',            # documented fake GitHub PAT
]
stopwords = [
  "example",
  "placeholder",
  "changeme",
]
```

Gitleaks looks for config in this order: `--config <path>`, `GITLEAKS_CONFIG` env var, `.gitleaks.toml` in the target repo root, then the embedded default.

Validate it loads:

```bash
gitleaks git . --config .gitleaks.toml --verbose 2>&1 | head -3
```

Expected: no `ERR` lines about TOML parsing, and the demo key from section 3 is now allowlisted (exit code `0` in `/tmp/gl-demo` if you copy the file there).

---

## 5. Pre-commit Hook

### Option A: the pre-commit framework (recommended)

```bash
pipx install pre-commit        # or: pip install --user pre-commit / brew install pre-commit
```

`.pre-commit-config.yaml`:

```yaml
repos:
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.21.2
    hooks:
      - id: gitleaks
        # The hook runs: gitleaks git --pre-commit --redact --staged --verbose
        # Override args only if you must, e.g. to point at a non-default config:
        # args: ["--config", ".gitleaks.toml"]
```

```bash
pre-commit install
pre-commit run gitleaks --all-files   # one-off full check
```

Expected output:

```text
Detect hardcoded secrets.................................................Passed
```

Now a commit containing a secret fails before the commit object is created:

```bash
echo 'TOKEN="ghp_EXAMPLE000000000000000000000000000000"' > leak.sh  # fake
git add leak.sh && git commit -m "oops"
```

```text
Detect hardcoded secrets.................................................Failed
- hook id: gitleaks
- exit code: 1

Finding:     TOKEN="REDACTED"
Secret:      REDACTED
RuleID:      github-pat
...
```

### Option B: raw Git hook (no Python)

```bash
cat > .git/hooks/pre-commit <<'EOF'
#!/usr/bin/env bash
set -euo pipefail
if ! command -v gitleaks >/dev/null; then
  echo "gitleaks not installed; see 08-devsecops/gate-1-secrets/gitleaks/01-installation-and-setup.md" >&2
  exit 1
fi
exec gitleaks git --pre-commit --staged --redact --verbose
EOF
chmod +x .git/hooks/pre-commit
```

Raw hooks are per-clone and not versioned; use `git config core.hooksPath .githooks` and commit the `.githooks/` directory if you want them shared without the pre-commit framework.

### Bypass policy

Developers *can* run `git commit --no-verify`. That is why the PR check exists. Do not try to make the hook un-bypassable; make the PR check authoritative and the hook convenient.

---

## 6. CI Setup (GitHub Actions)

Minimal PR check:

```yaml
# .github/workflows/gitleaks.yml
name: gitleaks
on: [pull_request, push]
permissions:
  contents: read
  security-events: write
jobs:
  scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0          # full history so commit-range scanning works
      - uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          # Required for organisation-owned repositories (free for individuals/OSS):
          GITLEAKS_LICENSE: ${{ secrets.GITLEAKS_LICENSE }}
          GITLEAKS_ENABLE_UPLOAD_ARTIFACT: "true"
          GITLEAKS_ENABLE_SUMMARY: "true"
```

The Action automatically scans the commit range for PRs and pushes, writes a job summary, and uploads `results.sarif` as an artefact. The full SARIF-to-code-scanning wiring, Docker-based alternative without the licence requirement, and scheduling are in [05-production-patterns.md](05-production-patterns.md).

---

## 7. Verification Checklist

| Check | Command | Expected |
|---|---|---|
| Binary on PATH | `gitleaks version` | `8.2x.y` |
| Config parses | `gitleaks git . --config .gitleaks.toml` | No `ERR` lines |
| Fake key detected without allowlist | section 3 demo | exit `1`, `RuleID: aws-access-token` |
| Fake key allowlisted with config | copy `.gitleaks.toml` to demo repo, re-run | exit `0` |
| Hook installed | `cat .git/hooks/pre-commit \| head -3` | pre-commit shim present |
| Hook blocks a commit | section 5 demo | `Failed` with `github-pat` |
| CI job runs | open a PR | green check, SARIF artefact |

---

## 8. Environment Variables

| Variable | Effect |
|---|---|
| `GITLEAKS_CONFIG` | Path to config file (overrides repo `.gitleaks.toml`) |
| `GITLEAKS_LICENSE` | Licence key for the GitHub Action on organisation repos |
| `GITLEAKS_ENABLE_UPLOAD_ARTIFACT` | Action: upload SARIF artefact (`true`/`false`) |
| `GITLEAKS_ENABLE_SUMMARY` | Action: write job summary |
| `GITLEAKS_ENABLE_COMMENTS` | Action: comment on PR with findings |
| `GITLEAKS_NOTIFY_USER_LIST` | Action: comma-separated `@users` to mention in comments |

Next: [02-core-concepts.md](02-core-concepts.md).
