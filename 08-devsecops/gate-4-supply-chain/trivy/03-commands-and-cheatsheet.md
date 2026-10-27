# Trivy: Commands and Cheatsheet

| | |
|---|---|
| **Cluster** | 08 DevSecOps |
| **Role** | Anchor |
| **Prerequisites** | [02-core-concepts.md](02-core-concepts.md) |
| **Target version** | Trivy 0.56+ |
| **Capstone phase** | Phase 5 (Months 17-20) |

> Task-organised reference. Every command is runnable as written against public images or the capstone repo. Long and short flags are both shown where they exist. Flags are stable across subcommands: anything that works on `trivy image` almost always works on `trivy fs`, `rootfs`, `repo`, and `sbom`.

---

## 1. Global Flags (apply to every subcommand)

| Flag | Short | Purpose |
|---|---|---|
| `--cache-dir <dir>` | | Cache location (default `~/.cache/trivy`) |
| `--config <file>` | `-c` | Config file (default `trivy.yaml`) |
| `--debug` | `-d` | Debug logging |
| `--quiet` | `-q` | Suppress progress bar and logs |
| `--timeout <dur>` | | Scan timeout (default `5m0s`) |
| `--insecure` | | Skip registry TLS verification (lab only) |
| `--format <fmt>` | `-f` | `table` (default), `json`, `sarif`, `cyclonedx`, `spdx`, `spdx-json`, `github`, `template`, `cosign-vuln` |
| `--output <file>` | `-o` | Write report to file instead of stdout |
| `--severity <list>` | `-s` | `UNKNOWN,LOW,MEDIUM,HIGH,CRITICAL` subset |
| `--exit-code <n>` | | Exit code when findings exist (default 0) |
| `--exit-on-eol <n>` | | Exit code when OS is end-of-life |
| `--scanners <list>` | | `vuln,misconfig,secret,license` |
| `--ignorefile <file>` | | Default `.trivyignore` |
| `--ignore-policy <rego>` | | Rego ignore policy |
| `--list-all-pkgs` | | Include non-vulnerable packages in output |
| `--dependency-tree` | | Show why a transitive package is present (table only) |
| `--skip-dirs <glob>` / `--skip-files <glob>` | | Exclude paths |
| `--file-patterns <type:regex>` | | Map unusual filenames to analyzers, e.g. `pip:requirements-.*\.txt` |
| `--offline-scan` | | Do not reach out to external services (Maven Central) |

---

## 2. Image Scanning

```bash
# Basic scan from the local daemon or registry
trivy image python:3.12-slim

# Gate: only HIGH/CRITICAL, only fixable, fail the shell
trivy image --severity HIGH,CRITICAL --ignore-unfixed --exit-code 1 ghcr.io/example-org/api:1.4.2

# Scan by digest (what CI should do)
trivy image ghcr.io/example-org/api@sha256:<DIGEST>

# Vulnerabilities only (skip secret scanning for speed)
trivy image --scanners vuln nginx:1.27

# All four scanners plus image-config checks (USER, ENV secrets)
trivy image --scanners vuln,misconfig,secret,license --image-config-scanners misconfig,secret nginx:1.27

# Scan an exported tarball (no daemon, no registry)
docker save ghcr.io/example-org/api:1.4.2 -o api.tar
trivy image --input api.tar

# Scan an OCI layout directory (output of buildx --output type=oci)
trivy image --input ./oci-layout

# Choose image source explicitly
trivy image --image-src remote ghcr.io/example-org/api:1.4.2
trivy image --image-src containerd --platform linux/arm64 ghcr.io/example-org/api:1.4.2

# Include non-vulnerable packages and the layer that introduced each
trivy image --list-all-pkgs --format json --output full.json alpine:3.20

# Show dependency tree for language packages (why is this transitive here?)
trivy image --dependency-tree --severity HIGH,CRITICAL node:22-alpine

# Ignore specific statuses rather than all unfixed
trivy image --ignore-status will_not_fix,fix_deferred,end_of_life registry.access.redhat.com/ubi9/ubi:latest

# Fail if the base OS is end-of-life even with no CVEs
trivy image --exit-on-eol 2 debian:10

# Skip Java DB download (speed) at the cost of unmatched shaded jars
trivy image --skip-java-db-update eclipse-temurin:21-jre

# Registry credentials inline (prefer docker login / OIDC)
TRIVY_USERNAME=<USER> TRIVY_PASSWORD=<TOKEN_FROM_SECRET_STORE> trivy image registry.example.com/api:1.4.2
```

