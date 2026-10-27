# Trivy: Hands-On Labs

| | |
|---|---|
| **Cluster** | 08 DevSecOps |
| **Role** | Anchor |
| **Prerequisites** | [01-installation-and-setup.md](01-installation-and-setup.md), Docker, `kind`, `kubectl`, `helm`, a GitHub repository you can push to |
| **Target version** | Trivy 0.56+, Trivy Operator 0.24+, Kubernetes 1.31+ |
| **Capstone phase** | Phase 5 (Months 17-20) |

> Nine labs that take you from a first image scan to a Kubernetes operator feeding compliance reports into Prometheus. Each lab builds on the previous one and leaves an artefact in the capstone repository. Expected outputs are representative; CVE counts drift with the database date.

Labs use a small FastAPI service. Create it once:

```bash
mkdir -p ~/labs/trivy/api && cd ~/labs/trivy/api
cat > main.py <<'EOF'
from fastapi import FastAPI
app = FastAPI()

@app.get("/healthz")
def healthz():
    return {"status": "ok"}
EOF
cat > requirements.txt <<'EOF'
fastapi==0.110.0
uvicorn==0.27.0
requests==2.31.0
EOF
cat > Dockerfile <<'EOF'
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY main.py .
EXPOSE 8000
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
EOF
docker build -t lab-api:v1 .
```

The Dockerfile is deliberately imperfect (no `USER`, old pins) so the labs have something to find.

---

## Lab 1: First Scan and Reading the Report

**Objective.** Scan `lab-api:v1`, understand every column, and separate OS from application findings.

**Prerequisites.** Trivy installed, image built above.

**Time.** 20 minutes.

### Steps

1. Run the default scan.

   ```bash
   trivy image lab-api:v1
   ```

   Expected output (abridged):

   ```text
   lab-api:v1 (debian 12.7)
   ========================
   Total: 58 (UNKNOWN: 0, LOW: 41, MEDIUM: 12, HIGH: 4, CRITICAL: 1)

   ┌────────────────┬────────────────┬──────────┬──────────────┬───────────────────┬───────────────┬──────────────────────────────┐
   │    Library     │ Vulnerability  │ Severity │    Status    │ Installed Version │ Fixed Version │            Title             │
   ├────────────────┼────────────────┼──────────┼──────────────┼───────────────────┼───────────────┼──────────────────────────────┤
   │ libexpat1      │ CVE-2024-45490 │ HIGH     │ fixed        │ 2.5.0-1           │ 2.5.0-1+deb12u1│ libexpat: Negative Length... │
   │ zlib1g         │ CVE-2023-45853 │ CRITICAL │ will_not_fix │ 1:1.2.13.dfsg-1   │               │ zlib: integer overflow...    │
   ...

   Python (python-pkg)
   ===================
   Total: 3 (UNKNOWN: 0, LOW: 0, MEDIUM: 1, HIGH: 2, CRITICAL: 0)

   ┌──────────────────────┬────────────────┬──────────┬────────┬───────────────────┬───────────────┬────────────────────────────┐
   │       Library        │ Vulnerability  │ Severity │ Status │ Installed Version │ Fixed Version │           Title            │
   ├──────────────────────┼────────────────┼──────────┼────────┼───────────────────┼───────────────┼────────────────────────────┤
   │ fastapi (METADATA)   │ CVE-2024-24762 │ HIGH     │ fixed  │ 0.110.0           │ 0.109.1, ...  │ python-multipart ReDoS...  │
   │ requests (METADATA)  │ CVE-2024-35195 │ MEDIUM   │ fixed  │ 2.31.0            │ 2.32.0        │ requests: subsequent req...│
   └──────────────────────┴────────────────┴──────────┴────────┴───────────────────┴───────────────┴────────────────────────────┘
   ```

2. Note the two result blocks: the OS block (`debian 12.7`, from `/var/lib/dpkg/status`) and the Python block (from `site-packages/*.dist-info/METADATA`). The `Status` column tells you which are actionable.

3. Produce the gate view only.

   ```bash
   trivy image --severity HIGH,CRITICAL --ignore-unfixed lab-api:v1
   ```

   The zlib CRITICAL disappears (`will_not_fix`), the fixable ones remain.

4. See which layer introduced a finding.

   ```bash
   trivy image --format json lab-api:v1 \
     | jq -r '.Results[] | select(.Class=="lang-pkgs") | .Vulnerabilities[] | "\(.VulnerabilityID) \(.PkgName) layer=\(.Layer.DiffID[7:19])"'
   docker history --no-trunc lab-api:v1 | head
   ```

   Match the `DiffID` prefix to the `RUN pip install` layer.

