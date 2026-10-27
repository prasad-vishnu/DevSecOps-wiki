# Trivy: Installation and Setup

| | |
|---|---|
| **Cluster** | 08 DevSecOps |
| **Role** | Anchor |
| **Prerequisites** | A shell, Docker or Podman for image scanning, outbound HTTPS to `ghcr.io` (or an internal mirror) |
| **Target version** | Trivy 0.56+ |
| **Capstone phase** | Phase 5 (Months 17-20) |

> After this file you will have Trivy installed on your workstation and in CI, the vulnerability and Java databases downloaded and cached, registry authentication working, and a first scan of a real image verified.

---

## 1. Installation

Trivy is a single static binary. Pick one method per environment and pin the version in anything reproducible.

### 1.1 Linux

**Debian / Ubuntu (apt repository):**

```bash
sudo apt-get install -y wget apt-transport-https gnupg lsb-release
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | gpg --dearmor | sudo tee /usr/share/keyrings/trivy.gpg > /dev/null
echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main" | sudo tee /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install -y trivy
```

**RHEL / Fedora / Amazon Linux (yum/dnf):**

```bash
cat <<'EOF' | sudo tee /etc/yum.repos.d/trivy.repo
[trivy]
name=Trivy repository
baseurl=https://aquasecurity.github.io/trivy-repo/rpm/releases/$basearch/
gpgcheck=1
enabled=1
gpgkey=https://aquasecurity.github.io/trivy-repo/rpm/public.key
EOF
sudo dnf install -y trivy
```

**Install script (any Linux/macOS, pinned version, verifies checksum):**

```bash
curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh \
  | sh -s -- -b /usr/local/bin v0.56.2
```

**Binary from GitHub releases with manual checksum verification (preferred for hardened CI images):**

```bash
TRIVY_VERSION=0.56.2
curl -sSLO "https://github.com/aquasecurity/trivy/releases/download/v${TRIVY_VERSION}/trivy_${TRIVY_VERSION}_Linux-64bit.tar.gz"
curl -sSLO "https://github.com/aquasecurity/trivy/releases/download/v${TRIVY_VERSION}/trivy_${TRIVY_VERSION}_checksums.txt"
grep "trivy_${TRIVY_VERSION}_Linux-64bit.tar.gz" "trivy_${TRIVY_VERSION}_checksums.txt" | sha256sum --check
tar -xzf "trivy_${TRIVY_VERSION}_Linux-64bit.tar.gz" trivy
sudo install -m 0755 trivy /usr/local/bin/trivy
```

Expected output of the checksum line:

```text
trivy_0.56.2_Linux-64bit.tar.gz: OK
```

Trivy releases are themselves signed with Cosign. Verifying the release signature is covered in [../cosign/04-hands-on-labs.md](../cosign/04-hands-on-labs.md).

### 1.2 macOS

```bash
brew install trivy
```

### 1.3 Windows

```powershell
# Scoop
scoop bucket add extras
scoop install trivy

# or Chocolatey
choco install trivy

# or winget
winget install AquaSecurity.Trivy
```

Manual: download `trivy_<VERSION>_windows-64bit.zip` from the releases page, extract `trivy.exe`, and place it on `PATH`. Trivy on Windows scans images through Docker Desktop and filesystems natively; `trivy k8s` works with any kubeconfig.

### 1.4 Container image

```bash
docker run --rm \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v "$HOME/.cache/trivy:/root/.cache/trivy" \
  aquasec/trivy:0.56.2 image python:3.12-slim
```

Mount the cache directory so the database is not re-downloaded every run. Mounting the Docker socket lets Trivy read local images; omit it and Trivy pulls from the registry instead. Note the socket mount grants root-equivalent access to the host; in CI prefer registry pulls or `--input image.tar`.

### 1.5 GitHub Actions

```yaml
- name: Install Trivy
  uses: aquasecurity/setup-trivy@v0.2.2
  with:
    version: v0.56.2
    cache: true
```

Or use the all-in-one action `aquasecurity/trivy-action` shown in [05-production-patterns.md](05-production-patterns.md).

### 1.6 Verify

```bash
trivy --version
```

Expected output (DB lines appear only after the first download):