Exit code semantics: `0` = scan ran, no findings at or above `--severity` (or `--exit-code` unset). `--exit-code 1` = that code when findings exist. Scan errors (pull failure, timeout) exit `1` regardless. Combine with `--exit-on-eol` for a distinct code on EOL.

---

## 3. Filesystem, Rootfs and Repository Scanning

```bash
# Scan a source checkout: lockfiles + IaC + secrets
trivy fs --scanners vuln,misconfig,secret .

# Only lockfiles, severity gate
trivy fs --scanners vuln --severity HIGH,CRITICAL --exit-code 1 ./services/fastapi-api

# Map a non-standard lockfile name to the pip analyzer
trivy fs --file-patterns "pip:requirements-prod\.txt" ./services/fastapi-api

# Skip vendored and virtualenv dirs
trivy fs --skip-dirs node_modules,.venv,vendor .

# Scan an unpacked container root or a host
sudo trivy rootfs --scanners vuln /
trivy rootfs ./extracted-image-root

# Remote repository, specific branch, no manual clone
trivy repo --branch release/1.4 https://github.com/example-org/api
trivy repo --commit <SHA> https://github.com/example-org/api
trivy repo --tag v1.4.2 https://github.com/example-org/api
```

---

## 4. Misconfiguration (IaC) Scanning

```bash
# Scan a directory: auto-detects Terraform, CloudFormation, K8s, Helm, Dockerfile
trivy config ./infra

# Fail on HIGH/CRITICAL misconfigs
trivy config --severity HIGH,CRITICAL --exit-code 1 ./infra/terraform

# Terraform with variable files
trivy config --tf-vars infra/terraform/envs/prod.tfvars ./infra/terraform

# Scan a Terraform plan (resolved values, including data sources)
terraform plan -out plan.out && terraform show -json plan.out > plan.json
trivy config plan.json

# Exclude downloaded modules (scan only your code)
trivy config --tf-exclude-downloaded-modules ./infra/terraform

# Helm chart with production values
trivy config --helm-values charts/api/values-prod.yaml --helm-set image.tag=1.4.2 charts/api
trivy config --helm-api-versions "networking.k8s.io/v1" --helm-kube-version 1.31.0 charts/api

# Kubernetes manifests rendered by kustomize
kubectl kustomize k8s/overlays/prod | trivy config --file-patterns "kubernetes:.*" -

# Dockerfile only
trivy config --file-patterns "dockerfile:Containerfile" ./services/spring-api

# Show passed checks too (useful for compliance evidence)
trivy config --include-non-failures ./infra

# Custom Rego checks (directory of .rego) and the namespaces they live in
trivy config --config-check ./security/trivy-checks --check-namespaces user ./k8s

# Skip a check globally
trivy config --skip-check-update --ignorefile .trivyignore.yaml ./infra

# Disable the embedded checks and use only custom ones
trivy config --config-check ./security/trivy-checks --check-namespaces user --skip-check-update --disable-embedded-policies ./k8s

# Misconfig scan as part of an image scan (Dockerfile in image history)
trivy image --scanners misconfig --image-config-scanners misconfig ghcr.io/example-org/api:1.4.2

# Pass custom data to Rego checks (data.* documents)
trivy config --config-data ./security/trivy-data ./k8s
```

### Common check IDs