### Verify

`trivy image --severity HIGH,CRITICAL --ignore-unfixed --exit-code 1 lab-api:v1; echo "exit=$?"` prints `exit=1`.

### Cleanup

None.

### What you learned

How Trivy groups results by target, what `Status` means for gating, and how to trace a finding back to a Dockerfile instruction.

---

## Lab 2: Fix the Image and Prove It

**Objective.** Remediate the fixable findings by bumping dependencies and the base image, then show the gate passing.

**Prerequisites.** Lab 1.

**Time.** 20 minutes.

### Steps

1. Ask Trivy what the fixed versions are, in machine-readable form.

   ```bash
   trivy image --format json --severity HIGH,CRITICAL --ignore-unfixed lab-api:v1 \
     | jq -r '.Results[].Vulnerabilities[]? | "\(.PkgName)\t\(.InstalledVersion)\t->\t\(.FixedVersion)"' | sort -u
   ```

2. Update `requirements.txt` to the fixed versions and rebuild on a fresh base pull.

   ```bash
   cat > requirements.txt <<'EOF'
   fastapi==0.115.0
   uvicorn==0.30.6
   requests==2.32.3
   EOF
   docker build --pull -t lab-api:v2 .
   ```

3. Re-run the gate.

   ```bash
   trivy image --severity HIGH,CRITICAL --ignore-unfixed --exit-code 1 lab-api:v2; echo "exit=$?"
   ```

   Expected output:

   ```text
   lab-api:v2 (debian 12.7)
   ========================
   Total: 0 (HIGH: 0, CRITICAL: 0)
   exit=0
   ```

   If OS findings remain with `fixed` status, the base image tag has not been rebuilt upstream yet; switch to `python:3.12-slim-bookworm` with an explicit digest, or to a distroless/Chainguard base.

4. Compare base images.

   ```bash
   for img in python:3.12-slim python:3.12-alpine cgr.dev/chainguard/python:latest gcr.io/distroless/python3-debian12; do
     printf "%-45s " "$img"
     trivy image --quiet --format json --scanners vuln "$img" | jq -c '[.Results[].Vulnerabilities[]?] | group_by(.Severity) | map({(.[0].Severity): length}) | add // {}'
   done
   ```

   Expected (illustrative):

   ```text
   python:3.12-slim                              {"HIGH":2,"LOW":41,"MEDIUM":12,"CRITICAL":1}
   python:3.12-alpine                            {"MEDIUM":1}
   cgr.dev/chainguard/python:latest              {}
   gcr.io/distroless/python3-debian12            {"LOW":12,"MEDIUM":3}
   ```

### Verify

Gate exits `0` on `lab-api:v2`.

### Cleanup

None.

### What you learned

Remediation is mostly base image choice and pin bumps. Minimal bases shrink the finding count because there is less installed, which is the xz-utils lesson in practice.

---

## Lab 3: Ignore Files, Expiry and VEX

**Objective.** Suppress an unfixable finding three ways and understand the audit trail each leaves.

**Prerequisites.** Lab 1.

**Time.** 30 minutes.

### Steps

1. Show the CRITICAL that `--ignore-unfixed` hides, because a security reviewer wants it tracked explicitly rather than filtered.

   ```bash
   trivy image --severity CRITICAL lab-api:v1
   ```

2. Flat ignore (the wrong way for prod, fine for a demo).

   ```bash
   echo "CVE-2023-45853" > .trivyignore
   trivy image --severity CRITICAL lab-api:v1
   ```

   The finding disappears with no record of why.

3. Structured ignore with expiry and scope.

   ```bash
   rm .trivyignore
   cat > .trivyignore.yaml <<'EOF'
   vulnerabilities:
     - id: CVE-2023-45853
       purls:
         - "pkg:deb/debian/zlib1g"
       statement: "minizip not compiled into Debian zlib1g; tracked in SEC-421"
       expired_at: 2027-01-31
   EOF
   trivy image --severity CRITICAL --ignorefile .trivyignore.yaml lab-api:v1
   trivy image --severity CRITICAL --ignorefile .trivyignore.yaml --show-suppressed lab-api:v1
   ```

   Expected (with `--show-suppressed`):

   ```text
   Suppressed Vulnerabilities (Total: 1)
   ┌─────────┬────────────────┬──────────┬────────────┬───────────────────────────────────────────┬────────────────────┐
   │ Library │ Vulnerability  │ Severity │   Status   │                 Statement                 │       Source       │
   ├─────────┼────────────────┼──────────┼────────────┼───────────────────────────────────────────┼────────────────────┤
   │ zlib1g  │ CVE-2023-45853 │ CRITICAL │ ignored    │ minizip not compiled into Debian zlib1g...│ .trivyignore.yaml  │
   └─────────┴────────────────┴──────────┴────────────┴───────────────────────────────────────────┴────────────────────┘
   ```

