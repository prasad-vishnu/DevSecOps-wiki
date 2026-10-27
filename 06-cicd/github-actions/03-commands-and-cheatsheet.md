# GitHub Actions: Commands and Cheatsheet

| | |
|---|---|
| **Cluster** | 06 CI/CD Pipelines |
| **Role** | Anchor |
| **Prerequisites** | [02-core-concepts.md](02-core-concepts.md) |
| **Target version** | GitHub.com (2025+), `gh` CLI 2.60+, `actions/*@v4` |
| **Capstone phase** | Phase 1 (Months 1-4) |

> An exhaustive, task-organised reference: the full workflow YAML schema with one-line explanations, every context and function, runner environment variables and files, workflow commands, the `gh` CLI for workflows/runs/secrets/variables/caches/runners, the REST API calls you actually use, and SHA-pinned references for the actions the capstone depends on. Keep this open while writing workflows.

---

## 1. Workflow File Skeleton

```yaml
name: ci                                   # display name; defaults to file path
run-name: CI for ${{ github.ref_name }} by @${{ github.actor }}   # per-run title (expressions allowed)

on:                                        # triggers (section 2)
  push: { branches: [main] }

permissions:                               # default GITHUB_TOKEN scopes for every job
  contents: read

env:                                       # workflow-level env (available to all jobs/steps)
  REGISTRY: ghcr.io

defaults:
  run:
    shell: bash
    working-directory: .

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  <job_id>:                                # a-z, 0-9, -, _; must start with letter or _
    name: Human name
    runs-on: ubuntu-24.04                  # label, array of labels, or { group:, labels: }
    needs: [other_job]
    if: ${{ github.event_name != 'pull_request' }}
    permissions: { contents: read }
    environment: { name: staging, url: https://staging.example.com }
    concurrency: { group: deploy-staging, cancel-in-progress: false }
    outputs: { digest: "${{ steps.push.outputs.digest }}" }
    env: { KEY: value }
    defaults: { run: { shell: bash } }
    timeout-minutes: 30
    continue-on-error: false
    container: { image: "image:tag", credentials: {username:, password:}, env: {}, ports: [], volumes: [], options: "" }
    services: { <id>: { image:, env:, ports:, volumes:, options:, credentials: } }
    strategy:
      fail-fast: true
      max-parallel: 4
      matrix: { key: [values], include: [], exclude: [] }
    steps:
      - name: Step
        id: step_id
        if: ${{ success() }}
        uses: owner/repo@sha            # or ./local/path or docker://image:tag
        with: { input: value }
        run: command                     # mutually exclusive with uses
        shell: bash
        working-directory: dir
        env: { KEY: value }
        continue-on-error: false
        timeout-minutes: 10
  <reusable_caller>:
    uses: owner/repo/.github/workflows/file.yml@sha
    with: { input: value }
    secrets: inherit                     # or explicit mapping
    permissions: { contents: read }
    needs: [job]
    if: ...
    strategy: { matrix: {} }
```

---

## 2. Triggers (`on:`)

| Trigger | Key options | One-liner |
|---|---|---|
| `push` | `branches`, `branches-ignore`, `tags`, `tags-ignore`, `paths`, `paths-ignore` | commit pushed; `GITHUB_SHA` = pushed commit |
| `pull_request` | `types`, `branches`, `paths` | PR from same repo or fork; read-only token for forks; merge commit checked out |
| `pull_request_target` | same | runs base-branch workflow with write token; never check out PR code with secrets present |
| `workflow_dispatch` | `inputs.<name>.{type: string/boolean/choice/number/environment, default, required, options, description}` | manual trigger; max 25 inputs, 65 KB payload |
| `schedule` | `cron` (UTC, 5 fields) | default branch only; min interval 5 min; may be delayed under load |
| `workflow_call` | `inputs`, `secrets`, `outputs` | makes the workflow reusable |
| `workflow_run` | `workflows: [names]`, `types: [completed, requested, in_progress]`, `branches` | runs after another workflow; default branch workflow, full token |
| `release` | `types: [published, created, released, prereleased, edited, deleted]` | `published` is the one you want |
| `repository_dispatch` | `types: [custom-event]` | `POST /repos/{o}/{r}/dispatches` with `client_payload` |
| `issue_comment` | `types: [created]` | ChatOps; filter with `if: github.event.issue.pull_request && startsWith(github.event.comment.body, '/deploy')` |
| `issues`, `label`, `discussion`, `check_run`, `check_suite`, `status`, `deployment`, `deployment_status`, `page_build`, `registry_package`, `merge_group`, `create`, `delete`, `fork`, `watch`, `public`, `milestone`, `project*` | | the long tail |

