# GitHub Actions: Hands-On Labs

| | |
|---|---|
| **Cluster** | 06 CI/CD Pipelines |
| **Role** | Anchor |
| **Prerequisites** | [01-installation-and-setup.md](01-installation-and-setup.md), [02-core-concepts.md](02-core-concepts.md), the capstone repo with `services/fastapi-api` and `services/springboot-api` skeletons, an AWS account you can create IAM roles in |
| **Target version** | GitHub.com (2025+), `actions/*@v4`, Python 3.12+, Java 21 LTS, Spring Boot 3.3+, Terraform 1.9+ |
| **Capstone phase** | Phase 1 (Labs 1-6), Phase 2 (Labs 7-8), Phase 5 (Lab 9-10) |

> Ten progressive labs. Labs 1-3 build the FastAPI and Spring Boot CI workflows with matrices, caching and Testcontainers. Labs 4-5 extract a composite action and a reusable workflow. Lab 6 builds, attests and signs the container image. Lab 7 federates to AWS with OIDC and runs Terraform plan-on-PR/apply-on-merge. Lab 8 adds the nightly destroy cron and environment approvals. Lab 9 deploys through Argo CD. Lab 10 is the full seven-gate DevSecOps pipeline. Every workflow here is the capstone's reference implementation.

Action SHAs below follow the pin-and-comment convention; re-resolve them when you adopt them (`gh api repos/<owner>/<repo>/git/ref/tags/<tag>`).

---

## Lab 1: FastAPI CI with uv, ruff, mypy and pytest

**Objective.** A PR-blocking workflow for `services/fastapi-api` that lints, type-checks and tests across a Python matrix, with dependency caching and a test summary.

**Prerequisites.** The FastAPI service has a `pyproject.toml` managed by uv, `uv.lock`, a `tests/` folder, and ruff/mypy configured.

**Time.** 45 minutes.

### Steps

1. Create `.github/workflows/ci-fastapi.yml`.

   ```yaml
   name: ci-fastapi

   on:
     push:
       branches: [main]
       paths: ['services/fastapi-api/**', '.github/workflows/ci-fastapi.yml']
     pull_request:
       branches: [main]
       paths: ['services/fastapi-api/**', '.github/workflows/ci-fastapi.yml']

   permissions:
     contents: read

   concurrency:
     group: ${{ github.workflow }}-${{ github.event.pull_request.number || github.ref }}
     cancel-in-progress: ${{ github.event_name == 'pull_request' }}

   defaults:
     run:
       working-directory: services/fastapi-api

   jobs:
     lint:
       runs-on: ubuntu-24.04
       timeout-minutes: 10
       steps:
         - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
           with: { persist-credentials: false }
         - uses: astral-sh/setup-uv@f3bcaebff5eace81a1c062af9f9011aae482ca9d # v3.1.7
           with:
             version: "0.5.x"
             enable-cache: true
             cache-dependency-glob: services/fastapi-api/uv.lock
         - run: uv python install 3.12
         - run: uv sync --frozen --all-extras
         - name: ruff (lint + format check)
           run: |
             uv run ruff check --output-format=github .
             uv run ruff format --check .
         - name: mypy
           run: uv run mypy app --strict

     test:
       needs: lint
       runs-on: ubuntu-24.04
       timeout-minutes: 15
       strategy:
         fail-fast: false
         matrix:
           python: ['3.12', '3.13']
           include:
             - python: '3.12'
               coverage: true
       services:
         postgres:
           image: postgres:16-alpine
           env:
             POSTGRES_USER: app
             POSTGRES_PASSWORD: localdev-only
             POSTGRES_DB: app_test
           ports: ['5432:5432']
           options: >-
             --health-cmd "pg_isready -U app"
             --health-interval 5s --health-timeout 5s --health-retries 10
       env:
         DATABASE_URL: postgresql+asyncpg://app:localdev-only@localhost:5432/app_test
       steps:
         - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
           with: { persist-credentials: false }
         - uses: astral-sh/setup-uv@f3bcaebff5eace81a1c062af9f9011aae482ca9d # v3.1.7
           with:
             version: "0.5.x"
             enable-cache: true
             cache-dependency-glob: services/fastapi-api/uv.lock
         - run: uv python install ${{ matrix.python }}
         - run: uv sync --frozen --all-extras --python ${{ matrix.python }}
         - name: pytest
           run: |
             uv run pytest -q \
               --junitxml=reports/junit.xml \
               ${{ matrix.coverage && '--cov=app --cov-report=xml:reports/coverage.xml --cov-fail-under=80' || '' }}
         - name: Test summary
           if: always()
           run: |
             total=$(grep -o 'tests="[0-9]*"' reports/junit.xml | head -1 | grep -o '[0-9]*')
             failures=$(grep -o 'failures="[0-9]*"' reports/junit.xml | head -1 | grep -o '[0-9]*')
             {
               echo "### pytest (Python ${{ matrix.python }})"
               echo "| Total | Failures |"; echo "|---|---|"; echo "| ${total} | ${failures} |"
             } >> "$GITHUB_STEP_SUMMARY"
         - uses: actions/upload-artifact@b4b15b8c7c6ac21ea08fcf65892d2ee8f75cf882 # v4.4.3
           if: always()
           with:
             name: fastapi-test-reports-${{ matrix.python }}
             path: services/fastapi-api/reports/
             retention-days: 14
   ```

2. Push a branch and open a PR.

   ```bash
   git switch -c ci/fastapi
   git add .github/workflows/ci-fastapi.yml && git commit -m "ci: fastapi lint/type/test matrix"
   git push -u origin ci/fastapi
   gh pr create --fill
   gh pr checks --watch
   ```

   Expected output:

   ```text
   All checks were successful
   ✓  ci-fastapi / lint          1m2s   https://github.com/...
   ✓  ci-fastapi / test (3.12)   1m48s  https://github.com/...
   ✓  ci-fastapi / test (3.13)   1m41s  https://github.com/...
   ```