4. Test expiry by setting `expired_at: 2020-01-01` and re-running; the finding returns.

5. OpenVEX document for the same finding, bound to the image digest.

   ```bash
   DIGEST=$(docker inspect --format='{{index .RepoDigests 0}}' lab-api:v1 2>/dev/null || docker image inspect lab-api:v1 --format '{{.Id}}')
   cat > lab-api.openvex.json <<EOF
   {
     "@context": "https://openvex.dev/ns/v0.2.0",
     "@id": "https://example.com/vex/lab-api-$(date +%Y%m%d)",
     "author": "Example Org Security",
     "timestamp": "$(date -u +%Y-%m-%dT%H:%M:%SZ)",
     "version": 1,
     "statements": [{
       "vulnerability": {"name": "CVE-2023-45853"},
       "products": [{"@id": "pkg:oci/lab-api?repository_url=localhost",
                     "subcomponents": [{"@id": "pkg:deb/debian/zlib1g"}]}],
       "status": "not_affected",
       "justification": "vulnerable_code_not_present",
       "impact_statement": "Debian zlib1g does not ship minizip"
     }]
   }
   EOF
   trivy image --severity CRITICAL --vex lab-api.openvex.json --show-suppressed lab-api:v1
   ```

   The `Source` column now reads `OpenVEX`.

### Verify

`trivy image --format json --vex lab-api.openvex.json lab-api:v1 | jq '.Results[].ExperimentalModifiedFindings'` lists the VEX-filtered finding with its status and statement.

### Cleanup

Keep `.trivyignore.yaml` and the VEX file for Lab 7.

### What you learned

Suppression is a spectrum: unexplained flat ignore, scoped and expiring YAML, and signed per-digest VEX. Choose by how far the artefact travels.

---

## Lab 4: IaC Misconfiguration Scanning with Built-in and Custom Checks

**Objective.** Scan Terraform, Kubernetes manifests, and the Dockerfile; write and unit-test a custom Rego check.

**Prerequisites.** Trivy installed; `opa` CLI for testing (optional).

**Time.** 45 minutes.

### Steps

1. Create insecure IaC fixtures.

   ```bash
   mkdir -p ~/labs/trivy/iac/{terraform,k8s} && cd ~/labs/trivy/iac
   cat > terraform/s3.tf <<'EOF'
   resource "aws_s3_bucket" "artifacts" {
     bucket = "example-org-lab-artifacts"
   }

   resource "aws_security_group" "web" {
     name = "web"
     ingress {
       from_port   = 22
       to_port     = 22
       protocol    = "tcp"
       cidr_blocks = ["0.0.0.0/0"]
     }
   }
   EOF
   cat > k8s/deploy.yaml <<'EOF'
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: api
   spec:
     replicas: 1
     selector:
       matchLabels: {app: api}
     template:
       metadata:
         labels: {app: api}
       spec:
         containers:
           - name: api
             image: docker.io/library/nginx:latest
             securityContext:
               privileged: true
   EOF
   ```

2. Scan both.

   ```bash
   trivy config --severity HIGH,CRITICAL .
   ```

   Expected (abridged):

   ```text
   terraform/s3.tf (terraform)
   ===========================
   Tests: 12 (SUCCESSES: 3, FAILURES: 9)
   Failures: 9 (HIGH: 5, CRITICAL: 1)

   CRITICAL: Security group rule allows ingress from public internet.
   ═══════════════════════════════════════════════════════════════
   See https://avd.aquasec.com/misconfig/avd-aws-0107
   ──────────────────────────────────────────────────────────────
    terraform/s3.tf:12
   ──────────────────────────────────────────────────────────────
     12 [     cidr_blocks = ["0.0.0.0/0"]
   ──────────────────────────────────────────────────────────────

   HIGH: Bucket does not have encryption enabled
   ...
   k8s/deploy.yaml (kubernetes)
   ============================
   Tests: 93 (SUCCESSES: 80, FAILURES: 13)
   Failures: 13 (HIGH: 3, CRITICAL: 1)

   CRITICAL: Container 'api' of Deployment 'api' should set 'securityContext.privileged' to false
   See https://avd.aquasec.com/misconfig/ksv017
   ```