| ID | Framework | Finding |
|---|---|---|
| `AVD-AWS-0086` / `0087` / `0088` / `0089` / `0090` / `0091` / `0092` / `0093` / `0094` | Terraform/CFN | S3 public ACL, public policy, no encryption, no logging, no versioning, block-public-acls, ignore-public-acls, restrict-public-buckets, block-public-policy |
| `AVD-AWS-0176` / `0177` | Terraform/CFN | RDS IAM auth disabled, deletion protection off |
| `AVD-AWS-0107` / `0104` | Terraform/CFN | Security group ingress from 0.0.0.0/0, egress to 0.0.0.0/0 |
| `AVD-AWS-0057` | Terraform/CFN | IAM policy with wildcard actions |
| `AVD-AWS-0038` / `0040` / `0041` | Terraform/CFN | EKS control-plane logging, public cluster endpoint, public access CIDRs |
| `KSV001` / `KSV003` / `KSV011` / `KSV012` / `KSV014` / `KSV017` / `KSV020` / `KSV021` / `KSV030` | Kubernetes | allowPrivilegeEscalation, drop capabilities, CPU limits, runAsNonRoot, read-only rootfs, privileged, runAsUser > 10000, runAsGroup > 10000, seccomp |
| `DS001` / `DS002` / `DS005` / `DS026` / `DS031` | Dockerfile | latest tag, no USER, ADD vs COPY, no HEALTHCHECK, secret in ENV |

---

## 5. Secret Scanning

```bash
# Secrets only, in a repo
trivy fs --scanners secret .

# Secrets inside an image filesystem and its config ENV/history
trivy image --scanners secret --image-config-scanners secret ghcr.io/example-org/api:1.4.2

# Custom rules file
trivy fs --scanners secret --secret-config trivy-secret.yaml .
```

`trivy-secret.yaml`:

```yaml
rules:
  - id: example-org-internal-token
    category: ExampleOrg
    title: Example Org internal service token
    severity: HIGH
    regex: 'xorg_[A-Za-z0-9]{32}'
    keywords: ["xorg_"]
    secret-group-name: ""
allow-rules:
  - id: test-fixtures
    description: Synthetic secrets used by the test suite
    path: 'tests/fixtures/.*'
  - id: example-aws-key
    description: AWS documentation example key
    regex: 'AKIAIOSFODNN7EXAMPLE'
disable-rules:
  - generic-unwanted-rule
disable-allow-rules:
  - markdown   # re-enable scanning of .md files
enable-builtin-rules:
  - aws-access-key-id
  - github-pat
```

Disabled by default for performance: scanning of very large files (`--secret-config` `max-file-size`). Secret findings show `Category`, `RuleID`, `Severity`, line numbers, and a redacted `Match`.

---

## 6. License Scanning

```bash
# Package metadata licenses in an image
trivy image --scanners license ghcr.io/example-org/api:1.4.2

# Full text classification of LICENSE files and headers (slower)
trivy fs --scanners license --license-full .

# Override the buckets
trivy image --scanners license --severity HIGH,CRITICAL ghcr.io/example-org/api:1.4.2
```

Bucket configuration in `trivy.yaml`:

```yaml
license:
  forbidden: ["AGPL-1.0", "AGPL-3.0", "SSPL-1.0"]
  restricted: ["GPL-2.0", "GPL-3.0", "LGPL-2.1", "LGPL-3.0", "CC-BY-SA-4.0"]
  reciprocal: ["MPL-2.0", "EPL-2.0", "CDDL-1.0"]
  notice: ["Apache-2.0", "MIT", "BSD-2-Clause", "BSD-3-Clause", "ISC"]
  permissive: []
  unencumbered: ["CC0-1.0", "Unlicense", "0BSD"]
  ignored: ["public-domain"]
  confidenceLevel: 0.9
```

---

## 7. SBOM: Generate and Scan

```bash
# Generate CycloneDX JSON from an image (includes OS + language packages)
trivy image --format cyclonedx --output sbom.cdx.json ghcr.io/example-org/api@sha256:<DIGEST>

# SPDX JSON and tag-value
trivy image --format spdx-json --output sbom.spdx.json ghcr.io/example-org/api@sha256:<DIGEST>
trivy image --format spdx --output sbom.spdx ghcr.io/example-org/api@sha256:<DIGEST>

# SBOM from a source tree (lockfiles)
trivy fs --format cyclonedx --output sbom.cdx.json .

# SBOM with vulnerabilities embedded (CycloneDX VEX-style vulnerabilities[])
trivy image --format cyclonedx --scanners vuln --output sbom-with-vulns.cdx.json ghcr.io/example-org/api:1.4.2

# Scan an existing SBOM (Syft, Trivy, or build-tool generated)
trivy sbom sbom.cdx.json
trivy sbom --severity HIGH,CRITICAL --exit-code 1 sbom.spdx.json

# Scan an SBOM attached to an image as a Cosign attestation
trivy image --sbom-sources oci ghcr.io/example-org/api@sha256:<DIGEST>

# Scan SBOM referenced via Rekor
trivy image --sbom-sources rekor ghcr.io/example-org/api@sha256:<DIGEST>

# Cosign vulnerability attestation predicate
trivy image --format cosign-vuln --output vuln.json ghcr.io/example-org/api@sha256:<DIGEST>
cosign attest --yes --predicate vuln.json --type vuln ghcr.io/example-org/api@sha256:<DIGEST>
```

