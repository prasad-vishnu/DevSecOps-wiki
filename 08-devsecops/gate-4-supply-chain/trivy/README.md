# Trivy

| | |
|---|---|
| **Cluster** | 08 DevSecOps |
| **Role** | Anchor (Gate 4: Container, Kubernetes & IaC scanning) |
| **Category** | Unified vulnerability, misconfiguration, secret, license and SBOM scanner |
| **Prerequisites** | Docker and OCI image layout, package managers (apk/dpkg/rpm, pip/npm/maven), YAML, basic Kubernetes |
| **Target version** | Trivy 0.56+ (CLI), Trivy Operator 0.24+ |
| **Capstone phase** | Phase 5 (Months 17-20) |
| **Official docs** | https://trivy.dev/latest/docs/ |

> Trivy is a single Go binary that answers five questions about any artefact: which known vulnerabilities does it contain, is it misconfigured, does it leak secrets, which licenses does it carry, and what exactly is inside it (SBOM). It scans container images, filesystems, Git repositories, Kubernetes clusters, VM images, and SBOM files with the same command structure. After working through this folder you will be able to wire Trivy as a hard CI gate that blocks `docker push` on fixable critical CVEs, run it continuously inside Kubernetes with the Trivy Operator, scan Terraform and Helm for misconfigurations with built-in and custom Rego checks, operate it in an air-gapped network, and feed its findings into DefectDojo.

## Why This Tool Exists

Before 2019 a team wanting to scan a container needed a running Clair server with a Postgres database, a registry integration, and a separate tool for Dockerfiles, another for Terraform, and another for secrets. Each had its own database update story and output format. Trivy's insight was that **all of these are the same problem: walk a filesystem, identify artefacts (packages, config files, strings), and match them against a database of rules**. By shipping the vulnerability database as a downloadable OCI artefact and embedding the misconfiguration checks as Rego compiled into the binary, Trivy removed the server entirely. One static binary, one command shape, under a minute per image.

The trade-off: Trivy does breadth. It is excellent at container and lockfile vulnerabilities, very good at IaC misconfigurations, adequate at secrets (Gitleaks and TruffleHog in Gate 1 are deeper), and its license detection is heuristic. Knowing where to pair it with a specialist is part of mastering it.

## Architecture

```mermaid
flowchart TB
    subgraph Input["Targets"]
        I1[image]
        I2[fs / rootfs]
        I3[repo]
        I4[config]
        I5[k8s]
        I6[sbom]
        I7[vm]
    end
    subgraph Core["Trivy binary"]
        A[Artifact resolver<br/>pulls image, walks FS] --> C[Cache<br/>layer-keyed blobs]
        C --> AN[Analyzers<br/>OS release, apk/dpkg/rpm DBs,<br/>lockfiles, jars, binaries,<br/>IaC files, secrets regex]
        AN --> SC[Scanners<br/>vuln | misconfig | secret | license]
        SC --> R[Report<br/>table/json/sarif/cyclonedx/spdx/template]
    end
    subgraph Data["Databases (OCI artefacts)"]
        DB[(trivy-db<br/>vulnerabilities, 6h refresh)]
        JDB[(trivy-java-db<br/>Maven GAV index)]
        CHK[(trivy-checks<br/>Rego misconfig bundle)]
        VEX[(VEX Hub)]
    end
    DB --> SC
    JDB --> AN
    CHK --> SC
    VEX --> SC
    Input --> A
    R --> O1[CI exit code]
    R --> O2[GitHub code scanning SARIF]
    R --> O3[DefectDojo]
    R --> O4[Cosign attestation]
```

- **Artifact resolver.** Turns a target into a filesystem view. For images it pulls layers (from the daemon, containerd, podman, a remote registry, or a tar archive) and squashes them into a virtual filesystem, remembering which layer introduced each file.
- **Cache.** Analysed layer results are cached by layer digest under `~/.cache/trivy`. Re-scanning an image whose base layers are unchanged only analyses the new layers.
- **Analyzers.** One analyzer per artefact type. OS analyzers read `/etc/os-release` and the package database (`/lib/apk/db/installed`, `/var/lib/dpkg/status`, `/var/lib/rpm/Packages`). Language analyzers parse lockfiles (`requirements.txt`, `poetry.lock`, `package-lock.json`, `pom.xml`, `go.sum`, `Gemfile.lock`, `Cargo.lock`, `*.jar` metadata, etc.).
- **Scanners.** Each scanner consumes the analyzer output: `vuln` matches packages against trivy-db, `misconfig` evaluates Rego against parsed IaC, `secret` runs regex rules over file contents, `license` classifies license strings.
- **Databases.** `trivy-db` is a bbolt file packaged as an OCI artefact on `ghcr.io/aquasecurity/trivy-db` (mirrored to ECR and GHCR mirrors). `trivy-java-db` maps jar SHA-1 hashes to Maven coordinates. `trivy-checks` bundles the misconfiguration policies.

## The Core Mental Model