3. Write a custom check that enforces the organisation registry.

   ```bash
   mkdir -p checks
   cat > checks/registry.rego <<'EOF'
   # METADATA
   # title: Images must come from the organisation registry
   # description: Only ghcr.io/example-org images may be deployed.
   # scope: package
   # schemas:
   #   - input: schema["kubernetes"]
   # custom:
   #   id: ORG-K8S-0001
   #   avd_id: ORG-K8S-0001
   #   severity: HIGH
   #   short_code: org-registry-only
   #   recommended_action: Use ghcr.io/example-org/<image>@sha256:<digest>
   #   input:
   #     selector:
   #       - type: kubernetes
   #         subtypes:
   #           - kind: Deployment
   #           - kind: StatefulSet
   #           - kind: DaemonSet
   #           - kind: Pod
   package user.kubernetes.ORG0001

   import rego.v1

   containers contains c if {
       some c in input.spec.template.spec.containers
   }

   containers contains c if {
       some c in input.spec.containers
   }

   deny contains res if {
       some c in containers
       not startswith(c.image, "ghcr.io/example-org/")
       res := result.new(
           sprintf("container %q uses image %q outside ghcr.io/example-org", [c.name, c.image]),
           c,
       )
   }
   EOF
   trivy config --config-check ./checks --check-namespaces user --severity HIGH k8s/
   ```

   Expected additional finding:

   ```text
   HIGH: container "api" uses image "docker.io/library/nginx:latest" outside ghcr.io/example-org
   ═══════════════════════════════════════════════════════════════
   Only ghcr.io/example-org images may be deployed.
   See https://avd.aquasec.com/misconfig/org-k8s-0001
   ```

4. Unit-test the Rego with OPA (the same skill as Gate 5).

   ```bash
   cat > checks/registry_test.rego <<'EOF'
   package user.kubernetes.ORG0001_test

   import rego.v1
   import data.user.kubernetes.ORG0001 as policy

   test_denies_external_image if {
       count(policy.deny) == 1 with input as {"spec": {"template": {"spec": {"containers": [{"name": "x", "image": "nginx:latest"}]}}}}
   }

   test_allows_org_image if {
       count(policy.deny) == 0 with input as {"spec": {"template": {"spec": {"containers": [{"name": "x", "image": "ghcr.io/example-org/api@sha256:abc"}]}}}}
   }
   EOF
   opa test checks/ -v
   ```

   Expected:

   ```text
   checks/registry_test.rego:
   data.user.kubernetes.ORG0001_test.test_denies_external_image: PASS (1.2ms)
   data.user.kubernetes.ORG0001_test.test_allows_org_image: PASS (0.8ms)
   --------------------------------------------------------------------------------
   PASS: 2/2
   ```

   `result.new` is a Trivy built-in, so in `opa test` it is undefined; if that breaks the test, stub it with `with result.new as ...` or test a helper rule that returns the message only.

5. Scan the Dockerfile from Lab 1.

   ```bash
   trivy config ~/labs/trivy/api/Dockerfile
   ```

   Expected: `DS002 (HIGH) Image user should not be 'root'`, `DS026 (LOW) No HEALTHCHECK defined`.

### Verify

`trivy config --config-check ./checks --check-namespaces user --severity HIGH --exit-code 1 . ; echo $?` prints `1`.

### Cleanup

Keep the `checks/` directory; it moves into the capstone under `security/trivy/checks/`.

### What you learned

Trivy's IaC scanner is a Rego engine with a rich built-in library; custom checks are small, testable, and reuse the Gate 5 skill set.

---

## Lab 5: Secrets and Licenses

**Objective.** Detect a planted secret in a repo and in an image's `ENV`, write an allow-rule, and run a license scan with custom buckets.

**Prerequisites.** Lab 1.

**Time.** 25 minutes.

### Steps