```text
Version: 0.56.2
Vulnerability DB:
  Version: 2
  UpdatedAt: 2026-10-27 06:12:41.123456789 +0000 UTC
  NextUpdate: 2026-10-27 12:12:41.123456789 +0000 UTC
  DownloadedAt: 2026-10-27 08:02:03.456789 +0000 UTC
Java DB:
  Version: 1
  UpdatedAt: 2026-10-26 01:00:12.000000 +0000 UTC
  NextUpdate: 2026-10-29 01:00:12.000000 +0000 UTC
  DownloadedAt: 2026-10-27 08:02:30.123456 +0000 UTC
Check Bundle:
  Digest: sha256:1a2b3c...
  DownloadedAt: 2026-10-27 08:02:35.000000 +0000 UTC
```

---

## 2. First-Run Configuration

### 2.1 Databases and cache location

| Item | Default location | Override |
|---|---|---|
| Cache root | `$HOME/.cache/trivy` (Linux/macOS), `%LOCALAPPDATA%\trivy` (Windows) | `--cache-dir`, `TRIVY_CACHE_DIR` |
| Vulnerability DB | `<cache>/db/trivy.db` + `metadata.json` | `--db-repository ghcr.io/aquasecurity/trivy-db:2` |
| Java DB | `<cache>/java-db/` | `--java-db-repository ghcr.io/aquasecurity/trivy-java-db:1` |
| Checks bundle | `<cache>/policy/` | `--checks-bundle-repository ghcr.io/aquasecurity/trivy-checks:1` |
| Layer analysis cache | `<cache>/fanal/` | `--cache-backend fs|redis://host:6379|memory` |

Pre-download everything once (useful in CI images and air-gapped prep):

```bash
trivy image --download-db-only
trivy image --download-java-db-only
```

Expected output:

```text
2026-10-27T08:02:03Z  INFO  [vulndb] Need to update DB
2026-10-27T08:02:03Z  INFO  [vulndb] Downloading vulnerability DB...
2026-10-27T08:02:03Z  INFO  [vulndb] Downloading artifact...  repo="mirror.gcr.io/aquasec/trivy-db:2"
2026-10-27T08:02:28Z  INFO  [vulndb] Artifact successfully downloaded  repo="mirror.gcr.io/aquasec/trivy-db:2"
```

