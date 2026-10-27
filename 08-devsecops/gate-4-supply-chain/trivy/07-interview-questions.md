# Trivy: Interview Questions

| | |
|---|---|
| **Cluster** | 08 DevSecOps |
| **Role** | Anchor |
| **Prerequisites** | All previous Trivy files |
| **Target version** | Trivy 0.56+ |
| **Capstone phase** | Phase 5 (Months 17-20) |

> Thirty questions from junior to principal level. Model answers are deliberately dense; in an interview, lead with the first sentence and expand on request.

---

## Junior

### Q1. What does Trivy scan, in one sentence each? (Junior)

**Answer.** Container images (OS and language packages, image config), filesystems and Git repositories (lockfiles, IaC, secrets), IaC configuration (Terraform, CloudFormation, Kubernetes, Helm, Dockerfile) for misconfigurations, Kubernetes clusters (workload images plus manifests and RBAC), SBOM files (CycloneDX/SPDX) for vulnerabilities, and VM disk images. The scanners are `vuln`, `misconfig`, `secret`, and `license`.

### Q2. What is the difference between `trivy image`, `trivy fs`, and `trivy rootfs`? (Junior)

**Answer.** `image` pulls and squashes container layers and scans what the container would see, including OS packages. `fs` scans a source tree for lockfiles, IaC, and secrets but does not look for OS package databases. `rootfs` scans an unpacked operating system (a chroot, host `/`, or extracted image) and enables the OS package analyzers.

### Q3. Explain `--severity HIGH,CRITICAL --ignore-unfixed --exit-code 1`. (Junior)

**Answer.** Report only HIGH and CRITICAL findings, drop any that have no fix available yet, and exit with code 1 if any remain. Together they make a CI gate that fails only on vulnerabilities the team can act on today by upgrading a package or rebuilding on a newer base image.

### Q4. Where does Trivy get its vulnerability data? (Junior)

**Answer.** From `trivy-db`, an aggregated database built from NVD, GitHub Security Advisories, OSV, and distribution feeds (Debian, Ubuntu, Alpine, Red Hat, Amazon, SUSE, Oracle, Photon, Wolfi, and others). It is published every six hours as an OCI artefact on `ghcr.io/aquasecurity/trivy-db` and cached locally under `~/.cache/trivy`.

### Q5. What is a `.trivyignore` file? (Junior)

**Answer.** A text file with one finding ID per line (CVE, AVD check ID, secret rule ID) that Trivy suppresses. The YAML variant `.trivyignore.yaml` adds `paths`, `purls`, `statement`, and `expired_at` so suppressions are scoped, justified, and self-expiring.

### Q6. How do you scan an image without Docker installed? (Junior)

**Answer.** Trivy pulls directly from the registry using `go-containerregistry` (`--image-src remote`), or scans an exported tarball or OCI layout with `--input`. No daemon is required.

### Q7. What output formats does Trivy support and when would you use SARIF? (Junior)

**Answer.** `table`, `json`, `sarif`, `cyclonedx`, `spdx`, `spdx-json`, `github`, `template`, `cosign-vuln`. SARIF is the standard for static analysis results; upload it to GitHub code scanning (or Azure DevOps, DefectDojo) so findings appear inline on pull requests.

---

## Mid-level

### Q8. Why does the same CVE show as fixed on Ubuntu, affected on Alpine, and absent on RHEL? (Mid-level)

**Answer.** Trivy uses each distribution's own security feed and version comparison for OS packages because distros backport fixes without changing upstream version numbers. Ubuntu may have shipped a patched `-0ubuntu0.1` revision, Alpine may not have released a fix yet, and Red Hat may have rated the package as not affected in its OVAL data. NVD's generic version range would be wrong for all three.

### Q9. What does the `Status` column mean and how does it affect gating? (Mid-level)

**Answer.** `fixed` means a fixed version exists; `affected` means vulnerable with no fix; `will_not_fix` and `fix_deferred` mean the vendor declined or postponed; `end_of_life` means the OS version has no feed; `under_investigation` means pending vendor analysis. `--ignore-unfixed` suppresses everything except `fixed`; `--ignore-status` lets you choose individual statuses.