1. Plant a fake AWS key (the documented AWS example key, which is not a real credential).

   ```bash
   cd ~/labs/trivy/api
   cat > config.py <<'EOF'
   AWS_ACCESS_KEY_ID = "AKIAIOSFODNN7EXAMPLE"
   AWS_SECRET_ACCESS_KEY = "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
   EOF
   trivy fs --scanners secret .
   ```

   Expected:

   ```text
   config.py (secrets)
   ===================
   Total: 2 (UNKNOWN: 0, LOW: 0, MEDIUM: 0, HIGH: 0, CRITICAL: 2)

   CRITICAL: AWS (aws-access-key-id)
   ════════════════════════════════════
   AWS Access Key ID
   ────────────────────────────────────
    config.py:1
   ────────────────────────────────────
      1 [ AWS_ACCESS_KEY_ID = "AKIA****************"
   ```

2. Bake it into an image `ENV` and detect via image config scanning.

   ```bash
   cat > Dockerfile.secret <<'EOF'
   FROM python:3.12-slim
   ENV AWS_ACCESS_KEY_ID=AKIAIOSFODNN7EXAMPLE
   EOF
   docker build -f Dockerfile.secret -t lab-api:secret .
   trivy image --scanners secret --image-config-scanners secret lab-api:secret
   ```

3. Allow the AWS documentation example key only.

   ```bash
   cat > trivy-secret.yaml <<'EOF'
   allow-rules:
     - id: aws-docs-example
       description: AWS documentation example key, not a credential
       regex: 'AKIAIOSFODNN7EXAMPLE|wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY'
   EOF
   trivy fs --scanners secret --secret-config trivy-secret.yaml .
   ```

   Expected: no findings.

4. Licenses.

   ```bash
   trivy image --scanners license lab-api:v2
   cat > trivy.yaml <<'EOF'
   license:
     forbidden: ["AGPL-3.0", "SSPL-1.0"]
     restricted: ["GPL-3.0", "LGPL-3.0"]
   EOF
   trivy image --scanners license --severity HIGH,CRITICAL --config trivy.yaml lab-api:v2
   ```

### Verify

`trivy fs --scanners secret --exit-code 1 . ; echo $?` prints `1` without the allow-rule and `0` with it.

### Cleanup

```bash
rm config.py Dockerfile.secret && docker rmi lab-api:secret
```

### What you learned

Trivy catches secrets that reach the image, which Gate 1 tools cannot see. Allow-rules must be narrow and justified.

---

## Lab 6: SBOM Generation, SBOM Scanning and Cosign Attestation

**Objective.** Produce CycloneDX and SPDX SBOMs, scan the SBOM instead of the image, and attest it to a local registry.

**Prerequisites.** Lab 2, Cosign installed ([../cosign/01-installation-and-setup.md](../cosign/01-installation-and-setup.md)), a local registry.

**Time.** 40 minutes.

### Steps

1. Start a local registry and push by digest.

   ```bash
   docker run -d --name registry -p 5000:5000 registry:2
   docker tag lab-api:v2 localhost:5000/lab-api:v2
   docker push localhost:5000/lab-api:v2
   DIGEST=$(docker inspect --format='{{index .RepoDigests 0}}' localhost:5000/lab-api:v2 | cut -d@ -f2)
   IMAGE="localhost:5000/lab-api@${DIGEST}"
   echo "$IMAGE"
   ```

2. Generate SBOMs.

   ```bash
   trivy image --format cyclonedx --output sbom.cdx.json "$IMAGE"
   trivy image --format spdx-json --output sbom.spdx.json "$IMAGE"
   jq '.components | length' sbom.cdx.json
   jq '.metadata.component' sbom.cdx.json
   ```

   Expected:

   ```text
   101
   {
     "bom-ref": "pkg:oci/lab-api@sha256%3A...?repository_url=localhost%3A5000%2Flab-api&arch=amd64",
     "type": "container",
     "name": "localhost:5000/lab-api",
     "purl": "pkg:oci/lab-api@sha256%3A...",
     "properties": [ {"name": "aquasecurity:trivy:RepoDigest", "value": "localhost:5000/lab-api@sha256:..."} ]
   }
   ```

3. Scan the SBOM and compare with the image scan.

   ```bash
   trivy sbom --format json sbom.cdx.json | jq '[.Results[].Vulnerabilities[]?] | length'
   trivy image --format json "$IMAGE" | jq '[.Results[].Vulnerabilities[]?] | length'
   ```

   Counts match because the SBOM carries OS purls with `distro` qualifiers.