Glob syntax in filters: `*` (no `/`), `**` (any), `?`, `+`, `[abc]`, `!` negation (must follow a positive pattern). Filters match `refs/heads/<branch>` short names.

Cron examples:

| Expression | Meaning |
|---|---|
| `0 2 * * *` | 02:00 UTC daily |
| `30 1 * * 1-5` | 01:30 UTC weekdays |
| `0 */6 * * *` | every 6 hours |
| `0 0 1 * *` | first of month |

---

## 3. Contexts Quick Reference

| Expression | Example value |
|---|---|
| `github.repository` | `octo-org/capstone-platform` |
| `github.repository_owner` | `octo-org` |
| `github.repository_id` | `123456789` |
| `github.ref` | `refs/heads/main`, `refs/pull/42/merge`, `refs/tags/v1.2.0` |
| `github.ref_name` | `main`, `42/merge`, `v1.2.0` |
| `github.ref_type` | `branch` or `tag` |
| `github.sha` | commit SHA (merge commit on PRs) |
| `github.head_ref` | PR source branch (PR events only; attacker-controlled name) |
| `github.base_ref` | PR target branch |
| `github.event_name` | `push`, `pull_request`, ... |
| `github.event.pull_request.number` | `42` |
| `github.event.pull_request.head.sha` | PR head commit (not the merge commit) |
| `github.event.inputs.<name>` / `inputs.<name>` | dispatch inputs (`inputs` is typed; `event.inputs` is always string) |
| `github.actor` | user who triggered |
| `github.triggering_actor` | user who re-ran (may differ from actor) |
| `github.run_id` / `run_number` / `run_attempt` | `12345678901` / `87` / `1` |
| `github.workflow` / `github.workflow_ref` | `ci` / `octo-org/repo/.github/workflows/ci.yml@refs/heads/main` |
| `github.job` | current job id |
| `github.server_url` / `api_url` / `graphql_url` | `https://github.com` ... |
| `github.workspace` | `/home/runner/work/repo/repo` |
| `github.token` | same as `secrets.GITHUB_TOKEN` |
| `runner.os` / `runner.arch` | `Linux` / `X64` |
| `runner.temp` / `runner.tool_cache` | scratch, tool cache |
| `runner.debug` | `1` when debug logging enabled |
| `job.status` | `success`, `failure`, `cancelled` |
| `job.services.<id>.ports['5432']` | mapped host port |
| `steps.<id>.outputs.<name>` / `outcome` / `conclusion` | |
| `needs.<job>.outputs.<name>` / `result` | `success`, `failure`, `cancelled`, `skipped` |
| `matrix.<key>` / `strategy.job-index` / `strategy.job-total` | |
| `vars.<NAME>` / `secrets.<NAME>` | |

Default environment variables (available in `run:` without `${{ }}`): `CI=true`, `GITHUB_ACTIONS`, `GITHUB_ACTOR`, `GITHUB_REPOSITORY`, `GITHUB_REF`, `GITHUB_REF_NAME`, `GITHUB_SHA`, `GITHUB_RUN_ID`, `GITHUB_RUN_NUMBER`, `GITHUB_RUN_ATTEMPT`, `GITHUB_JOB`, `GITHUB_EVENT_NAME`, `GITHUB_EVENT_PATH` (JSON file of the payload), `GITHUB_WORKSPACE`, `GITHUB_SERVER_URL`, `GITHUB_API_URL`, `GITHUB_HEAD_REF`, `GITHUB_BASE_REF`, `GITHUB_ACTION`, `GITHUB_ACTION_PATH`, `GITHUB_ACTION_REPOSITORY`, `RUNNER_OS`, `RUNNER_ARCH`, `RUNNER_NAME`, `RUNNER_TEMP`, `RUNNER_TOOL_CACHE`, `RUNNER_DEBUG`.

Read the raw payload: `jq . "$GITHUB_EVENT_PATH"`.