1. **Trivy scans a filesystem, not a build.** It sees what is *installed* in the final image, which is exactly what will run. It does not know about dev dependencies that were removed in a multi-stage build, and it cannot see which code paths are executed. That is why it finds real OS-level exposure that source-only SCA (Snyk, Gate 3) misses, and why VEX exists to express "present but not reachable".
2. **A finding is a (package, version, advisory) triple matched by ecosystem.** For OS packages the matcher uses the *distribution's* security feed (Debian security tracker, Alpine SecDB, Red Hat OVAL), because the distro backports fixes without changing upstream version numbers. For language packages it uses GHSA/OSV/NVD by package URL. Understanding this explains most "false positive" debates: the question is always *which feed decided this*.
3. **Severity gating plus fixability equals an actionable gate.** `--severity HIGH,CRITICAL --ignore-unfixed --exit-code 1` says: block only on things a developer can fix today by bumping a version or rebuilding on a newer base. Everything else goes to the tracking system, not the pipeline.

## Anchor-and-Bridge Mapping

| Concept in Trivy | Grype + Syft | Snyk Container (Gate 3) | Checkov (Gate 5) |
|---|---|---|---|
| `trivy image ref` | `grype ref` | `snyk container test ref` | n/a |
| `trivy image --format cyclonedx` | `syft ref -o cyclonedx-json` | `snyk container sbom` | n/a |
| `trivy sbom file` | `grype sbom:file` | n/a | n/a |
| `trivy fs .` (lockfiles) | `grype dir:.` | `snyk test` | n/a |
| `trivy config .` | n/a | `snyk iac test` | `checkov -d .` |
| `trivy image --scanners secret` | n/a | n/a | `checkov --framework secrets` |
| `--severity HIGH,CRITICAL` | `--fail-on high` | `--severity-threshold=high` | `--check` / `--skip-check` |
| `--ignore-unfixed` | `--only-fixed` | `--fail-on=upgradable` | n/a |
| `.trivyignore.yaml` | `.grype.yaml ignore:` | `.snyk` policy | inline `checkov:skip` |
| `--vex` | `--vex` | n/a | n/a |
| Trivy Operator | n/a | Snyk K8s monitor | n/a |

## Learning Path

| Step | File | Time |
|---|---|---|
| 1 | [01-installation-and-setup.md](01-installation-and-setup.md) | 1 h |
| 2 | [02-core-concepts.md](02-core-concepts.md) | 3 h |
| 3 | [03-commands-and-cheatsheet.md](03-commands-and-cheatsheet.md) | 2 h |
| 4 | [04-hands-on-labs.md](04-hands-on-labs.md) | 8 h |
| 5 | [05-production-patterns.md](05-production-patterns.md) | 3 h |
| 6 | [06-troubleshooting.md](06-troubleshooting.md) | 1 h |
| 7 | [07-interview-questions.md](07-interview-questions.md) | 2 h |

## What Trivy Scans: Target Matrix

| Target | Command | Scanners available | Typical use |
|---|---|---|---|
| Container image | `trivy image` | vuln, misconfig (image config), secret, license | CI gate before push |
| Filesystem | `trivy fs` | vuln (lockfiles), misconfig, secret, license | Repo checkout, build context |
| Root filesystem | `trivy rootfs` | vuln (OS + lang), secret, license | Unpacked container, host, chroot |
| Git repository | `trivy repo` | vuln, misconfig, secret, license | Remote repo scan without clone |
| IaC config | `trivy config` | misconfig only | Terraform, CloudFormation, Kubernetes, Helm, Dockerfile, Azure ARM |
| Kubernetes cluster | `trivy k8s` | vuln, misconfig, secret, RBAC, compliance | Ad-hoc cluster audit |
| SBOM | `trivy sbom` | vuln, license | Scan CycloneDX/SPDX produced earlier |
| VM image | `trivy vm` | vuln, secret, license | AMI, VMDK, local disk image |

## Role in the Capstone

Trivy is the first tool that runs on the artefact rather than the code, so it is the hinge between the build and release stages of `.github/workflows/release.yml`:

- **Gate step.** `aquasecurity/trivy-action` scans `ghcr.io/example-org/fastapi-api@sha256:<DIGEST>` and `ghcr.io/example-org/spring-api@sha256:<DIGEST>` with `severity: HIGH,CRITICAL`, `ignore-unfixed: true`, `exit-code: 1`. SARIF is uploaded to GitHub code scanning so findings appear on the PR.
- **IaC step.** `trivy config infra/terraform platform/helm` runs in the `plan` job alongside Checkov; the two overlap deliberately so that an ADR can compare them on real findings.
- **Attestation.** `trivy image --format cosign-vuln` produces the predicate for `cosign attest --type vuln`, which Kyverno checks at admission.
- **Runtime.** Trivy Operator runs in the kind and EKS clusters; its `VulnerabilityReport` metrics are scraped by Prometheus and a Grafana panel shows critical CVE count per namespace next to the SLO dashboard.
- **ASPM.** JSON reports are posted to DefectDojo (`Trivy Scan` parser) with `engagement` per service; DefectDojo owns deduplication and SLA tracking.

## Further Reading

- Trivy documentation: https://trivy.dev/latest/docs/
- Trivy Operator: https://aquasecurity.github.io/trivy-operator/latest/
- trivy-checks (misconfiguration policy source): https://github.com/aquasecurity/trivy-checks
- OpenVEX specification and Trivy VEX Hub: https://github.com/aquasecurity/vexhub
- CycloneDX and SPDX specifications
- "Container Security" by Liz Rice (O'Reilly), chapters on image scanning