### Q10. How does Trivy detect Java dependencies inside an image, and what is the Java DB? (Mid-level)

**Answer.** It walks `*.jar/war/ear` files, reads `META-INF/maven/*/pom.properties` for group/artifact/version, and recurses into nested jars. When `pom.properties` is missing (shaded or repackaged jars) it computes the jar's SHA-1 and looks it up in `trivy-java-db`, a Maven Central index downloaded lazily. Locally built jars without published hashes cannot be identified this way.

### Q11. Explain layer caching and why it matters for a CI fleet. (Mid-level)

**Answer.** Analyzer results are cached per layer digest. Two images sharing a base analyse the base once. On a CI fleet, use `--cache-backend redis://...` so every runner benefits, and cache the DB directory with `actions/cache` keyed by date so each job skips the ~60 MB download.

### Q12. How do you scan Terraform so that variable values are resolved? (Mid-level)

**Answer.** Trivy's HCL evaluator resolves variables, locals, and modules on its own, and `--tf-vars prod.tfvars` supplies environment values. For full fidelity including data sources and computed values, scan the plan: `terraform show -json plan.out > plan.json && trivy config plan.json`.

### Q13. What is VEX and how does it differ from an ignore file? (Mid-level)

**Answer.** VEX (Vulnerability Exploitability eXchange) is a supplier statement that a specific product digest is `not_affected`, `affected`, `fixed`, or `under_investigation` by a CVE, with a justification. Unlike an ignore file it is tied to a product and subcomponent purl, authored, timestamped, and can be signed and attached to the image. Trivy consumes OpenVEX, CycloneDX VEX, and CSAF via `--vex file|oci|repo`.

### Q14. How would you integrate Trivy into GitHub Actions so findings appear on pull requests? (Mid-level)

**Answer.** Build the image, run `aquasecurity/trivy-action` with `format: sarif` and `output: trivy.sarif`, upload with `github/codeql-action/upload-sarif` (`permissions: security-events: write`), then run a second Trivy step with `exit-code: 1`, `severity: HIGH,CRITICAL`, `ignore-unfixed: true` to gate. Pin the Trivy version and cache the DB.

### Q15. What is the Trivy Operator and what CRDs does it create? (Mid-level)

**Answer.** A Kubernetes operator that continuously scans workloads and writes results as CRDs owned by the workload: `VulnerabilityReport`, `ConfigAuditReport`, `ExposedSecretReport`, `SbomReport`, `RbacAssessmentReport`, `ClusterRbacAssessmentReport`, `InfraAssessmentReport`, and `ClusterComplianceReport` (CIS, NSA, PSS). It exports Prometheus metrics.

### Q16. How do you write a custom misconfiguration check? (Mid-level)

**Answer.** Write a Rego package under a custom namespace (for example `user.kubernetes.ORG0001`) with a `METADATA` comment block declaring `id`, `severity`, `title`, and an input selector, and a `deny` rule that yields `result.new(message, object)`. Load with `trivy config --config-check ./checks --check-namespaces user`. Unit test with `opa test`.

---

## Senior

### Q17. Trivy and Snyk disagree on a Python image: Trivy reports 12 CVEs, Snyk reports 3. Explain and decide. (Senior)

**Answer.** Trivy scans the final filesystem including OS packages and `site-packages`; Snyk `test` on the repo scans the lockfile and Snyk Container applies reachability heuristics. The extra nine are almost certainly OS-level (`libexpat`, `zlib`, `openssl`) which Snyk's SCA never saw. Neither is wrong; they answer different questions. Gate on Trivy for the image, use Snyk for developer-facing lockfile fixes, send both to DefectDojo for deduplication, and write VEX for OS findings that are present but unreachable.

### Q18. Design an air-gapped Trivy deployment. (Senior)