---

## 4. Expression Functions

| Function | Signature | Example |
|---|---|---|
| `contains` | `(search, item)` | `contains(github.event.head_commit.message, '[skip ci]')` |
| `startsWith` / `endsWith` | `(string, prefix)` | `startsWith(github.ref, 'refs/tags/')` |
| `format` | `(fmt, v0, v1...)` | `format('{0}-{1}', runner.os, matrix.python)` (escape literal braces as `{{`) |
| `join` | `(array, sep=',')` | `join(github.event.pull_request.labels.*.name, ', ')` |
| `toJSON` | `(value)` | `run: echo '${{ toJSON(github.event) }}'` (use env instead for safety) |
| `fromJSON` | `(string)` | `fromJSON(vars.MATRIX)`; `fromJSON('true')` → boolean |
| `hashFiles` | `(glob, ...)` | `hashFiles('**/package-lock.json', '**/yarn.lock')` (relative to `GITHUB_WORKSPACE`) |
| `success` / `failure` / `cancelled` / `always` | `()` | `if: always()` |

Idioms:

```yaml
# Ternary
VERSION: ${{ github.ref_type == 'tag' && github.ref_name || github.sha }}

# Default a possibly-empty input
TAG: ${{ inputs.image_tag || 'latest' }}

# Run only on the default branch of the canonical repo (not forks)
if: github.repository == 'octo-org/capstone-platform' && github.ref == format('refs/heads/{0}', github.event.repository.default_branch)

# Label-driven conditional
if: contains(github.event.pull_request.labels.*.name, 'run-e2e')

# Check a previous job's soft failure
if: needs.scan.result == 'failure' || needs.scan.result == 'success'
```

---

## 5. Runner Files and Workflow Commands

| Mechanism | Syntax | Effect |
|---|---|---|
| Output | `echo "name=value" >> "$GITHUB_OUTPUT"` | `steps.<id>.outputs.name` |
| Multi-line output | `{ echo "name<<EOF"; cat file; echo "EOF"; } >> "$GITHUB_OUTPUT"` | |
| Env for later steps | `echo "NAME=value" >> "$GITHUB_ENV"` | |
| PATH | `echo "/opt/tool/bin" >> "$GITHUB_PATH"` | prepends |
| Summary | `echo "# Title" >> "$GITHUB_STEP_SUMMARY"` | Markdown, 1 MiB per step |
| Mask | `echo "::add-mask::$VALUE"` | redact value in logs |
| Group | `echo "::group::Title"` ... `echo "::endgroup::"` | collapsible log |
| Annotations | `echo "::error file=f,line=1,endLine=2,title=T::msg"` (`::warning`, `::notice`) | shows in PR Files tab (max 10 warnings + 10 errors per step) |
| Debug | `echo "::debug::msg"` | shown only with `ACTIONS_STEP_DEBUG` |
| Stop commands | `echo "::stop-commands::$TOKEN"` ... `echo "::$TOKEN::"` | disable processing of untrusted output |
| Echo commands | `echo "::echo::on"` | print commands in log |

PowerShell equivalents write with `Out-File -FilePath $env:GITHUB_OUTPUT -Append -Encoding utf8`.

---

## 6. `gh` CLI by Task

### 6.1 Workflows and runs

```bash
gh workflow list [--all]                          # list workflows (incl. disabled)
gh workflow view ci [--yaml]                      # show workflow, or its YAML
gh workflow run ci --ref main -f env=staging -f dry_run=true   # dispatch with inputs
gh workflow run deploy.yml --json <<< '{"environment":"production"}'
gh workflow enable ci / gh workflow disable ci

gh run list --workflow ci --branch main --limit 10 --json databaseId,status,conclusion,headSha
gh run list --event pull_request --status failure
gh run view 12345678901 [--log] [--log-failed] [--job 98765432101] [--web]
gh run watch 12345678901 --exit-status             # block until done; exit code = conclusion
gh run rerun 12345678901 [--failed] [--debug]      # rerun all / only failed jobs / with debug logging
gh run cancel 12345678901
gh run download 12345678901 --name sbom-fastapi --dir ./out
gh run download 12345678901 --pattern 'sbom-*'
gh run delete 12345678901
```

### 6.2 Secrets and variables