Trivy 0.56+ tries `mirror.gcr.io/aquasec/trivy-db`, then `ghcr.io/aquasecurity/trivy-db`, so GHCR rate limits (`TOOMANYREQUESTS`) are rarer. If you see them, set `--db-repository` to an internal mirror (see [05-production-patterns.md](05-production-patterns.md#air-gapped)).

### 2.2 Configuration file

Every flag can live in `trivy.yaml` in the working directory (or `--config path`). Flag name maps to a nested key: `--severity` becomes `severity`, `--ignore-unfixed` becomes `vulnerability.ignore-unfixed`. Flags beat the file; environment variables `TRIVY_<FLAG>` beat both.

```yaml
# trivy.yaml - repo-level defaults
severity:
  - HIGH
  - CRITICAL
exit-code: 1
timeout: 10m
cache:
  dir: /tmp/trivy-cache
scan:
  scanners:
    - vuln
    - secret
    - misconfig
  skip-dirs:
    - node_modules
    - .venv
vulnerability:
  ignore-unfixed: true
ignorefile: .trivyignore.yaml
db:
  repository: ghcr.io/aquasecurity/trivy-db:2
  java-repository: ghcr.io/aquasecurity/trivy-java-db:1
misconfiguration:
  helm:
    values:
      - charts/api/values-prod.yaml
```

### 2.3 Environment variables

| Variable | Purpose |
|---|---|
| `TRIVY_SEVERITY=HIGH,CRITICAL` | Same as `--severity` |
| `TRIVY_EXIT_CODE=1` | Same as `--exit-code` |
| `TRIVY_IGNORE_UNFIXED=true` | Same as `--ignore-unfixed` |
| `TRIVY_CACHE_DIR` | Cache location |
| `TRIVY_DB_REPOSITORY` | DB mirror |
| `TRIVY_USERNAME` / `TRIVY_PASSWORD` | Registry basic auth (prefer `docker login` or OIDC-based registry tokens) |
| `TRIVY_INSECURE=true` | Skip registry TLS verification (lab only) |
| `TRIVY_TIMEOUT=15m` | Overall scan timeout (default 5 m) |
| `TRIVY_DEBUG=true` | Debug logging |
| `TRIVY_NON_SSL=true` | Plain HTTP registries (lab only) |
| `TRIVY_OFFLINE_SCAN=true` | Do not call external services (Maven Central lookups) |

---

## 3. Authentication

### 3.1 Container registries

Trivy reads `~/.docker/config.json`, so `docker login` is sufficient. Cloud registries work with their native credential helpers:

```bash
# Amazon ECR (uses the AWS credential chain, OIDC-assumed role in CI)
aws ecr get-login-password --region <REGION> \
  | docker login --username AWS --password-stdin 123456789012.dkr.ecr.<REGION>.amazonaws.com
trivy image 123456789012.dkr.ecr.<REGION>.amazonaws.com/api:1.4.2

# GHCR in GitHub Actions
echo "${{ secrets.GITHUB_TOKEN }}" | docker login ghcr.io -u "${{ github.actor }}" --password-stdin

# Google Artifact Registry
gcloud auth configure-docker <REGION>-docker.pkg.dev
```

Trivy also supports the registry credential chain directly without a daemon: for ECR it uses the AWS SDK, for GCR/GAR the Google credentials, for ACR `az acr login`. No Docker daemon is needed to scan remote images; Trivy pulls layers with `go-containerregistry`.

### 3.2 Kubernetes

`trivy k8s` uses the current kubeconfig context and needs read access to Pods, Deployments, and other workload kinds plus the ability to pull images (it pulls images itself from outside the cluster, so your workstation needs registry credentials too).

```bash
kubectl config current-context
trivy k8s --report summary cluster
```

### 3.3 Private Git repositories

```bash
GITHUB_TOKEN=<FINE_GRAINED_TOKEN_FROM_SECRET_STORE> trivy repo https://github.com/example-org/private-api
# or local clone
git clone git@github.com:example-org/private-api.git && trivy fs ./private-api
```

---

## 4. Shell Completion and Editor Support

```bash
trivy completion bash | sudo tee /etc/bash_completion.d/trivy > /dev/null
trivy completion zsh > "${fpath[1]}/_trivy"
trivy completion fish > ~/.config/fish/completions/trivy.fish
```

VS Code: the "Aqua Trivy" extension runs `trivy fs` on the workspace and shows findings in the Problems pane.

---

## 5. Smoke Test

```bash
trivy image --severity HIGH,CRITICAL python:3.12-slim
```

Expected output (counts vary by date):

```text
2026-10-27T08:05:11Z  INFO  [vuln] Vulnerability scanning is enabled
2026-10-27T08:05:11Z  INFO  [secret] Secret scanning is enabled
2026-10-27T08:05:11Z  INFO  [secret] If your scanning is slow, please try '--scanners vuln' to disable secret scanning
2026-10-27T08:05:19Z  INFO  Detected OS  family="debian" version="12.7"
2026-10-27T08:05:19Z  INFO  [debian] Detecting vulnerabilities...  os_version="12" pkg_num=97
2026-10-27T08:05:19Z  INFO  Number of language-specific files  num=1
2026-10-27T08:05:19Z  INFO  [python-pkg] Detecting vulnerabilities...

python:3.12-slim (debian 12.7)
==============================
Total: 2 (HIGH: 2, CRITICAL: 0)

┌──────────┬────────────────┬──────────┬────────┬───────────────────┬───────────────┬─────────────────────────────────────────────┐
│ Library  │ Vulnerability  │ Severity │ Status │ Installed Version │ Fixed Version │                    Title                    │
├──────────┼────────────────┼──────────┼────────┼───────────────────┼───────────────┼─────────────────────────────────────────────┤
│ libexpat1│ CVE-2024-45490 │ HIGH     │ fixed  │ 2.5.0-1           │ 2.5.0-1+deb12u1│ libexpat: Negative Length Parsing Vulnerab..│
│ zlib1g   │ CVE-2023-45853 │ CRITICAL │ affected│ 1:1.2.13.dfsg-1  │               │ zlib: integer overflow and resultant heap...│
└──────────┴────────────────┴──────────┴────────┴───────────────────┴───────────────┴─────────────────────────────────────────────┘
```

The `Status` column is the key to triage: `fixed` means a newer package exists (rebuild on a fresher base), `affected` means no fix yet (`--ignore-unfixed` suppresses it), `will_not_fix` means the distro has declined to patch.

Exit code is `0` here because `--exit-code` was not set. Add `--exit-code 1` to make the same scan fail the shell.

---

## 6. Uninstall

```bash
sudo apt-get remove -y trivy          # or dnf remove / brew uninstall / scoop uninstall
rm -rf "$HOME/.cache/trivy"
```

---

## Next

Continue with [02-core-concepts.md](02-core-concepts.md) to understand what the analyzers found and why the status column says what it says.