3. Make the `test (3.12)` check required in branch protection (see [01-installation-and-setup.md](01-installation-and-setup.md#23-branch-protection-with-a-required-check)). The matrix leg name is `test (3.12)`.

### Verify

Rerun the workflow and confirm the uv cache was hit: the `setup-uv` step logs `Cache restored from key: setup-uv-1-x86_64-unknown-linux-gnu-<hash>`. Second-run duration should drop by roughly half.

### Cleanup

Merge or close the PR. Caches expire after 7 days unused.

### What you learned

Path filters, PR-only cancellation, job dependencies, matrix with `include`, service containers with health checks, JUnit artifacts and step summaries, and why the matrix leg name matters for required checks.

---

## Lab 2: Spring Boot CI with Maven, Gradle and Testcontainers

**Objective.** CI for `services/springboot-api` that builds with both Maven and Gradle (to prove the pattern) on Java 21, runs Testcontainers integration tests against a real Postgres, and publishes JUnit results as a check.

**Prerequisites.** Spring Boot 3.3+ project with `pom.xml` (and optionally `build.gradle.kts`), Testcontainers dependency, Docker available on the runner (GitHub-hosted Ubuntu has it).

**Time.** 45 minutes.

### Steps

1. Create `.github/workflows/ci-springboot.yml`.

   ```yaml
   name: ci-springboot

   on:
     push:
       branches: [main]
       paths: ['services/springboot-api/**', '.github/workflows/ci-springboot.yml']
     pull_request:
       paths: ['services/springboot-api/**', '.github/workflows/ci-springboot.yml']

   permissions:
     contents: read
     checks: write          # publish JUnit as a check run

   concurrency:
     group: ${{ github.workflow }}-${{ github.event.pull_request.number || github.ref }}
     cancel-in-progress: ${{ github.event_name == 'pull_request' }}

   defaults:
     run:
       working-directory: services/springboot-api

   jobs:
     build:
       runs-on: ubuntu-24.04
       timeout-minutes: 25
       strategy:
         fail-fast: false
         matrix:
           tool: [maven, gradle]
       env:
         TESTCONTAINERS_RYUK_DISABLED: "false"
         TESTCONTAINERS_REUSE_ENABLE: "false"
       steps:
         - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
           with: { persist-credentials: false }

         - uses: actions/setup-java@8df1039502a15bceb9433410b1a100fbe190c53b # v4.5.0
           with:
             distribution: temurin
             java-version: '21'
             cache: ${{ matrix.tool }}
             cache-dependency-path: |
               services/springboot-api/pom.xml
               services/springboot-api/*.gradle.kts

         - uses: gradle/actions/setup-gradle@d156388eb19639ec20ade50009f3d199ce1e2808 # v4.1.0
           if: matrix.tool == 'gradle'
           with:
             cache-read-only: ${{ github.ref != 'refs/heads/main' }}

         - name: Build and test (Maven)
           if: matrix.tool == 'maven'
           run: ./mvnw --batch-mode --no-transfer-progress verify

         - name: Build and test (Gradle)
           if: matrix.tool == 'gradle'
           run: ./gradlew build --no-daemon --scan=false

         - name: Publish JUnit results
           uses: dorny/test-reporter@31a54ee7ebcacc03a09ea97a7e5465a47b84aea5 # v1.9.1
           if: always()
           with:
             name: junit (${{ matrix.tool }})
             path: services/springboot-api/**/target/surefire-reports/*.xml,services/springboot-api/**/build/test-results/test/*.xml
             reporter: java-junit
             fail-on-error: true

         - uses: actions/upload-artifact@b4b15b8c7c6ac21ea08fcf65892d2ee8f75cf882 # v4.4.3
           if: matrix.tool == 'maven'
           with:
             name: springboot-jar
             path: services/springboot-api/target/*.jar
             if-no-files-found: error
   ```

2. Ensure the Testcontainers test uses the JDBC URL shortcut so no manual container wiring is needed:

   ```java
   @SpringBootTest
   @Testcontainers
   class OrderRepositoryIT {

       @Container
       @ServiceConnection
       static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine");

       @Autowired OrderRepository repository;

       @Test
       void savesAndLoads() {
           var saved = repository.save(new Order("ord-1", 42));
           assertThat(repository.findById(saved.id())).isPresent();
       }
   }
   ```

3. Push and watch. Expected check names: `ci-springboot / build (maven)`, `ci-springboot / build (gradle)`, plus two `junit (...)` check runs with per-test detail.

### Verify

In the job log for the Maven leg, confirm `Cache restored from key: setup-java-Linux-x64-maven-<hash>` on the second run and that Testcontainers logs `Container postgres:16-alpine started`.

### Cleanup

None required.

### What you learned

Toolchain caching through `setup-java`, Gradle's read-only cache on PRs to avoid cache thrash, Docker availability on hosted runners for Testcontainers, and publishing structured test results through the Checks API with `checks: write`.

---

## Lab 3: Composite Action for Toolchain Setup

**Objective.** Remove the duplicated uv setup from Lab 1 into a local composite action with an output.

**Prerequisites.** Lab 1.

**Time.** 20 minutes.

### Steps

1. Create `.github/actions/setup-python-uv/action.yml` (see [02-core-concepts.md](02-core-concepts.md#62-composite-action) for the full file).

2. Replace the three setup steps in `ci-fastapi.yml` with:

   ```yaml
   - uses: ./.github/actions/setup-python-uv
     id: py
     with:
       python-version: ${{ matrix.python || '3.12' }}
   - run: echo "cache-hit=${{ steps.py.outputs.cache-hit }}"
   ```

3. Push and confirm both jobs still pass.

### Verify

`gh run view --log | grep -i "cache-hit"` shows `true` on the second run.

### What you learned

Composite actions run inside the caller's job, must declare `shell:` on each `run`, and surface outputs via `runs.outputs`. Local actions (`./.github/actions/...`) require a prior `actions/checkout`.

---

## Lab 4: Reusable Workflow for Image Build

**Objective.** Move image building into a reusable workflow in a platform repository and call it from both services.

**Prerequisites.** Lab 1 and 2; a second repository `<ORG>/platform-workflows` (can be the same repo while learning, using `./.github/workflows/build-image.yml`).

**Time.** 30 minutes.

### Steps

1. Create `.github/workflows/build-image.yml` with the `workflow_call` definition from [02-core-concepts.md](02-core-concepts.md#61-reusable-workflow).

2. Add a caller job to `ci-fastapi.yml`:

   ```yaml
     image:
       needs: test
       if: github.event_name == 'push' && github.ref == 'refs/heads/main'
       uses: ./.github/workflows/build-image.yml
       permissions:
         contents: read
         packages: write
         id-token: write
         attestations: write
       with:
         context: services/fastapi-api
         image: ghcr.io/${{ github.repository }}/fastapi-api
   ```

3. Merge to `main`; the image job appears in the run graph as `image / build`.

### Verify

```bash
gh api "orgs/<ORG>/packages/container/capstone-platform%2Ffastapi-api/versions" --jq '.[0].metadata.container.tags'
```

### What you learned

Reusable workflows get their own runners and can declare their own permissions (never more than the caller grants). Outputs flow back via `jobs.<id>.outputs` → `on.workflow_call.outputs`. Local references (`./.github/workflows/x.yml`) use the caller's checkout ref; remote references must be pinned.

---

## Lab 5: Environments and Required Reviewers

**Objective.** Create `staging` (auto) and `production` (2 reviewers, `main` only, 5-minute wait) and gate a deploy job on them.

**Prerequisites.** A team or two collaborators to act as reviewers.

**Time.** 20 minutes.

### Steps

1. Create environments:

   ```bash
   gh api --method PUT repos/<OWNER>/capstone-platform/environments/staging
   gh api --method PUT repos/<OWNER>/capstone-platform/environments/production --input - <<'EOF'
   { "wait_timer": 5, "prevent_self_review": true,
     "reviewers": [{ "type": "User", "id": <USER_ID> }],
     "deployment_branch_policy": { "protected_branches": true, "custom_branch_policies": false } }
   EOF
   gh variable set DEPLOY_TARGET --env staging --body "staging-cluster"
   gh variable set DEPLOY_TARGET --env production --body "prod-cluster"
   ```

2. Add a deploy workflow stub `.github/workflows/deploy.yml`:

   ```yaml
   name: deploy
   on:
     workflow_dispatch:
       inputs:
         digest: { type: string, required: true, description: "Image digest (sha256:...)" }
   permissions: { contents: read }
   jobs:
     staging:
       runs-on: ubuntu-24.04
       environment: { name: staging, url: https://staging.example.com }
       steps:
         - run: echo "deploy ${DIGEST} to ${{ vars.DEPLOY_TARGET }}"
           env: { DIGEST: "${{ inputs.digest }}" }
     production:
       needs: staging
       runs-on: ubuntu-24.04
       environment: { name: production, url: https://api.example.com }
       steps:
         - run: echo "deploy ${DIGEST} to ${{ vars.DEPLOY_TARGET }}"
           env: { DIGEST: "${{ inputs.digest }}" }
   ```

3. Dispatch it and observe the production job waiting:

   ```bash
   gh workflow run deploy -f digest=sha256:0000000000000000000000000000000000000000000000000000000000000000
   gh run list --workflow deploy --limit 1
   ```

   ```text
   *  deploy  main  workflow_dispatch  12345678902  waiting
   ```

4. Approve from the UI (Review deployments) or via API as a different user.

### Verify

The Deployments tab shows one `staging` and one `production` deployment with status `success` and the environment URL.

### What you learned

Environment variables and secrets are only readable after protection rules pass; `prevent_self_review` stops the deployer from approving their own change; the run shows `waiting` rather than consuming runner minutes while blocked.

---

## Lab 6: Docker Build with buildx, Cache, Attestations and Cosign

**Objective.** Build the FastAPI image multi-arch with layer caching, generate SBOM and provenance attestations, sign with Cosign keyless, and verify.

**Prerequisites.** Lab 4.

**Time.** 40 minutes.

### Steps

1. Replace the `build-image.yml` build job with the full version:

   ```yaml
   name: build-image
   on:
     workflow_call:
       inputs:
         context: { type: string, required: true }
         image:   { type: string, required: true }
         platforms: { type: string, default: "linux/amd64,linux/arm64" }
       outputs:
         digest: { value: "${{ jobs.build.outputs.digest }}" }
         tags:   { value: "${{ jobs.build.outputs.tags }}" }

   permissions: {}

   jobs:
     build:
       runs-on: ubuntu-24.04
       timeout-minutes: 30
       permissions:
         contents: read
         packages: write
         id-token: write
         attestations: write
       outputs:
         digest: ${{ steps.push.outputs.digest }}
         tags: ${{ steps.meta.outputs.tags }}
       steps:
         - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
           with: { persist-credentials: false }

         - uses: docker/setup-qemu-action@49b3bc8e6bdd4a60e6116a5414239cba5943d3cf # v3.2.0
         - uses: docker/setup-buildx-action@c47758b77c9736f4b2ef4073d4d51994fabfe349 # v3.7.1

         - uses: docker/login-action@9780b0c442fbb1117ed29e0efdff1e18412f7567 # v3.3.0
           with:
             registry: ghcr.io
             username: ${{ github.actor }}
             password: ${{ secrets.GITHUB_TOKEN }}

         - id: meta
           uses: docker/metadata-action@369eb591f429131d6889c46b94e711f089e6ca96 # v5.6.1
           with:
             images: ${{ inputs.image }}
             tags: |
               type=sha,format=long
               type=ref,event=branch
               type=semver,pattern={{version}}
               type=raw,value=latest,enable={{is_default_branch}}
             labels: |
               org.opencontainers.image.source=${{ github.server_url }}/${{ github.repository }}
               org.opencontainers.image.revision=${{ github.sha }}

         - id: push
           uses: docker/build-push-action@4f58ea79222b3b9dc2c8bbdd6debcef730109a75 # v6.9.0
           with:
             context: ${{ inputs.context }}
             platforms: ${{ inputs.platforms }}
             push: true
             tags: ${{ steps.meta.outputs.tags }}
             labels: ${{ steps.meta.outputs.labels }}
             provenance: mode=max
             sbom: true
             cache-from: type=gha,scope=${{ inputs.image }}
             cache-to: type=gha,scope=${{ inputs.image }},mode=max
             build-args: |
               VERSION=${{ steps.meta.outputs.version }}

         - name: GitHub artifact attestation (SLSA provenance)
           uses: actions/attest-build-provenance@ef244123eb79f2f7a7e75d99086184180e6d0018 # v1.4.4
           with:
             subject-name: ${{ inputs.image }}
             subject-digest: ${{ steps.push.outputs.digest }}
             push-to-registry: true

         - name: SBOM with Syft
           uses: anchore/sbom-action@fc46e51fd3cb168ffb36c6d1915723c47db58abb # v0.17.7
           with:
             image: ${{ inputs.image }}@${{ steps.push.outputs.digest }}
             format: cyclonedx-json
             output-file: sbom.cdx.json
             upload-artifact: true
             artifact-name: sbom-${{ github.job }}-${{ github.run_id }}.cdx.json

         - uses: sigstore/cosign-installer@dc72c7d5c4d10cd6bcb8cf6e3fd625a9e5e537da # v3.7.0

         - name: Cosign keyless sign + SBOM attestation
           env:
             IMAGE: ${{ inputs.image }}@${{ steps.push.outputs.digest }}
           run: |
             cosign sign --yes "$IMAGE"
             cosign attest --yes --type cyclonedx --predicate sbom.cdx.json "$IMAGE"

         - name: Verify what we just signed
           env:
             IMAGE: ${{ inputs.image }}@${{ steps.push.outputs.digest }}
             IDENTITY: https://github.com/${{ github.repository }}/.github/workflows/build-image.yml@${{ github.ref }}
           run: |
             cosign verify "$IMAGE" \
               --certificate-identity "$IDENTITY" \
               --certificate-oidc-issuer https://token.actions.githubusercontent.com \
               --output text
   ```

   Note the identity in the Fulcio certificate is the **reusable workflow's** `job_workflow_ref`, not the caller's. That is the identity Kyverno will verify in Lab 10.

2. Merge; inspect the result.

   ```bash
   DIGEST=$(gh run view --json jobs --jq '.jobs[] | select(.name=="image / build") | .steps[] | select(.name=="push") | .outputs.digest' 2>/dev/null || true)
   cosign tree ghcr.io/<org>/capstone-platform/fastapi-api:latest
   ```

   Expected output:

   ```text
   📦 Supply Chain Security Related artifacts for an image: ghcr.io/<org>/capstone-platform/fastapi-api:latest
   └── 💾 Attestations for an image tag: ghcr.io/<org>/capstone-platform/fastapi-api:sha256-....att
      └── 🍒 sha256:...
   └── 🔐 Signatures for an image tag: ghcr.io/<org>/capstone-platform/fastapi-api:sha256-....sig
      └── 🍒 sha256:...
   ```

   ```bash
   gh attestation verify oci://ghcr.io/<org>/capstone-platform/fastapi-api:latest --owner <org>
   ```

   ```text
   Loaded digest sha256:... for oci://ghcr.io/<org>/capstone-platform/fastapi-api:latest
   Loaded 1 attestation from GitHub API
   ✓ Verification succeeded!
   sha256:... was attested by:
   REPO                         PREDICATE_TYPE                  WORKFLOW
   <org>/capstone-platform      https://slsa.dev/provenance/v1  .github/workflows/build-image.yml@refs/heads/main
   ```

### Verify

Second build of an unchanged Dockerfile should show `CACHED` on most layers and finish in under a minute.

### Cleanup

Delete test tags via Packages UI or `gh api --method DELETE orgs/<ORG>/packages/container/<name>/versions/<id>`.

### What you learned

Buildx with the `gha` cache backend, buildkit's own provenance/SBOM attestations versus GitHub artifact attestations versus Cosign signatures (three overlapping mechanisms; the capstone uses GitHub attestations for `gh attestation verify` and Cosign for Kyverno admission), and how the OIDC identity of a reusable workflow becomes the signing identity.

---

## Lab 7: OIDC to AWS and Terraform Plan-on-PR / Apply-on-Merge

**Objective.** Create an IAM OIDC provider and two roles (plan: read-only; apply: scoped write), then a workflow that comments the plan on PRs and applies on merge to `main`.

**Prerequisites.** AWS account `123456789012`, an S3 bucket and DynamoDB table for Terraform state (or S3 native locking in Terraform 1.10+), Terraform code under `infra/terraform/envs/staging`.

**Time.** 60 minutes.

### Steps

1. Create the OIDC provider (once per account).

   ```bash
   aws iam create-open-id-connect-provider \
     --url https://token.actions.githubusercontent.com \
     --client-id-list sts.amazonaws.com \
     --thumbprint-list 6938fd4d98bab03faadb97b34396831e3780aea1
   ```

   (AWS now validates GitHub's certificate chain itself; the thumbprint is required by the API but no longer load-bearing.)

2. Trust policy for the **plan** role, `trust-plan.json`. Any PR or push in the repository may assume it:

   ```json
   {
     "Version": "2012-10-17",
     "Statement": [
       {
         "Effect": "Allow",
         "Principal": {
           "Federated": "arn:aws:iam::123456789012:oidc-provider/token.actions.githubusercontent.com"
         },
         "Action": "sts:AssumeRoleWithWebIdentity",
         "Condition": {
           "StringEquals": {
             "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
           },
           "StringLike": {
             "token.actions.githubusercontent.com:sub": "repo:<ORG>/capstone-platform:*"
           }
         }
       }
     ]
   }
   ```

3. Trust policy for the **apply** role, `trust-apply.json`. Only the `staging` environment, which (via deployment branch policy) only runs from `main`:

   ```json
   {
     "Version": "2012-10-17",
     "Statement": [
       {
         "Effect": "Allow",
         "Principal": {
           "Federated": "arn:aws:iam::123456789012:oidc-provider/token.actions.githubusercontent.com"
         },
         "Action": "sts:AssumeRoleWithWebIdentity",
         "Condition": {
           "StringEquals": {
             "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
             "token.actions.githubusercontent.com:sub": "repo:<ORG>/capstone-platform:environment:staging",
             "token.actions.githubusercontent.com:repository_id": "<REPOSITORY_ID>"
           }
         }
       }
     ]
   }
   ```

   `repository_id` protects against repository rename/recreate attacks (a deleted repo name can be re-registered by someone else; the numeric id cannot).

4. Create the roles and attach policies:

   ```bash
   aws iam create-role --role-name gha-capstone-plan  --assume-role-policy-document file://trust-plan.json  --max-session-duration 3600
   aws iam create-role --role-name gha-capstone-apply --assume-role-policy-document file://trust-apply.json --max-session-duration 3600

   aws iam attach-role-policy --role-name gha-capstone-plan  --policy-arn arn:aws:iam::aws:policy/ReadOnlyAccess
   # State bucket access for both; scoped write policy for apply (write your own least-privilege policy per stack)
   aws iam put-role-policy --role-name gha-capstone-plan  --policy-name tfstate --policy-document file://tfstate-policy.json
   aws iam put-role-policy --role-name gha-capstone-apply --policy-name tfstate --policy-document file://tfstate-policy.json
   aws iam attach-role-policy --role-name gha-capstone-apply --policy-arn arn:aws:iam::123456789012:policy/capstone-staging-apply
   ```

   `tfstate-policy.json`:

   ```json
   {
     "Version": "2012-10-17",
     "Statement": [
       { "Effect": "Allow", "Action": ["s3:ListBucket"], "Resource": "arn:aws:s3:::<STATE_BUCKET>" },
       { "Effect": "Allow", "Action": ["s3:GetObject", "s3:PutObject", "s3:DeleteObject"],
         "Resource": "arn:aws:s3:::<STATE_BUCKET>/capstone/*" },
       { "Effect": "Allow", "Action": ["dynamodb:GetItem", "dynamodb:PutItem", "dynamodb:DeleteItem"],
         "Resource": "arn:aws:dynamodb:<REGION>:123456789012:table/<LOCK_TABLE>" }
     ]
   }
   ```

5. Store non-secret identifiers as variables:

   ```bash
   gh variable set AWS_REGION --body eu-west-1
   gh variable set AWS_PLAN_ROLE_ARN --body arn:aws:iam::123456789012:role/gha-capstone-plan
   gh variable set AWS_APPLY_ROLE_ARN --env staging --body arn:aws:iam::123456789012:role/gha-capstone-apply
   ```

6. Create `.github/workflows/terraform.yml`:

   ```yaml
   name: terraform

   on:
     pull_request:
       paths: ['infra/terraform/**', '.github/workflows/terraform.yml']
     push:
       branches: [main]
       paths: ['infra/terraform/**', '.github/workflows/terraform.yml']

   permissions:
     contents: read

   concurrency:
     group: terraform-staging
     cancel-in-progress: false      # never cancel an apply

   env:
     TF_IN_AUTOMATION: "true"
     TF_INPUT: "false"
     WORKDIR: infra/terraform/envs/staging

   jobs:
     plan:
       runs-on: ubuntu-24.04
       timeout-minutes: 20
       permissions:
         contents: read
         id-token: write
         pull-requests: write
       outputs:
         exitcode: ${{ steps.plan.outputs.exitcode }}
       steps:
         - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
           with: { persist-credentials: false }

         - uses: aws-actions/configure-aws-credentials@e3dd6a429d7300a6a4c196c26e071d42e0343502 # v4.0.2
           with:
             role-to-assume: ${{ vars.AWS_PLAN_ROLE_ARN }}
             aws-region: ${{ vars.AWS_REGION }}
             role-session-name: gha-plan-${{ github.run_id }}

         - uses: hashicorp/setup-terraform@b9cd54a3c349d3f38e8881555d616ced269862dd # v3.1.2
           with:
             terraform_version: "1.9.8"
             terraform_wrapper: false

         - run: terraform fmt -check -recursive
           working-directory: infra/terraform

         - run: terraform init -lockfile=readonly
           working-directory: ${{ env.WORKDIR }}

         - run: terraform validate -no-color
           working-directory: ${{ env.WORKDIR }}

         - name: Checkov (IaC policy)
           uses: bridgecrewio/checkov-action@99bb2caf247dfd9f03cf984373bc6043d4e32ebf # v12.2930.0
           with:
             directory: ${{ env.WORKDIR }}
             framework: terraform
             soft_fail: false
             output_format: cli,sarif
             output_file_path: console,checkov.sarif

         - id: plan
           working-directory: ${{ env.WORKDIR }}
           run: |
             set +e
             terraform plan -no-color -detailed-exitcode -out=tfplan 2>&1 | tee plan.txt
             code=${PIPESTATUS[0]}
             set -e
             echo "exitcode=${code}" >> "$GITHUB_OUTPUT"
             # 0 = no changes, 2 = changes, 1 = error
             if [ "$code" -eq 1 ]; then exit 1; fi
             {
               echo "summary<<EOF"
               grep -E '^(Plan:|No changes)' plan.txt || echo "Plan produced no summary line"
               echo "EOF"
             } >> "$GITHUB_OUTPUT"

         - uses: actions/upload-artifact@b4b15b8c7c6ac21ea08fcf65892d2ee8f75cf882 # v4.4.3
           with:
             name: tfplan-${{ github.sha }}
             path: |
               ${{ env.WORKDIR }}/tfplan
               ${{ env.WORKDIR }}/plan.txt
             retention-days: 7

         - name: Comment plan on PR
           if: github.event_name == 'pull_request'
           uses: actions/github-script@60a0d83039c74a4aee543508d2ffcb1c3799cdea # v7.0.1
           env:
             PLAN: ${{ steps.plan.outputs.summary }}
             WORKDIR: ${{ env.WORKDIR }}
           with:
             script: |
               const fs = require('fs');
               const full = fs.readFileSync(`${process.env.WORKDIR}/plan.txt`, 'utf8');
               const trimmed = full.length > 60000 ? full.slice(-60000) : full;
               const body = [
                 `### Terraform plan (\`${process.env.WORKDIR}\`)`,
                 `**${process.env.PLAN}**`,
                 '<details><summary>Show plan</summary>', '', '```hcl', trimmed, '```', '</details>',
                 `*Run: ${context.serverUrl}/${context.repo.owner}/${context.repo.repo}/actions/runs/${context.runId}*`
               ].join('\n');
               const { data: comments } = await github.rest.issues.listComments({
                 ...context.repo, issue_number: context.issue.number });
               const prev = comments.find(c => c.user.type === 'Bot' && c.body.startsWith('### Terraform plan'));
               if (prev) await github.rest.issues.updateComment({ ...context.repo, comment_id: prev.id, body });
               else await github.rest.issues.createComment({ ...context.repo, issue_number: context.issue.number, body });

     apply:
       needs: plan
       if: github.event_name == 'push' && github.ref == 'refs/heads/main' && needs.plan.outputs.exitcode == '2'
       runs-on: ubuntu-24.04
       timeout-minutes: 30
       environment:
         name: staging
       permissions:
         contents: read
         id-token: write
       steps:
         - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
           with: { persist-credentials: false }

         - uses: aws-actions/configure-aws-credentials@e3dd6a429d7300a6a4c196c26e071d42e0343502 # v4.0.2
           with:
             role-to-assume: ${{ vars.AWS_APPLY_ROLE_ARN }}
             aws-region: ${{ vars.AWS_REGION }}
             role-session-name: gha-apply-${{ github.run_id }}

         - uses: hashicorp/setup-terraform@b9cd54a3c349d3f38e8881555d616ced269862dd # v3.1.2
           with: { terraform_version: "1.9.8", terraform_wrapper: false }

         - uses: actions/download-artifact@fa0a91b85d4f404e444e00e005971372dc801d16 # v4.1.8
           with:
             name: tfplan-${{ github.sha }}
             path: ${{ env.WORKDIR }}

         - run: terraform init -lockfile=readonly
           working-directory: ${{ env.WORKDIR }}

         - name: Apply the exact plan that was reviewed
           working-directory: ${{ env.WORKDIR }}
           run: terraform apply -no-color -auto-approve tfplan

         - run: terraform output -json > "$RUNNER_TEMP/outputs.json" && echo "Applied $(jq length "$RUNNER_TEMP/outputs.json") outputs" >> "$GITHUB_STEP_SUMMARY"
           working-directory: ${{ env.WORKDIR }}
   ```

7. Open a PR changing a tag on a resource; confirm the plan comment appears and updates on subsequent pushes. Merge; confirm `apply` runs under `staging` with the apply role.

### Verify

```bash
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=AssumeRoleWithWebIdentity --max-results 3 \
  --query 'Events[].CloudTrailEvent' --output text | jq -r '.userIdentity.userName, .requestParameters.roleArn'
```

You should see `gha-plan-<run_id>` with the plan role on the PR and `gha-apply-<run_id>` with the apply role on merge. Attempt to assume the apply role from a PR by temporarily pointing `AWS_PLAN_ROLE_ARN` at it: it must fail with `Not authorized to perform sts:AssumeRoleWithWebIdentity`.

### Cleanup

Keep the roles; they are the capstone's. Delete the plan artifacts if desired.

### What you learned

The `sub` claim is the policy surface; environment-scoped subjects bind IAM authorisation to GitHub approval; `-detailed-exitcode` turns "no changes" into a skip; applying the saved plan file guarantees you apply what was reviewed.

---

## Lab 8: Nightly `terraform destroy` Cost-Control Cron

**Objective.** Tear the staging lab environment down every night and rebuild on demand, saving most of the monthly bill.

**Prerequisites.** Lab 7.

**Time.** 20 minutes.

### Steps

1. Create `.github/workflows/nightly-destroy.yml`:

   ```yaml
   name: nightly-destroy

   on:
     schedule:
       - cron: '0 22 * * 1-5'       # 22:00 UTC weekdays (adjust to your evening)
     workflow_dispatch:
       inputs:
         confirm:
           description: Type DESTROY to confirm
           type: string
           required: true

   permissions:
     contents: read

   concurrency:
     group: terraform-staging       # same group as the apply workflow: destroy never races an apply
     cancel-in-progress: false

   jobs:
     destroy:
       if: github.event_name == 'schedule' || inputs.confirm == 'DESTROY'
       runs-on: ubuntu-24.04
       timeout-minutes: 40
       environment: staging
       permissions:
         contents: read
         id-token: write
         issues: write
       env:
         WORKDIR: infra/terraform/envs/staging
       steps:
         - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
           with: { persist-credentials: false }
         - uses: aws-actions/configure-aws-credentials@e3dd6a429d7300a6a4c196c26e071d42e0343502 # v4.0.2
           with:
             role-to-assume: ${{ vars.AWS_APPLY_ROLE_ARN }}
             aws-region: ${{ vars.AWS_REGION }}
             role-session-name: gha-destroy-${{ github.run_id }}
         - uses: hashicorp/setup-terraform@b9cd54a3c349d3f38e8881555d616ced269862dd # v3.1.2
           with: { terraform_version: "1.9.8", terraform_wrapper: false }
         - run: terraform init -lockfile=readonly
           working-directory: ${{ env.WORKDIR }}
         - name: Guard: refuse if protected resources exist in state
           working-directory: ${{ env.WORKDIR }}
           run: |
             if terraform state list | grep -Eq 'aws_rds_cluster\.prod|aws_s3_bucket\.audit'; then
               echo "::error::Protected resources present; refusing to destroy"; exit 1
             fi
         - name: Destroy
           working-directory: ${{ env.WORKDIR }}
           run: terraform destroy -no-color -auto-approve
         - name: Open an issue on failure
           if: failure()
           uses: actions/github-script@60a0d83039c74a4aee543508d2ffcb1c3799cdea # v7.0.1
           with:
             script: |
               await github.rest.issues.create({
                 ...context.repo,
                 title: `Nightly destroy failed (${new Date().toISOString().slice(0,10)})`,
                 body: `Run: ${context.serverUrl}/${context.repo.owner}/${context.repo.repo}/actions/runs/${context.runId}\n\nCheck for orphaned resources: \`aws resourcegroupstaggingapi get-resources --tag-filters Key=Project,Values=capstone\``,
                 labels: ['cost', 'ops']
               });
   ```

2. Add a matching `workflow_dispatch`-only `rebuild` job to `terraform.yml` (or rely on the next merge to `main`). Schedule a morning `apply` if you want it back automatically:

   ```yaml
   on:
     schedule:
       - cron: '0 7 * * 1-5'
   ```

   wired to the apply job with `if: github.event_name == 'schedule'`.

### Verify

Next morning: `gh run list --workflow nightly-destroy --limit 1` shows `success`, and `aws ec2 describe-instances --filters Name=tag:Project,Values=capstone --query 'Reservations[].Instances[].State.Name'` returns only `terminated`.

### What you learned

Shared `concurrency` groups serialise mutually exclusive operations; `schedule` runs only on the default branch and may drift by several minutes under load (never rely on it for second-level timing); a guard step plus automated issue on failure is the minimum for a destructive cron.

---

## Lab 9: Deploy via Argo CD Image Bump

**Objective.** After an image is built and signed, update the GitOps repository's Kustomize overlay so Argo CD rolls it out to staging, then promote to production behind the environment gate.

**Prerequisites.** Lab 6; a GitOps repo `<ORG>/capstone-gitops` with `apps/fastapi-api/overlays/{staging,production}/kustomization.yaml`; Argo CD watching it. A GitHub App with `contents: write` on the GitOps repo (its App ID in `vars.GITOPS_APP_ID`, private key in secret `GITOPS_APP_PRIVATE_KEY`).

**Time.** 40 minutes.

### Steps

1. Add to `ci-fastapi.yml`:

   ```yaml
     deploy-staging:
       needs: image
       if: github.event_name == 'push' && github.ref == 'refs/heads/main'
       runs-on: ubuntu-24.04
       environment: { name: staging, url: https://staging.example.com }
       permissions: { contents: read }
       steps:
         - name: Mint GitOps repo token
           id: app
           uses: actions/create-github-app-token@5d869da34e18e7287c1daad50e0b8ea0f506ce69 # v1.11.0
           with:
             app-id: ${{ vars.GITOPS_APP_ID }}
             private-key: ${{ secrets.GITOPS_APP_PRIVATE_KEY }}
             owner: ${{ github.repository_owner }}
             repositories: capstone-gitops
         - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
           with:
             repository: ${{ github.repository_owner }}/capstone-gitops
             token: ${{ steps.app.outputs.token }}
             persist-credentials: true
         - name: Bump image digest
           env:
             DIGEST: ${{ needs.image.outputs.digest }}
             IMAGE: ghcr.io/${{ github.repository }}/fastapi-api
           run: |
             cd apps/fastapi-api/overlays/staging
             yq -i '(.images[] | select(.name == strenv(IMAGE)) | .digest) = strenv(DIGEST)' kustomization.yaml
             git config user.name "capstone-gitops[bot]"
             git config user.email "capstone-gitops[bot]@users.noreply.github.com"
             git commit -am "staging: fastapi-api ${DIGEST:7:12} from ${GITHUB_SHA::7}" || echo "no change"
             git push
         - name: Wait for Argo CD sync
           env:
             ARGOCD_SERVER: argocd.example.com
             ARGOCD_AUTH_TOKEN: ${{ secrets.ARGOCD_READONLY_TOKEN }}
           run: |
             curl -fsSL -o argocd https://github.com/argoproj/argo-cd/releases/download/v2.13.0/argocd-linux-amd64 && chmod +x argocd
             ./argocd app wait fastapi-api-staging --sync --health --timeout 600 --grpc-web

     deploy-production:
       needs: deploy-staging
       runs-on: ubuntu-24.04
       environment: { name: production, url: https://api.example.com }
       permissions: { contents: read }
       steps:
         # identical to staging with overlays/production
         - run: echo "promote ${{ needs.image.outputs.digest }}"
   ```

   The `ARGOCD_READONLY_TOKEN` is a read-only Argo CD project token used only to *wait*; the pipeline never has write access to the cluster. Argo CD does the deploying. If you must `kubectl` directly (for example a one-off Job), use OIDC to assume a role mapped by EKS access entries with a namespaced RBAC role, never a kubeconfig secret.

2. Merge and watch the Deployments tab.

### Verify

`argocd app get fastapi-api-staging` shows `Sync Status: Synced`, `Health Status: Healthy`, and the image digest matches `needs.image.outputs.digest`.

### What you learned

The pipeline's write access is to a Git repository, not a cluster (GitOps boundary); GitHub App tokens are the correct way to write to another repository (short-lived, scoped, and unlike `GITHUB_TOKEN` they *do* trigger downstream workflows); environment gates sit between staging and production.

---

## Lab 10: The Seven-Gate DevSecOps Pipeline

**Objective.** Compose everything into `devsecops.yml`: Gitleaks → SonarQube/Semgrep → build → Syft/Trivy/Cosign → Checkov/Kyverno → deploy staging → OWASP ZAP → Falco runtime check → DefectDojo import. Every scanner uploads SARIF to code scanning and JSON to DefectDojo.

**Prerequisites.** Labs 1-9; a SonarQube instance (`vars.SONAR_HOST_URL`, secret `SONAR_TOKEN`); a DefectDojo instance (`vars.DEFECTDOJO_URL`, secret `DEFECTDOJO_API_KEY`, `vars.DEFECTDOJO_ENGAGEMENT_ID`); Kyverno installed in the staging cluster with a `verifyImages` policy; Falco with Falcosidekick exporting to a webhook or Loki.

**Time.** 2 hours.

### Steps

1. Create `.github/workflows/devsecops.yml`.

   ```yaml
   name: devsecops

   on:
     push:
       branches: [main]
     pull_request:
       branches: [main]
     workflow_dispatch:

   permissions:
     contents: read

   concurrency:
     group: devsecops-${{ github.event.pull_request.number || github.ref }}
     cancel-in-progress: ${{ github.event_name == 'pull_request' }}

   env:
     IMAGE: ghcr.io/${{ github.repository }}/fastapi-api
     SERVICE_DIR: services/fastapi-api

   jobs:
     # ---------------------------------------------------------------- Gate 1
     gate1-secrets:
       name: "Gate 1: Gitleaks"
       runs-on: ubuntu-24.04
       timeout-minutes: 10
       permissions:
         contents: read
         security-events: write
       steps:
         - uses: step-security/harden-runner@0080882f6c36860b6ba35c610c98ce87d4e2f26f # v2.10.2
           with: { egress-policy: audit }
         - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
           with: { fetch-depth: 0, persist-credentials: false }
         - name: Gitleaks
           run: |
             docker run --rm -v "$PWD:/repo" -w /repo ghcr.io/gitleaks/gitleaks:v8.21.2 \
               detect --source . --redact --report-format sarif --report-path gitleaks.sarif --exit-code 1 \
               --log-opts="${{ github.event_name == 'pull_request' && format('{0}..{1}', github.event.pull_request.base.sha, github.event.pull_request.head.sha) || '' }}"
         - uses: github/codeql-action/upload-sarif@ea9e4e37992a54ee68a9622e985e60c8e8f12d9f # v3.27.4
           if: always()
           with: { sarif_file: gitleaks.sarif, category: gitleaks }
         - uses: actions/upload-artifact@b4b15b8c7c6ac21ea08fcf65892d2ee8f75cf882 # v4.4.3
           if: always()
           with: { name: report-gitleaks, path: gitleaks.sarif }

     # ---------------------------------------------------------------- Gate 2
     gate2-sast:
       name: "Gate 2: SonarQube + Semgrep"
       needs: gate1-secrets
       runs-on: ubuntu-24.04
       timeout-minutes: 20
       permissions:
         contents: read
         security-events: write
         pull-requests: read
       steps:
         - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
           with: { fetch-depth: 0, persist-credentials: false }
         - uses: ./.github/actions/setup-python-uv
         - name: Tests with coverage (SonarQube needs the XML)
           working-directory: ${{ env.SERVICE_DIR }}
           run: uv sync --frozen && uv run pytest -q --cov=app --cov-report=xml:coverage.xml --junitxml=junit.xml
         - name: Semgrep
           run: |
             docker run --rm -v "$PWD:/src" -w /src semgrep/semgrep:1.96.0 \
               semgrep scan --config p/ci --config p/python --config p/owasp-top-ten \
               --config .semgrep/ --sarif --output semgrep.sarif --error --metrics=off
         - uses: github/codeql-action/upload-sarif@ea9e4e37992a54ee68a9622e985e60c8e8f12d9f # v3.27.4
           if: always()
           with: { sarif_file: semgrep.sarif, category: semgrep }
         - name: SonarQube scan
           uses: SonarSource/sonarqube-scan-action@bfd4e558cda28cda6b5defafb9232d191be8c203 # v4.2.1
           env:
             SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
             SONAR_HOST_URL: ${{ vars.SONAR_HOST_URL }}
           with:
             projectBaseDir: ${{ env.SERVICE_DIR }}
             args: >
               -Dsonar.projectKey=capstone-fastapi-api
               -Dsonar.python.coverage.reportPaths=coverage.xml
               -Dsonar.python.xunit.reportPath=junit.xml
               -Dsonar.qualitygate.wait=true
               -Dsonar.qualitygate.timeout=300
         - uses: actions/upload-artifact@b4b15b8c7c6ac21ea08fcf65892d2ee8f75cf882 # v4.4.3
           if: always()
           with: { name: report-sast, path: "semgrep.sarif" }

     # ---------------------------------------------------------------- Build
     build:
       name: Build image
       needs: gate2-sast
       uses: ./.github/workflows/build-image.yml
       permissions:
         contents: read
         packages: write
         id-token: write
         attestations: write
       with:
         context: services/fastapi-api
         image: ghcr.io/${{ github.repository }}/fastapi-api
         platforms: linux/amd64

     # ---------------------------------------------------------------- Gate 3/4
     gate3-supply-chain:
       name: "Gate 3/4: Syft + Trivy + Cosign verify"
       needs: build
       runs-on: ubuntu-24.04
       timeout-minutes: 20
       permissions:
         contents: read
         packages: read
         security-events: write
       steps:
         - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
           with: { persist-credentials: false }
         - uses: docker/login-action@9780b0c442fbb1117ed29e0efdff1e18412f7567 # v3.3.0
           with: { registry: ghcr.io, username: "${{ github.actor }}", password: "${{ secrets.GITHUB_TOKEN }}" }
         - name: Syft SBOM (CycloneDX)
           uses: anchore/sbom-action@fc46e51fd3cb168ffb36c6d1915723c47db58abb # v0.17.7
           with:
             image: ${{ env.IMAGE }}@${{ needs.build.outputs.digest }}
             format: cyclonedx-json
             output-file: sbom.cdx.json
             upload-artifact: false
         - name: Trivy image scan (fail on fixable CRITICAL/HIGH)
           uses: aquasecurity/trivy-action@18f2510ee396bbf400402947b394f2dd8c87dbb0 # v0.29.0
           with:
             image-ref: ${{ env.IMAGE }}@${{ needs.build.outputs.digest }}
             format: sarif
             output: trivy-image.sarif
             severity: CRITICAL,HIGH
             ignore-unfixed: true
             exit-code: '1'
             vuln-type: os,library
         - name: Trivy SBOM scan to JSON (for DefectDojo)
           if: always()
           uses: aquasecurity/trivy-action@18f2510ee396bbf400402947b394f2dd8c87dbb0 # v0.29.0
           with:
             scan-type: sbom
             scan-ref: sbom.cdx.json
             format: json
             output: trivy-image.json
             exit-code: '0'
         - uses: github/codeql-action/upload-sarif@ea9e4e37992a54ee68a9622e985e60c8e8f12d9f # v3.27.4
           if: always()
           with: { sarif_file: trivy-image.sarif, category: trivy-image }
         - uses: sigstore/cosign-installer@dc72c7d5c4d10cd6bcb8cf6e3fd625a9e5e537da # v3.7.0
         - name: Verify signature and SBOM attestation
           env:
             REF: ${{ env.IMAGE }}@${{ needs.build.outputs.digest }}
           run: |
             cosign verify "$REF" \
               --certificate-identity-regexp "^https://github.com/${{ github.repository }}/\.github/workflows/build-image\.yml@" \
               --certificate-oidc-issuer https://token.actions.githubusercontent.com > /dev/null
             cosign verify-attestation "$REF" --type cyclonedx \
               --certificate-identity-regexp "^https://github.com/${{ github.repository }}/\.github/workflows/build-image\.yml@" \
               --certificate-oidc-issuer https://token.actions.githubusercontent.com > /dev/null
             echo "Signature and CycloneDX attestation verified for $REF" >> "$GITHUB_STEP_SUMMARY"
         - uses: actions/upload-artifact@b4b15b8c7c6ac21ea08fcf65892d2ee8f75cf882 # v4.4.3
           if: always()
           with:
             name: report-supply-chain
             path: |
               sbom.cdx.json
               trivy-image.sarif
               trivy-image.json

     # ---------------------------------------------------------------- Gate 5
     gate5-policy:
       name: "Gate 5: Checkov + Kyverno"
       needs: build
       runs-on: ubuntu-24.04
       timeout-minutes: 15
       permissions:
         contents: read
         security-events: write
       steps:
         - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
           with: { persist-credentials: false }
         - name: Checkov on Terraform, Helm, Kubernetes, Dockerfile
           uses: bridgecrewio/checkov-action@99bb2caf247dfd9f03cf984373bc6043d4e32ebf # v12.2930.0
           with:
             directory: .
             framework: terraform,kubernetes,helm,dockerfile,github_actions
             output_format: cli,sarif,json
             output_file_path: console,checkov.sarif,checkov.json
             soft_fail: false
             skip_check: CKV_GHA_7            # example: allow workflow_dispatch inputs in this repo
         - uses: github/codeql-action/upload-sarif@ea9e4e37992a54ee68a9622e985e60c8e8f12d9f # v3.27.4
           if: always()
           with: { sarif_file: checkov.sarif, category: checkov }
         - name: Render manifests and test against Kyverno policies offline
           run: |
             curl -fsSL https://github.com/kyverno/kyverno/releases/download/v1.13.1/kyverno-cli_v1.13.1_linux_x86_64.tar.gz \
               | tar xz kyverno && chmod +x kyverno
             kubectl kustomize deploy/overlays/staging > rendered.yaml
             ./kyverno apply policies/kyverno/ --resource rendered.yaml --audit-warn \
               --policy-report --output kyverno-report.yaml
             if grep -q 'result: fail' kyverno-report.yaml; then
               echo "::error::Kyverno policy violations"; grep -B2 -A6 'result: fail' kyverno-report.yaml; exit 1
             fi
         - uses: actions/upload-artifact@b4b15b8c7c6ac21ea08fcf65892d2ee8f75cf882 # v4.4.3
           if: always()
           with: { name: report-policy, path: "checkov.json\nkyverno-report.yaml" }

     # ---------------------------------------------------------------- Deploy staging
     deploy-staging:
       name: Deploy staging (Argo CD)
       needs: [gate3-supply-chain, gate5-policy]
       if: github.event_name == 'push' && github.ref == 'refs/heads/main'
       runs-on: ubuntu-24.04
       environment: { name: staging, url: https://staging.example.com }
       permissions: { contents: read }
       steps:
         - uses: actions/create-github-app-token@5d869da34e18e7287c1daad50e0b8ea0f506ce69 # v1.11.0
           id: app
           with:
             app-id: ${{ vars.GITOPS_APP_ID }}
             private-key: ${{ secrets.GITOPS_APP_PRIVATE_KEY }}
             owner: ${{ github.repository_owner }}
             repositories: capstone-gitops
         - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
           with:
             repository: ${{ github.repository_owner }}/capstone-gitops
             token: ${{ steps.app.outputs.token }}
         - env:
             DIGEST: ${{ needs.build.outputs.digest }}
             IMAGE: ${{ env.IMAGE }}
           run: |
             cd apps/fastapi-api/overlays/staging
             yq -i '(.images[] | select(.name == strenv(IMAGE)) | .digest) = strenv(DIGEST)' kustomization.yaml
             git config user.name "capstone-gitops[bot]"; git config user.email "capstone-gitops[bot]@users.noreply.github.com"
             git commit -am "staging: fastapi-api ${DIGEST:7:12}" || echo "no change"; git push
         - name: Wait for Argo CD (Kyverno admission is enforced here)
           env:
             ARGOCD_SERVER: argocd.example.com
             ARGOCD_AUTH_TOKEN: ${{ secrets.ARGOCD_READONLY_TOKEN }}
           run: |
             curl -fsSL -o argocd https://github.com/argoproj/argo-cd/releases/download/v2.13.0/argocd-linux-amd64 && chmod +x argocd
             ./argocd app wait fastapi-api-staging --sync --health --timeout 600 --grpc-web

     # ---------------------------------------------------------------- Gate 6
     gate6-dast:
       name: "Gate 6: OWASP ZAP"
       needs: deploy-staging
       runs-on: ubuntu-24.04
       timeout-minutes: 30
       permissions:
         contents: read
         issues: write
       steps:
         - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
           with: { persist-credentials: false }
         - name: ZAP API scan against staging OpenAPI
           uses: zaproxy/action-api-scan@3f9ce6db3e0a4f6a4c2b6a3a1a0f3c5c9a8a1b2c # v0.9.0
           with:
             target: https://staging.example.com/openapi.json
             format: openapi
             rules_file_name: .zap/rules.tsv
             cmd_options: "-a -J zap.json -r zap.html -z '-config api.disablekey=true'"
             fail_action: true
             allow_issue_writing: false
         - uses: actions/upload-artifact@b4b15b8c7c6ac21ea08fcf65892d2ee8f75cf882 # v4.4.3
           if: always()
           with: { name: report-dast, path: "zap.json\nzap.html" }

     # ---------------------------------------------------------------- Gate 7
     gate7-runtime:
       name: "Gate 7: Falco runtime check"
       needs: gate6-dast
       runs-on: ubuntu-24.04
       timeout-minutes: 10
       permissions: { contents: read }
       steps:
         - name: Query Falco events for the new pods during the DAST window
           env:
             LOKI_URL: ${{ vars.LOKI_URL }}
             LOKI_TOKEN: ${{ secrets.LOKI_READ_TOKEN }}
           run: |
             end=$(date +%s)000000000; start=$(( $(date +%s) - 1800 ))000000000
             q='{job="falco"} | json | priority=~"Critical|Error|Warning" | k8s_ns_name="fastapi-staging"'
             curl -fsS -H "Authorization: Bearer $LOKI_TOKEN" \
               --data-urlencode "query=$q" --data-urlencode "start=$start" --data-urlencode "end=$end" \
               "$LOKI_URL/loki/api/v1/query_range" > falco.json
             count=$(jq '[.data.result[].values[]] | length' falco.json)
             echo "Falco alerts in window: $count" >> "$GITHUB_STEP_SUMMARY"
             if [ "$count" -gt 0 ]; then
               jq -r '.data.result[].values[][1]' falco.json | head -20
               echo "::error::Falco raised $count runtime alerts for the new release"; exit 1
             fi
         - uses: actions/upload-artifact@b4b15b8c7c6ac21ea08fcf65892d2ee8f75cf882 # v4.4.3
           if: always()
           with: { name: report-runtime, path: falco.json }

     # ---------------------------------------------------------------- ASPM
     defectdojo:
       name: Import findings to DefectDojo
       needs: [gate1-secrets, gate2-sast, gate3-supply-chain, gate5-policy, gate6-dast, gate7-runtime]
       if: always() && !cancelled()
       runs-on: ubuntu-24.04
       timeout-minutes: 10
       permissions: { contents: read }
       steps:
         - uses: actions/download-artifact@fa0a91b85d4f404e444e00e005971372dc801d16 # v4.1.8
           with: { pattern: "report-*", merge-multiple: true, path: reports }
         - name: Reimport each scan
           env:
             DD_URL: ${{ vars.DEFECTDOJO_URL }}
             DD_KEY: ${{ secrets.DEFECTDOJO_API_KEY }}
             DD_ENGAGEMENT: ${{ vars.DEFECTDOJO_ENGAGEMENT_ID }}
             VERSION: ${{ github.sha }}
           run: |
             import() {  # $1 = scan_type, $2 = file
               [ -f "reports/$2" ] || { echo "skip $1 ($2 missing)"; return 0; }
               curl -fsS -X POST "$DD_URL/api/v2/reimport-scan/" \
                 -H "Authorization: Token $DD_KEY" \
                 -F "engagement=$DD_ENGAGEMENT" -F "scan_type=$1" -F "file=@reports/$2" \
                 -F "version=$VERSION" -F "auto_create_context=true" -F "close_old_findings=true" \
                 -F "minimum_severity=Low" > /dev/null && echo "imported $1"
             }
             import "Gitleaks Scan"      gitleaks.sarif
             import "Semgrep JSON Report" semgrep.sarif
             import "Trivy Scan"          trivy-image.json
             import "CycloneDX Scan"      sbom.cdx.json
             import "Checkov Scan"        checkov.json
             import "ZAP Scan"            zap.json
         - name: Gate summary
           if: always()
           env:
             R1: ${{ needs.gate1-secrets.result }}
             R2: ${{ needs.gate2-sast.result }}
             R3: ${{ needs.gate3-supply-chain.result }}
             R5: ${{ needs.gate5-policy.result }}
             R6: ${{ needs.gate6-dast.result }}
             R7: ${{ needs.gate7-runtime.result }}
           run: |
             {
               echo "## DevSecOps gates"
               echo "| Gate | Result |"; echo "|---|---|"
               echo "| 1 Gitleaks | $R1 |"; echo "| 2 SonarQube/Semgrep | $R2 |"
               echo "| 3/4 Syft/Trivy/Cosign | $R3 |"; echo "| 5 Checkov/Kyverno | $R5 |"
               echo "| 6 OWASP ZAP | $R6 |"; echo "| 7 Falco | $R7 |"
             } >> "$GITHUB_STEP_SUMMARY"
   ```

2. Kyverno policy enforced in the staging cluster so the admission gate is real, `policies/kyverno/verify-image.yaml`:

   ```yaml
   apiVersion: kyverno.io/v1
   kind: ClusterPolicy
   metadata:
     name: verify-capstone-images
   spec:
     validationFailureAction: Enforce
     webhookTimeoutSeconds: 30
     rules:
       - name: keyless-github-actions
         match:
           any:
             - resources:
                 kinds: [Pod]
                 namespaces: [fastapi-staging, fastapi-production]
         verifyImages:
           - imageReferences: ["ghcr.io/<org>/capstone-platform/*"]
             mutateDigest: true
             required: true
             attestors:
               - entries:
                   - keyless:
                       subjectRegExp: "^https://github.com/<org>/capstone-platform/\\.github/workflows/build-image\\.yml@refs/heads/main$"
                       issuer: https://token.actions.githubusercontent.com
                       rekor: { url: https://rekor.sigstore.dev }
             attestations:
               - type: https://cyclonedx.org/bom
                 attestors:
                   - entries:
                       - keyless:
                           subjectRegExp: "^https://github.com/<org>/capstone-platform/"
                           issuer: https://token.actions.githubusercontent.com
   ```

3. Push to a branch, open a PR: gates 1, 2, build, 3/4 and 5 run; deploy and later gates are skipped. Merge: everything runs. Watch the graph in the Actions UI.

### Verify

- Security tab → Code scanning shows categories `gitleaks`, `semgrep`, `trivy-image`, `checkov`.
- DefectDojo engagement shows six tests with the commit SHA as version.
- Deliberately add `AKIAIOSFODNN7EXAMPLE` (the AWS documentation example key) to a file in a PR: Gate 1 fails, nothing downstream runs, SARIF annotation appears on the PR diff.
- Deliberately change `build-image.yml` identity in the Kyverno policy to a wrong path: `argocd app wait` fails with pods rejected by admission (`image verification failed`).

### Cleanup

Revert the deliberate failures. Reduce ZAP to `action-baseline` on PRs if 30-minute DAST runs are too slow for your budget.

### What you learned

How seven independent tools compose into one DAG with a single trust story: the identity that signed the image is the identity Kyverno admits; SARIF gives developers findings in the PR; DefectDojo gives security a deduplicated view; every gate is a separate job so a failing gate has an obvious owner.

---

## Where Next

[05-production-patterns.md](05-production-patterns.md) hardens what you built: egress control, OIDC trust policies for Azure and GCP, ARC at scale, cost, release engineering and the anti-pattern list.