```bash
gh secret set SONAR_TOKEN --body "$(cat token.txt)"            # repo secret (prefer --body from a file; avoid shell history)
gh secret set SONAR_TOKEN < token.txt
gh secret set DB_PASSWORD --env production                     # environment secret
gh secret set NPM_TOKEN --org octo-org --visibility selected --repos capstone-platform,other
gh secret set GHCR_PULL --app dependabot                       # Dependabot secret
gh secret list [--env production] [--org octo-org]
gh secret delete SONAR_TOKEN

gh variable set AWS_REGION --body eu-west-1
gh variable set AZURE_CLIENT_ID --env production --body 00000000-0000-0000-0000-000000000000
gh variable list
```

### 6.3 Caches

```bash
gh cache list [--limit 50] [--sort size_in_bytes] [--order desc]
gh cache delete <key|id>
gh cache delete --all
gh api repos/{owner}/{repo}/actions/cache/usage
```

### 6.4 Runners

```bash
gh api repos/{owner}/{repo}/actions/runners --jq '.runners[] | {name, status, busy, labels: [.labels[].name]}'
gh api --method POST repos/{owner}/{repo}/actions/runners/registration-token --jq .token
gh api --method POST repos/{owner}/{repo}/actions/runners/remove-token --jq .token
gh api --method DELETE repos/{owner}/{repo}/actions/runners/<RUNNER_ID>
gh api orgs/{org}/actions/runner-groups
```

### 6.5 Environments and deployments

```bash
gh api --method PUT repos/{owner}/{repo}/environments/production --input - <<'EOF'
{
  "wait_timer": 5,
  "prevent_self_review": true,
  "reviewers": [{ "type": "Team", "id": 1234567 }],
  "deployment_branch_policy": { "protected_branches": false, "custom_branch_policies": true }
}
EOF
gh api --method POST repos/{owner}/{repo}/environments/production/deployment-branch-policies \
  -f name=main -f type=branch
gh api repos/{owner}/{repo}/deployments --jq '.[] | {environment, ref, created_at}'
gh api repos/{owner}/{repo}/actions/runs/<RUN_ID>/pending_deployments
gh api --method POST repos/{owner}/{repo}/actions/runs/<RUN_ID>/pending_deployments \
  -F 'environment_ids[]=<ENV_ID>' -f state=approved -f comment="LGTM"
```

### 6.6 OIDC subject customisation

```bash
# Use job_workflow_ref in sub so trust policies can pin to a reusable workflow
gh api --method PUT repos/{owner}/{repo}/actions/oidc/customization/sub \
  -F use_default=false -f 'include_claim_keys[]=repo' -f 'include_claim_keys[]=context' -f 'include_claim_keys[]=job_workflow_ref'
gh api repos/{owner}/{repo}/actions/oidc/customization/sub
```

### 6.7 Releases and attestations

```bash
gh release create v1.4.0 --generate-notes --verify-tag dist/*.whl
gh release create v1.5.0-rc.1 --prerelease --notes-file CHANGELOG.md
gh release upload v1.4.0 sbom.cdx.json --clobber
gh attestation verify oci://ghcr.io/octo-org/fastapi-api:v1.4.0 --owner octo-org
gh attestation verify dist/app-1.4.0-py3-none-any.whl --repo octo-org/capstone-platform
```

### 6.8 Billing

```bash
gh api orgs/{org}/settings/billing/actions
gh api users/{username}/settings/billing/actions
```

```json
{ "total_minutes_used": 1240, "total_paid_minutes_used": 0, "included_minutes": 3000,
  "minutes_used_breakdown": { "UBUNTU": 980, "MACOS": 20, "WINDOWS": 240 } }
```

---

## 7. Actions the Capstone Uses (SHA-pinned)

Pin to the full commit SHA and keep the version tag in a comment so Dependabot can bump both. Verify SHAs yourself with `gh api repos/<owner>/<repo>/git/ref/tags/<tag>`; the ones below are illustrative of the convention and must be re-resolved when you adopt them.