**Answer.** Mirror `trivy-db`, `trivy-java-db`, and `trivy-checks` from GHCR into an internal registry on a schedule using `oras` from a connected bastion; point runners at `--db-repository`, `--java-db-repository`, `--checks-bundle-repository`; set `--skip-db-update` where no registry is reachable and import by copying the cache directory; use `--offline-scan` to stop Maven Central lookups; consider `trivy server` inside the enclave so clients share one DB. Alert on DB age (`metadata.json` `UpdatedAt`) exceeding 48 hours.

### Q19. How do you reduce false positives without blinding the pipeline? (Senior)

**Answer.** First classify: wrong package version detection (fix with `--list-all-pkgs` and file patterns), distro status disagreement (trust the distro feed), unreachable code (VEX `not_affected` with justification), test fixtures (scoped `paths` in `.trivyignore.yaml`), or policy-level noise (Rego `--ignore-policy` for LOW severity or kernel headers). Every suppression carries a statement and `expired_at`; VEX documents are signed and attached to the image so the admission controller can honour them too.

### Q20. Compare Trivy with Grype and Syft for a regulated pipeline. (Senior)

**Answer.** Trivy is one binary covering vulnerabilities, misconfigurations, secrets, licenses, SBOM generation, and Kubernetes. Grype is a dedicated vulnerability scanner that consumes Syft SBOMs natively; Syft is the de facto SBOM generator with richer cataloger control. In regulated pipelines teams often generate the SBOM once with Syft at build time, scan with Trivy in the gate and Grype nightly against stored SBOMs (second opinion, different DB), and use Trivy for IaC and the Operator. Running two scanners with independent databases is itself a control.

### Q21. Explain how `trivy sbom` quality depends on the SBOM producer. (Senior)

**Answer.** Trivy reconstructs the package list from `purl`s in the SBOM. If the producer omitted OS packages (build-tool generators like `cyclonedx-maven-plugin` only know application dependencies), or emitted purls without `distro` qualifiers, OS matching is impossible or degraded. Verify SBOM completeness (NTIA minimum elements, component counts versus `--list-all-pkgs`) before relying on SBOM scanning as the sole gate.

### Q22. How does the Trivy Operator scale on a 2,000-pod cluster, and what are the knobs? (Senior)

**Answer.** It scans per unique image, not per pod, and creates scan Jobs limited by `operator.scanJobsConcurrentLimit`. Use `trivy.mode=ClientServer` with a `trivy server` Deployment so Jobs do not each download the DB; set `operator.scanJobTimeout`, exclude namespaces (`operator.excludeNamespaces`), restrict kinds (`operator.targetWorkloads`), tune `operator.vulnerabilityScannerReportTTL` for rescans, and give scan Jobs a node selector. Watch `trivy-operator` memory; large SBOMs are the usual culprit.

### Q23. How do Trivy findings become Cosign attestations and get enforced? (Senior)

**Answer.** `trivy image --format cosign-vuln` emits a predicate conforming to the Cosign vulnerability attestation schema; `cosign attest --type vuln --predicate vuln.json <image>@<digest>` signs it as an in-toto statement whose subject is the digest. At admission, Kyverno `verifyImages.attestations` with `type: https://cosign.sigstore.dev/attestation/vuln/v1` checks the signer identity and evaluates `conditions` on predicate fields such as `scanner.result.summary.critical == 0`.

---

## Principal

### Q24. A zero-day lands at 09:00 (think Log4Shell). Describe the Trivy-centric response plan. (Principal)

**Answer.** The plan is only as good as the pre-work: SBOMs for every artefact in Dependency-Track, Trivy Operator reports in every cluster, and a DB mirror we control. Response: (1) query Dependency-Track and `kubectl get vulnerabilityreports -A` for the package, usually within minutes; (2) if trivy-db has not yet ingested the advisory, add a temporary Rego check or secret-style rule for the artefact name/version so the pipeline blocks immediately; (3) once trivy-db updates (6 h cycle), force the Operator to rescan by deleting reports; (4) prioritise by exposure using ConfigAuditReports (internet-facing Services) and Falco runtime signals; (5) the fix PRs flow through the same gates; (6) afterwards, a postmortem on inventory gaps. The lesson is that Trivy's value in a crisis is inventory speed, not detection novelty.