4. Attest the SBOM and a vulnerability report to the image (key-based for a local registry; keyless in Lab 8).

   ```bash
   export COSIGN_PASSWORD=""   # lab shortcut: empty password; never do this outside a throwaway lab
   cosign generate-key-pair
   cosign attest --yes --key cosign.key --predicate sbom.cdx.json --type cyclonedx "$IMAGE"
   trivy image --format cosign-vuln --output vuln.json "$IMAGE"
   cosign attest --yes --key cosign.key --predicate vuln.json --type vuln "$IMAGE"
   cosign tree "$IMAGE"
   ```

   Expected:

   ```text
   📦 Supply Chain Security Related artifacts for an image: localhost:5000/lab-api@sha256:...
   └── 💾 Attestations for an image tag: localhost:5000/lab-api:sha256-....att
      ├── 🍒 sha256:...
      └── 🍒 sha256:...
   ```

5. Scan using the attached SBOM instead of pulling layers.

   ```bash
   trivy image --sbom-sources oci "$IMAGE"
   ```

   Log shows `Detected SBOM  format="cyclonedx-json"`.

### Verify

`cosign verify-attestation --key cosign.pub --type cyclonedx "$IMAGE" | jq -r .payload | base64 -d | jq '.predicate.components | length'` prints the component count.

### Cleanup

Keep the registry for Lab 7 and 8. `rm cosign.key cosign.pub` at the end of the lab series.

### What you learned

The SBOM is a first-class scan target and a signable artefact; its quality determines what `trivy sbom` can find.

---

## Lab 7: Trivy in GitHub Actions with SARIF and Gating

**Objective.** A workflow that builds, scans, uploads SARIF to code scanning, gates on fixable HIGH/CRITICAL, and caches the DB.

**Prerequisites.** A GitHub repo containing the `api/` folder; GHCR enabled.

**Time.** 40 minutes.

### Steps

1. Create `.github/workflows/trivy.yml`.

   ```yaml
   name: trivy
   on:
     pull_request:
     push:
       branches: [main]
   permissions:
     contents: read
     packages: write
     security-events: write
     id-token: write
   env:
     IMAGE: ghcr.io/${{ github.repository }}/lab-api
   jobs:
     build-scan:
       runs-on: ubuntu-24.04
       steps:
         - uses: actions/checkout@v4

         - name: Log in to GHCR
           uses: docker/login-action@v3
           with:
             registry: ghcr.io
             username: ${{ github.actor }}
             password: ${{ secrets.GITHUB_TOKEN }}

         - name: Build (load locally, push later only if gate passes)
           id: build
           uses: docker/build-push-action@v6
           with:
             context: ./api
             load: true
             tags: ${{ env.IMAGE }}:${{ github.sha }}

         - name: Cache Trivy DB
           uses: actions/cache@v4
           with:
             path: ~/.cache/trivy
             key: trivy-db-${{ runner.os }}-${{ hashFiles('**/trivy.yaml') }}-${{ github.run_id }}
             restore-keys: trivy-db-${{ runner.os }}-

         - name: Trivy full report (SARIF, never fails)
           uses: aquasecurity/trivy-action@0.28.0
           with:
             image-ref: ${{ env.IMAGE }}:${{ github.sha }}
             format: sarif
             output: trivy-image.sarif
             severity: MEDIUM,HIGH,CRITICAL
             scanners: vuln,secret,misconfig
             ignore-unfixed: false
             exit-code: "0"
           env:
             TRIVY_SKIP_DB_UPDATE: false

         - name: Upload SARIF
           uses: github/codeql-action/upload-sarif@v3
           if: always()
           with:
             sarif_file: trivy-image.sarif
             category: trivy-image

         - name: Trivy gate (fixable HIGH/CRITICAL only)
           uses: aquasecurity/trivy-action@0.28.0
           with:
             image-ref: ${{ env.IMAGE }}:${{ github.sha }}
             format: table
             severity: HIGH,CRITICAL
             scanners: vuln
             ignore-unfixed: true
             exit-code: "1"
             trivyignores: api/.trivyignore.yaml
             skip-setup-trivy: true
           env:
             TRIVY_SKIP_DB_UPDATE: true
             TRIVY_SKIP_JAVA_DB_UPDATE: true

         - name: IaC scan
           uses: aquasecurity/trivy-action@0.28.0
           with:
             scan-type: config
             scan-ref: .
             format: sarif
             output: trivy-config.sarif
             severity: HIGH,CRITICAL
             exit-code: "0"
             skip-setup-trivy: true

         - name: Upload IaC SARIF
           uses: github/codeql-action/upload-sarif@v3
           if: always()
           with:
             sarif_file: trivy-config.sarif
             category: trivy-config

         - name: Push (only after gate)
           if: github.ref == 'refs/heads/main'
           run: docker push ${{ env.IMAGE }}:${{ github.sha }}
   ```