---

## 8. VEX

```bash
# Local OpenVEX / CycloneDX VEX / CSAF
trivy image --vex security/vex/api.openvex.json ghcr.io/example-org/api@sha256:<DIGEST>

# VEX attached as attestation on the image
trivy image --vex oci ghcr.io/example-org/api@sha256:<DIGEST>

# VEX Hub (community VEX for upstream projects)
trivy vex repo init
trivy vex repo download
trivy vex repo list
trivy image --vex repo ghcr.io/example-org/api@sha256:<DIGEST>

# Show which findings VEX suppressed
trivy image --vex api.openvex.json --show-suppressed ghcr.io/example-org/api@sha256:<DIGEST>
```

---

## 9. Output Formats

```bash
# JSON (canonical)
trivy image --format json --output report.json alpine:3.20

# SARIF for GitHub code scanning / Azure DevOps / DefectDojo
trivy image --format sarif --output trivy.sarif alpine:3.20

# GitHub Dependency Snapshot (submit to dependency graph)
trivy image --format github --output deps.json alpine:3.20

# Built-in templates (shipped in /contrib or the container at /contrib)
trivy image --format template --template "@contrib/html.tpl" --output report.html alpine:3.20
trivy image --format template --template "@contrib/junit.tpl" --output junit.xml alpine:3.20
trivy image --format template --template "@contrib/gitlab.tpl" --output gl-container-scanning-report.json alpine:3.20
trivy image --format template --template "@contrib/asff.tpl" --output asff.json alpine:3.20   # AWS Security Hub

# Inline Go template: one CSV line per vuln
trivy image --format template \
  --template '{{ range .Results }}{{ $t := .Target }}{{ range .Vulnerabilities }}{{ $t }},{{ .PkgName }},{{ .InstalledVersion }},{{ .FixedVersion }},{{ .VulnerabilityID }},{{ .Severity }}{{ "\n" }}{{ end }}{{ end }}' \
  alpine:3.20

# Table with only the columns you want (use jq on JSON instead)
trivy image --format json alpine:3.20 | jq -r '.Results[].Vulnerabilities[]? | [.VulnerabilityID, .Severity, .PkgName, .FixedVersion] | @tsv'

# Table with non-failing checks and compact view
trivy config --format table --include-non-failures ./infra
```

Convert an existing JSON report to another format without re-scanning:

```bash
trivy convert --format sarif --output trivy.sarif report.json
trivy convert --format table --severity CRITICAL report.json
trivy convert --format template --template "@contrib/html.tpl" --output report.html report.json
```

---

## 10. Database Management

```bash
# Download only (no scan), e.g. in a Dockerfile layer for CI images
trivy image --download-db-only
trivy image --download-java-db-only
trivy image --download-db-only --db-repository ghcr.io/aquasecurity/trivy-db:2

# Skip updates (air-gapped or pinned)
trivy image --skip-db-update --skip-java-db-update --skip-check-update alpine:3.20
export TRIVY_SKIP_DB_UPDATE=true

# Use an internal mirror (ECR/ACR/Harbor)
trivy image --db-repository 123456789012.dkr.ecr.<REGION>.amazonaws.com/trivy-db:2 \
            --java-db-repository 123456789012.dkr.ecr.<REGION>.amazonaws.com/trivy-java-db:1 \
            --checks-bundle-repository 123456789012.dkr.ecr.<REGION>.amazonaws.com/trivy-checks:1 alpine:3.20

# Multiple fallbacks
trivy image --db-repository registry.example.com/trivy-db:2,ghcr.io/aquasecurity/trivy-db:2 alpine:3.20

# Mirror the DB yourself with oras
oras pull ghcr.io/aquasecurity/trivy-db:2
oras push 123456789012.dkr.ecr.<REGION>.amazonaws.com/trivy-db:2 \
  --artifact-type application/vnd.aquasec.trivy.config.v1+json \
  db.tar.gz:application/vnd.aquasec.trivy.db.layer.v1.tar+gzip

# Inspect the current DB
cat ~/.cache/trivy/db/metadata.json

# Reset
trivy clean --all
trivy clean --vuln-db
trivy clean --java-db
trivy clean --checks-bundle
trivy clean --scan-cache
```