| Purpose | Reference |
|---|---|
| Checkout | `actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2` |
| Python | `actions/setup-python@0b93645e9fea7318ecaed2b359559ac225c90a2b # v5.3.0` |
| uv | `astral-sh/setup-uv@f3bcaebff5eace81a1c062af9f9011aae482ca9d # v3.1.7` |
| Java | `actions/setup-java@8df1039502a15bceb9433410b1a100fbe190c53b # v4.5.0` |
| Gradle | `gradle/actions/setup-gradle@d156388eb19639ec20ade50009f3d199ce1e2808 # v4.1.0` |
| Node | `actions/setup-node@39370e3970a6d050c480ffad4ff0ed4d3fdee5af # v4.1.0` |
| Cache | `actions/cache@6849a6489940f00c2f30c0fb92c6274307ccb58a # v4.1.2` |
| Upload artifact | `actions/upload-artifact@b4b15b8c7c6ac21ea08fcf65892d2ee8f75cf882 # v4.4.3` |
| Download artifact | `actions/download-artifact@fa0a91b85d4f404e444e00e005971372dc801d16 # v4.1.8` |
| Buildx | `docker/setup-buildx-action@c47758b77c9736f4b2ef4073d4d51994fabfe349 # v3.7.1` |
| QEMU (multi-arch) | `docker/setup-qemu-action@49b3bc8e6bdd4a60e6116a5414239cba5943d3cf # v3.2.0` |
| Registry login | `docker/login-action@9780b0c442fbb1117ed29e0efdff1e18412f7567 # v3.3.0` |
| Image metadata | `docker/metadata-action@369eb591f429131d6889c46b94e711f089e6ca96 # v5.6.1` |
| Build and push | `docker/build-push-action@4f58ea79222b3b9dc2c8bbdd6debcef730109a75 # v6.9.0` |
| Cosign | `sigstore/cosign-installer@dc72c7d5c4d10cd6bcb8cf6e3fd625a9e5e537da # v3.7.0` |
| Attest provenance | `actions/attest-build-provenance@ef244123eb79f2f7a7e75d99086184180e6d0018 # v1.4.4` |
| Attest SBOM | `actions/attest-sbom@5026d3663fb6c3d8e9c2aa0fce3bb7e6e1e8a1b6 # v1.4.1` |
| Syft SBOM | `anchore/sbom-action@fc46e51fd3cb168ffb36c6d1915723c47db58abb # v0.17.7` |
| Trivy | `aquasecurity/trivy-action@18f2510ee396bbf400402947b394f2dd8c87dbb0 # v0.29.0` |
| Gitleaks | `gitleaks/gitleaks-action@83373cf2f8c4db6e24b41c1a9b086bb9619e9cd3 # v2.3.7` |
| Semgrep | `returntocorp/semgrep-action` is deprecated; run the `semgrep/semgrep` container image directly |
| SonarQube | `SonarSource/sonarqube-scan-action@bfd4e558cda28cda6b5defafb9232d191be8c203 # v4.2.1` |
| Checkov | `bridgecrewio/checkov-action@99bb2caf247dfd9f03cf984373bc6043d4e32ebf # v12.2930.0` |
| ZAP baseline | `zaproxy/action-baseline@7cbe4d5cc9f6fbf50bb9d9bea9de4ab2b8e1ddd1 # v0.14.0` |
| CodeQL | `github/codeql-action/init@…`, `github/codeql-action/analyze@…`, `github/codeql-action/upload-sarif@…` (pin the same SHA for all three) |
| Dependency review | `actions/dependency-review-action@3b139cfc5fae8b618d3eae3675e383bb1769c019 # v4.5.0` |
| Harden runner | `step-security/harden-runner@0080882f6c36860b6ba35c610c98ce87d4e2f26f # v2.10.2` |
| Scorecard | `ossf/scorecard-action@62b2cac7ed8198b15735ed49ab1e5cf35480ba46 # v2.4.0` |
| AWS creds | `aws-actions/configure-aws-credentials@e3dd6a429d7300a6a4c196c26e071d42e0343502 # v4.0.2` |
| ECR login | `aws-actions/amazon-ecr-login@062b18b96a7aff071d4dc91bc00c4c1a7945b076 # v2.0.1` |
| Azure login | `azure/login@a65d910e8af852a8061c627c456678983e180302 # v2.2.0` |
| GCP auth | `google-github-actions/auth@6fc4af4b145ae7821d527454aa9bd537d1f2dc5f # v2.1.7` |
| Terraform | `hashicorp/setup-terraform@b9cd54a3c349d3f38e8881555d616ced269862dd # v3.1.2` |
| Argo CD image bump | `mikefarah/yq@…` in a step, or the `argocd-image-updater` controller |
| Release please | `googleapis/release-please-action@7987652d64b4581673a76e33ad5e98e3dd56832f # v4.1.3` |
| Semantic release | `cycjimmy/semantic-release-action@b1b432f13acb7768e0c8efdec416d363a57546f2 # v4.1.1` |
| Paths filter | `dorny/paths-filter@de90cc6fb38fc0963ad72b210f1f284cd68cea36 # v3.0.2` |
| act (local) | `act -j <job>`; `act -P ubuntu-latest=catthehacker/ubuntu:act-latest` |