2. Commit and open a PR. The first step uploads a full SARIF so the Security tab and PR annotations show all findings; the second step fails the job only on fixable HIGH/CRITICAL. The second and third Trivy invocations reuse the DB downloaded by the first (`TRIVY_SKIP_DB_UPDATE: true`, `skip-setup-trivy: true`), which is the main speed trick.

3. Break it on purpose: pin `requests==2.31.0` again, push, and watch the gate step fail with the table and exit code 1. Fix and push again.

### Verify

Security tab → Code scanning shows `trivy-image` and `trivy-config` categories; the PR check `trivy / build-scan` is red then green.

### Cleanup

Keep the workflow.

### What you learned

Separate reporting from gating; cache the DB; scan IaC in the same job; push only after the gate.

---

## Lab 8: Trivy Operator on kind with Prometheus Metrics

**Objective.** Install the Operator, observe VulnerabilityReports and ConfigAuditReports, run a CIS compliance report, and scrape metrics.

**Prerequisites.** `kind`, `helm`, Lab 2 image pushed to a registry the cluster can pull from (use `kind load docker-image lab-api:v2` for simplicity).

**Time.** 60 minutes.

### Steps

1. Create a cluster and load the image.

   ```bash
   kind create cluster --name trivy-lab --image kindest/node:v1.31.2
   kind load docker-image lab-api:v2 --name trivy-lab
   kubectl create namespace demo
   kubectl -n demo create deployment api --image=lab-api:v2 --replicas=1
   kubectl -n demo set image deployment/api lab-api=lab-api:v2
   kubectl -n demo patch deployment api -p '{"spec":{"template":{"spec":{"containers":[{"name":"lab-api","imagePullPolicy":"IfNotPresent"}]}}}}'
   ```

2. Install the Operator.

   ```bash
   helm repo add aquasecurity https://aquasecurity.github.io/helm-charts/ && helm repo update
   helm install trivy-operator aquasecurity/trivy-operator \
     --namespace trivy-system --create-namespace --version 0.24.1 \
     --set trivy.ignoreUnfixed=true \
     --set operator.scanJobsConcurrentLimit=3 \
     --set operator.metricsVulnIdEnabled=true \
     --set operator.metricsExposedSecretInfo=true \
     --set operator.metricsConfigAuditInfo=true \
     --set compliance.cron="*/10 * * * *" \
     --set serviceMonitor.enabled=false
   kubectl -n trivy-system rollout status deploy/trivy-operator
   ```

3. Watch reports appear.

   ```bash
   kubectl get jobs -n trivy-system -w &
   sleep 90; kill %1
   kubectl get vulnerabilityreports -n demo -o wide
   kubectl get configauditreports -n demo -o wide
   ```

   Expected:

   ```text
   NAME                                 REPOSITORY   TAG   SCANNER   AGE   CRITICAL   HIGH   MEDIUM   LOW   UNKNOWN
   replicaset-api-6c9f7b8d4f-lab-api    lab-api      v2    Trivy     45s   0          0      0        0     0

   NAME                            SCANNER   AGE   CRITICAL   HIGH   MEDIUM   LOW
   replicaset-api-6c9f7b8d4f       Trivy     50s   0          2      5        3
   ```

   The ConfigAuditReport flags the Deployment created by `kubectl create` (no securityContext, no limits).

4. Inspect one report.

   ```bash
   kubectl get configauditreport -n demo -o jsonpath='{.items[0].report.checks[?(@.success==false)].checkID}' | tr ' ' '\n'
   ```

   Expected:

   ```text
   KSV001
   KSV003
   KSV011
   KSV012
   KSV014
   KSV015
   KSV016
   KSV018
   KSV020
   KSV021
   KSV030
   KSV106
   ```

5. Compliance.

   ```bash
   kubectl get clustercompliancereports
   kubectl get clustercompliancereport cis -o jsonpath='{.status.summary}'; echo
   kubectl get clustercompliancereport cis -o json | jq '.status.summaryReport.controlCheck[] | select(.totalFail > 0) | {id, name, totalFail}' | head -40
   ```