### Q25. Argue for and against making Trivy a hard blocking gate versus advisory-only. (Principal)

**Answer.** For blocking: it forces base image hygiene, keeps the fixable backlog near zero, and makes the attestation at admission meaningful. Against: DB updates can turn a green main branch red overnight with no code change, breaking deploys during incidents; severity inflation (NVD CRITICAL on an unreachable library) generates toil; and teams start reaching for blanket ignores. The mature position: block on `fixed` HIGH/CRITICAL only, exempt via signed VEX with expiry, run the advisory full scan into DefectDojo with SLAs (CRITICAL 7 days, HIGH 30), and provide a documented break-glass label that skips the gate but creates a ticket and pages AppSec.

### Q26. How would you evaluate Trivy's scanning accuracy for your organisation rather than trusting vendor claims? (Principal)

**Answer.** Build a benchmark corpus: your top 50 production images plus deliberately vulnerable references (for example images pinned to known-bad versions). Run Trivy, Grype, and a commercial scanner; normalise by `purl` + CVE; compute agreement, unique-to-tool sets, and manually adjudicate a sample of disagreements against distro trackers. Measure time to detect for newly published CVEs over a month. Publish the result as an ADR. Repeat annually; databases drift.

### Q27. What are the security risks of running Trivy itself, and how do you mitigate them? (Principal)

**Answer.** Trivy parses untrusted content (tarballs, jars, lockfiles, HCL) so parser vulnerabilities are a real attack surface; run it unprivileged, pinned, in an ephemeral runner, never with the Docker socket in CI when `--input` or registry pulls suffice. DB supply chain: verify the Trivy release signature with Cosign and pin `--db-repository` to a mirror you control. Secret scanning output can itself leak secrets into logs; redact (`--format json` with `Match` redaction is default) and restrict artefact retention. The Operator needs read RBAC cluster-wide; scope it and audit its ServiceAccount.

### Q28. How do Trivy's misconfiguration checks relate to Checkov and Kyverno in a defence-in-depth model? (Principal)

**Answer.** Trivy `config` and Checkov both operate on IaC before apply (plan-time), overlapping heavily on Terraform/K8s/Dockerfile checks; Checkov has graph-based cross-resource queries and a Python policy API, Trivy has Rego and the advantage of being the same binary as the image scanner. Kyverno operates at apply-time on live API requests and catches everything that bypassed CI (kubectl, Helm from a laptop). The pattern is: shift left with one IaC scanner as the developer feedback loop, enforce at admission as the control, and make the admission policy the source of truth that the IaC checks approximate. Document the mapping (KSV012 ↔ Kyverno `require-run-as-nonroot`) so findings read the same in both places.

### Q29. The Operator's `ClusterComplianceReport` says CIS 5.2.5 fails on 40 workloads. Walk through turning that into closed findings. (Principal)

**Answer.** 5.2.5 is "minimise the admission of containers with allowPrivilegeEscalation". Export the report, map the failing `ConfigAuditReport` entries to owning teams via the `owner` label the Kyverno require-labels policy enforces, open DefectDojo findings per workload with the compliance control attached, give an SLA, and add a Kyverno policy in Audit mode for `allowPrivilegeEscalation: false` so PolicyReports track remediation in real time. When the count hits zero, flip Kyverno to Enforce so regression is impossible, and the next compliance cron proves it.

### Q30. What would make you *not* choose Trivy? (Principal)

**Answer.** When reachability analysis is mandatory for developer-facing triage and the budget exists (commercial SCA); when the organisation already runs Anchore Enterprise or Prisma Cloud with policy engines teams know; when license compliance is the primary goal (dedicated tooling is more accurate); or when the sole requirement is secrets in Git history (Gitleaks/TruffleHog). Even then Trivy usually stays as the free second opinion and as the Kubernetes Operator, because its cost is near zero and its CRD model integrates cleanly with Prometheus and GitOps.