Expected `metadata.json`:

```json
{"Version":2,"NextUpdate":"2026-10-27T12:12:41Z","UpdatedAt":"2026-10-27T06:12:41Z","DownloadedAt":"2026-10-27T08:02:28Z"}
```

---

## 11. Cache

```bash
# Filesystem cache (default)
trivy image --cache-dir /var/cache/trivy alpine:3.20

# Shared Redis cache for a runner fleet
trivy image --cache-backend redis://trivy-cache.internal:6379 --redis-tls alpine:3.20
trivy image --cache-backend redis://trivy-cache.internal:6379 \
  --redis-ca /etc/trivy/ca.crt --redis-cert /etc/trivy/client.crt --redis-key /etc/trivy/client.key alpine:3.20

# No cache (single-shot)
trivy image --cache-backend memory alpine:3.20

# Clear only analysed layers
trivy clean --scan-cache
```

---

## 12. Client/Server Mode

The server holds the DB and cache; thin clients send layers for analysis results. Useful when runners cannot reach the internet.

```bash
# Server (listens on 0.0.0.0:4954), auto-updates DB
trivy server --listen 0.0.0.0:4954 --cache-dir /var/cache/trivy --token <SERVER_TOKEN_FROM_SECRET_STORE>

# Server in Kubernetes via Helm
helm repo add aquasecurity https://aquasecurity.github.io/helm-charts/
helm install trivy aquasecurity/trivy --namespace trivy-system --create-namespace \
  --set trivy.debugMode=false --set persistence.enabled=true --set persistence.size=10Gi

# Client
trivy image --server http://trivy-server.internal:4954 --token <SERVER_TOKEN_FROM_SECRET_STORE> alpine:3.20
trivy fs --server http://trivy-server.internal:4954 .

# Client with custom header (behind an auth proxy)
trivy image --server https://trivy.example.com --custom-headers "Authorization: Bearer <OIDC_TOKEN>" alpine:3.20
```

In client/server mode the client still pulls the image locally and sends analysed blobs; only the vulnerability lookup is remote. Misconfiguration and secret scanning run client-side.

---

## 13. Kubernetes CLI

```bash
# Summary of the whole cluster (workloads, infra, RBAC)
trivy k8s --report summary

# Full report for one namespace
trivy k8s --namespace prod --report all

# Specific workload
trivy k8s --namespace prod deployment/api

# Only vulnerabilities, HIGH+
trivy k8s --scanners vuln --severity HIGH,CRITICAL --report summary

# Include nodes (uses node-collector DaemonSet, needs privileges)
trivy k8s --include-kinds node --report summary

# Compliance reports
trivy k8s --compliance k8s-cis-1.23 --report summary
trivy k8s --compliance k8s-nsa-1.0 --report all --format json --output nsa.json
trivy k8s --compliance k8s-pss-restricted-0.1 --report summary
trivy k8s --compliance eks-cis-1.4 --report summary

# Different context / kubeconfig
trivy k8s --kubeconfig ~/.kube/prod --context prod-eks --report summary

# Exclude kinds / namespaces
trivy k8s --exclude-kinds configmap,secret --exclude-namespaces kube-system,kube-public --report summary

# Tolerate taints / scan nodes with tolerations
trivy k8s --tolerations "node-role.kubernetes.io/control-plane=:NoSchedule" --report summary
```

Expected summary output excerpt:

```text
Summary Report for prod-eks
┌───────────┬───────────────────┬────────────────────────┬────────────────────┬───────────────────┐
│ Namespace │     Resource      │    Vulnerabilities     │ Misconfigurations  │      Secrets      │
│           │                   ├────┬────┬────┬────┬────┼───┬───┬───┬───┬────┼───┬───┬───┬───┬───┤
│           │                   │ C  │ H  │ M  │ L  │ U  │ C │ H │ M │ L │ U  │ C │ H │ M │ L │ U │
├───────────┼───────────────────┼────┼────┼────┼────┼────┼───┼───┼───┼───┼────┼───┼───┼───┼───┼───┤
│ prod      │ Deployment/api    │    │ 2  │ 14 │ 31 │    │   │ 1 │ 3 │   │    │   │   │   │   │   │
│ prod      │ Deployment/worker │ 1  │ 5  │ 20 │ 40 │    │   │ 2 │ 3 │   │    │   │   │   │   │   │
└───────────┴───────────────────┴────┴────┴────┴────┴────┴───┴───┴───┴───┴────┴───┴───┴───┴───┴───┘
```

---

## 14. Trivy Operator kubectl Reference

```bash
# Install (Helm)
helm install trivy-operator aquasecurity/trivy-operator \
  --namespace trivy-system --create-namespace \
  --set="trivy.ignoreUnfixed=true" \
  --set="trivy.severity=HIGH\,CRITICAL" \
  --set="operator.scanJobsConcurrentLimit=5" \
  --set="compliance.cron=0 */6 * * *" \
  --set="operator.metricsVulnIdEnabled=true"

# Reports
kubectl get vulnerabilityreports --all-namespaces -o wide
kubectl get vulnerabilityreports -n prod -o custom-columns=NAME:.metadata.name,CRIT:.report.summary.criticalCount,HIGH:.report.summary.highCount
kubectl describe vulnerabilityreport -n prod replicaset-api-7d9f8b6c5-api
kubectl get configauditreports -n prod
kubectl get exposedsecretreports -A
kubectl get sbomreports -n prod -o jsonpath='{.items[0].report.components}' | jq .
kubectl get rbacassessmentreports -A
kubectl get clusterrbacassessmentreports
kubectl get infraassessmentreports -n kube-system
kubectl get clustercompliancereports
kubectl get clustercompliancereport cis -o jsonpath='{.status.summary}'
kubectl get clustercompliancereport nsa -o yaml

# Force rescan by deleting the report (operator recreates)
kubectl delete vulnerabilityreport -n prod replicaset-api-7d9f8b6c5-api

# Watch scan jobs
kubectl get jobs -n trivy-system -w
kubectl logs -n trivy-system deploy/trivy-operator -f

# Metrics endpoint
kubectl port-forward -n trivy-system svc/trivy-operator 8080:80
curl -s localhost:8080/metrics | grep trivy_image_vulnerabilities | head
```

---

## 15. Plugins and Utilities

```bash
trivy plugin list
trivy plugin install github.com/aquasecurity/trivy-plugin-referrer     # OCI referrers for SBOM/attestations
trivy plugin install github.com/aquasecurity/trivy-plugin-aqua
trivy referrer put --type cyclonedx ghcr.io/example-org/api@sha256:<DIGEST> --file sbom.cdx.json

trivy version --format json
trivy filesystem --help          # alias of fs
trivy rootfs --help
trivy module install github.com/aquasecurity/trivy-module-spring4shell   # WASM modules for post-scan analysis
```

---

## 16. Minimal Reference `trivy.yaml` for the Capstone

```yaml
severity: [HIGH, CRITICAL]
exit-code: 1
timeout: 15m
scan:
  scanners: [vuln, secret, misconfig]
  skip-dirs: [node_modules, .venv, .git, target]
vulnerability:
  ignore-unfixed: true
  ignore-status: [will_not_fix, end_of_life]
ignorefile: security/trivy/.trivyignore.yaml
misconfiguration:
  config-check: [security/trivy/checks]
  check-namespaces: [user]
  helm:
    values: [charts/api/values-prod.yaml]
  terraform:
    exclude-downloaded-modules: true
secret:
  config: security/trivy/trivy-secret.yaml
db:
  repository: ghcr.io/aquasecurity/trivy-db:2
  java-repository: ghcr.io/aquasecurity/trivy-java-db:1
cache:
  backend: fs
```

Run `trivy image --config security/trivy/trivy.yaml ghcr.io/example-org/api@sha256:<DIGEST>`.

---

## Next

[04-hands-on-labs.md](04-hands-on-labs.md) applies these commands in nine progressive labs.