6. Metrics.

   ```bash
   kubectl -n trivy-system port-forward svc/trivy-operator 8080:80 &
   curl -s localhost:8080/metrics | grep -E '^trivy_(image_vulnerabilities|resource_configaudits)' | head
   kill %1
   ```

   Expected:

   ```text
   trivy_image_vulnerabilities{container_name="lab-api",image_digest="",image_registry="index.docker.io",image_repository="library/lab-api",image_tag="v2",name="replicaset-api-6c9f7b8d4f-lab-api",namespace="demo",resource_kind="ReplicaSet",resource_name="api-6c9f7b8d4f",severity="Critical"} 0
   trivy_resource_configaudits{...,severity="High"} 2
   ```

7. Fix the workload and watch the report reconcile.

   ```bash
   kubectl -n demo patch deployment api --type merge -p '{
     "spec":{"template":{"spec":{
       "securityContext":{"runAsNonRoot":true,"runAsUser":10001,"seccompProfile":{"type":"RuntimeDefault"}},
       "containers":[{"name":"lab-api",
         "securityContext":{"allowPrivilegeEscalation":false,"readOnlyRootFilesystem":true,"capabilities":{"drop":["ALL"]}},
         "resources":{"requests":{"cpu":"50m","memory":"64Mi"},"limits":{"cpu":"250m","memory":"128Mi"}}}]}}}}'
   sleep 30; kubectl get configauditreports -n demo -o wide
   ```

   HIGH drops to 0; the old ReplicaSet's report is garbage-collected with it.

### Verify

`kubectl get vulnerabilityreports,configauditreports -n demo` shows reports for the current ReplicaSet only.

### Cleanup

```bash
kind delete cluster --name trivy-lab
```

### What you learned

The Operator turns scanning into Kubernetes-native desired state: reports are owned by workloads, metrics are first-class, and compliance is a cron.

---

## Lab 9: Air-Gapped Operation and Client/Server Mode

**Objective.** Mirror the databases to a private registry, run Trivy with no internet, and share one DB across clients with `trivy server`.

**Prerequisites.** `oras` CLI, local registry from Lab 6.

**Time.** 40 minutes.

### Steps

1. Mirror the three OCI artefacts (run on a connected machine).

   ```bash
   for art in trivy-db:2 trivy-java-db:1 trivy-checks:1; do
     oras copy "ghcr.io/aquasecurity/${art}" "localhost:5000/aquasecurity/${art}"
   done
   oras repo tags localhost:5000/aquasecurity/trivy-db
   ```

   Expected: `2`.

2. Simulate the enclave: block GHCR and point Trivy at the mirror.

   ```bash
   trivy clean --all
   trivy image --db-repository localhost:5000/aquasecurity/trivy-db:2 \
               --java-db-repository localhost:5000/aquasecurity/trivy-java-db:1 \
               --checks-bundle-repository localhost:5000/aquasecurity/trivy-checks:1 \
               --insecure --offline-scan --download-db-only
   trivy image --skip-db-update --skip-java-db-update --skip-check-update --offline-scan lab-api:v2
   ```

   The second command performs zero network calls except to the local daemon.

3. Alternative: copy the cache directory as a tarball.

   ```bash
   tar -C ~/.cache -czf trivy-cache.tgz trivy/db trivy/java-db trivy/policy
   # on the air-gapped host:
   mkdir -p ~/.cache && tar -C ~/.cache -xzf trivy-cache.tgz
   trivy image --skip-db-update --skip-java-db-update --skip-check-update lab-api:v2
   ```

4. Server mode.

   ```bash
   trivy server --listen 127.0.0.1:4954 --skip-db-update --token lab-token &
   trivy image --server http://127.0.0.1:4954 --token lab-token lab-api:v2
   kill %1
   ```

   The client log shows `Scanning via server` and completes in a few seconds because no DB is loaded locally.

5. Alert on DB freshness (what you would run as a cron in the enclave).

   ```bash
   jq -r '.UpdatedAt' ~/.cache/trivy/db/metadata.json
   age_h=$(( ( $(date +%s) - $(date -d "$(jq -r .UpdatedAt ~/.cache/trivy/db/metadata.json)" +%s) ) / 3600 ))
   [ "$age_h" -gt 48 ] && echo "DB stale: ${age_h}h" || echo "DB fresh: ${age_h}h"
   ```

### Verify

Scan completes with `TRIVY_DEBUG=true` showing no `ghcr.io` requests.

### Cleanup

```bash
docker rm -f registry
rm -rf ~/labs/trivy
```

### What you learned

Trivy's dependencies are three OCI artefacts; mirror them and you own the whole supply chain of your scanner.

---

## Where to Go Next

[05-production-patterns.md](05-production-patterns.md) turns these labs into the hardened pipeline and operator configuration used in the capstone.