Dependabot config to keep them current (`.github/dependabot.yml`):

```yaml
version: 2
updates:
  - package-ecosystem: github-actions
    directory: /
    schedule: { interval: weekly }
    groups:
      actions: { patterns: ["*"] }
  - package-ecosystem: docker
    directory: /services/fastapi-api
    schedule: { interval: weekly }
  - package-ecosystem: uv
    directory: /services/fastapi-api
    schedule: { interval: weekly }
  - package-ecosystem: maven
    directory: /services/springboot-api
    schedule: { interval: weekly }
```

---

## 8. Common Snippets

### 8.1 Minimal secure job header

```yaml
permissions:
  contents: read
jobs:
  build:
    runs-on: ubuntu-24.04
    timeout-minutes: 20
    steps:
      - uses: step-security/harden-runner@0080882f6c36860b6ba35c610c98ce87d4e2f26f # v2.10.2
        with:
          egress-policy: audit          # switch to block + allowed-endpoints once the list is known
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
        with:
          persist-credentials: false   # do not leave the token in .git/config
```

### 8.2 Image tags and labels with metadata-action

```yaml
- id: meta
  uses: docker/metadata-action@369eb591f429131d6889c46b94e711f089e6ca96 # v5.6.1
  with:
    images: ghcr.io/${{ github.repository }}/fastapi-api
    tags: |
      type=sha,format=long
      type=ref,event=pr
      type=semver,pattern={{version}}
      type=semver,pattern={{major}}.{{minor}}
      type=raw,value=latest,enable={{is_default_branch}}
```

### 8.3 Cosign keyless sign and verify

```bash
cosign sign --yes "ghcr.io/octo-org/fastapi-api@${DIGEST}"
cosign attest --yes --predicate sbom.cdx.json --type cyclonedx "ghcr.io/octo-org/fastapi-api@${DIGEST}"

cosign verify "ghcr.io/octo-org/fastapi-api@${DIGEST}" \
  --certificate-identity-regexp '^https://github.com/octo-org/capstone-platform/\.github/workflows/.*@refs/heads/main$' \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com
```

### 8.4 SARIF upload

```yaml
- uses: github/codeql-action/upload-sarif@<SHA> # v3
  if: always()
  with:
    sarif_file: results.sarif
    category: trivy-image
```

Requires `permissions: security-events: write`. `category` keeps multiple tools' results separate in code scanning.

### 8.5 Post a PR comment with the token

```yaml
- uses: actions/github-script@60a0d83039c74a4aee543508d2ffcb1c3799cdea # v7.0.1
  with:
    script: |
      const body = process.env.BODY;
      await github.rest.issues.createComment({
        owner: context.repo.owner, repo: context.repo.repo,
        issue_number: context.issue.number, body
      });
  env:
    BODY: ${{ steps.plan.outputs.summary }}
```

Requires `permissions: pull-requests: write`.

### 8.6 Skip CI

Commit messages containing `[skip ci]`, `[ci skip]`, `[no ci]`, `[skip actions]`, or `[actions skip]` suppress `push` and `pull_request` workflows. Required checks will then be missing; prefer `paths-ignore` for predictable behaviour.

### 8.7 Enable debug logging

```bash
gh variable set ACTIONS_STEP_DEBUG --body true     # step-level debug lines
gh variable set ACTIONS_RUNNER_DEBUG --body true   # runner diagnostic logs in the run archive
gh run rerun <RUN_ID> --debug                      # one-off
```

Continue to [04-hands-on-labs.md](04-hands-on-labs.md).
