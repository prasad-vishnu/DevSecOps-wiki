# Gitleaks: Hands-On Labs

| | |
|---|---|
| **Cluster** | 08 DevSecOps |
| **Role** | Gate 1 anchor |
| **Prerequisites** | [01-installation-and-setup.md](01-installation-and-setup.md), [02-core-concepts.md](02-core-concepts.md) |
| **Target version** | Gitleaks 8.18+ |
| **Capstone phase** | Phase 5 (Months 17-20) |

> Six progressive labs: detect a seeded fake secret, allowlist and fingerprint it, write and test a custom rule, baseline a legacy repository, install the pre-commit hook, and build the CI gate with SARIF upload. Every secret in these labs is a clearly fake, non-functional string.

All labs use a scratch repository at `~/labs/gl-lab`. Commits use a throwaway identity so your real name is not in the fixture history.

```bash
mkdir -p ~/labs/gl-lab && cd ~/labs/gl-lab && git init -q -b main
git config user.name lab && git config user.email lab@example.com
```

---

## Lab 1: Seed and Detect

**Objective.** Commit three fake secrets of different shapes and observe which rules fire and why.

**Prerequisites.** Gitleaks installed.

**Time.** 15 minutes.

### Steps

1. Create three files with fake secrets of different structural precision.

   ```bash
   cat > settings.py <<'EOF'
   # All values below are FAKE and used only to exercise scanners.
   AWS_ACCESS_KEY_ID = "AKIAEXAMPLEEXAMPLE00"
   GITHUB_TOKEN = "ghp_EXAMPLE000000000000000000000000000000"
   DATABASE_PASSWORD = "changeme"
   EOF
   cat > deploy.sh <<'EOF'
   #!/usr/bin/env bash
   # FAKE Slack bot token for scanner testing
   export SLACK_BOT_TOKEN="xoxb-000000000000-000000000000-EXAMPLEEXAMPLEEXAMPLEEXA"
   EOF
   cat > app.yaml <<'EOF'
   api_key: "f4k3f4k3f4k3f4k3f4k3f4k3f4k3f4k3"   # FAKE, 32 hex-ish chars
   EOF
   git add . && git commit -qm "lab1: seed fake secrets"
   ```

2. Scan.

   ```bash
   gitleaks git . --verbose --no-banner
   ```

   Expected output (abridged):

   ```text
   Finding:     AWS_ACCESS_KEY_ID = "AKIAEXAMPLEEXAMPLE00"
   RuleID:      aws-access-token
   File:        settings.py
   Line:        2

   Finding:     GITHUB_TOKEN = "ghp_EXAMPLE000000000000000000000000000000"
   RuleID:      github-pat
   File:        settings.py
   Line:        3

   Finding:     export SLACK_BOT_TOKEN="xoxb-000000000000-000000000000-EXAMPLEEXAMPLEEXAMPLEEXA"
   RuleID:      slack-bot-token
   File:        deploy.sh
   Line:        3

   Finding:     api_key: "f4k3f4k3f4k3f4k3f4k3f4k3f4k3f4k3"
   RuleID:      generic-api-key
   File:        app.yaml
   Line:        1

   INF 1 commits scanned.
   WRN leaks found: 4
   ```

   Your exact count may differ by one: `generic-api-key` requires entropy >= 3.5 on the value and `f4k3` repeated has entropy about 2.0, so on most versions it does **not** fire, giving 3 findings. `DATABASE_PASSWORD = "changeme"` never fires: low entropy and `changeme` is a stopword.

3. Replace the repeated string with a high-entropy fake and re-scan.

   ```bash
   sed -i.bak 's/f4k3f4k3f4k3f4k3f4k3f4k3f4k3f4k3/9c1e4b7a2d8f6e3b5a0c7d9e1f2a4b6c/' app.yaml && rm app.yaml.bak
   git commit -qam "lab1: higher entropy fake"
   gitleaks git . --verbose --no-banner 2>&1 | grep -A1 generic-api-key
   ```

   Now `generic-api-key` fires on `app.yaml` (entropy about 3.9).

### Verify

`echo $?` after the scan prints `1`. `gitleaks git . --report-format json --report-path lab1.json; jq length lab1.json` prints the finding count.

### Cleanup

Keep the repository; later labs build on it.

### What you learned

Precise-prefix rules fire on structure alone; the generic rule needs entropy; stopwords drop obvious placeholders. The low-entropy `changeme` password is a reminder that Gitleaks is not a substitute for not committing config at all.

---

## Lab 2: Allowlist, Inline Allow and Fingerprint Ignore

**Objective.** Suppress the lab fixtures three different ways and understand the trade-off of each.

**Prerequisites.** Lab 1.

**Time.** 20 minutes.

### Steps

1. Add a repository config that allowlists the two documented fakes by regex and the `fixtures/` path.

   ```bash
   cat > .gitleaks.toml <<'EOF'
   title = "gl-lab"
   [extend]
   useDefault = true

   [allowlist]
   description = "Documented fake credentials used in labs (SEC-101)"
   regexes = [
     '''AKIAEXAMPLEEXAMPLE00''',
     '''ghp_EXAMPLE0{32}''',
   ]
   paths = ['''(^|/)fixtures/''']
   EOF
   gitleaks git . --no-banner 2>&1 | tail -1
   ```

   Expected: `WRN leaks found: 2` (Slack token and the generic key remain).

2. Suppress the Slack token with an inline comment.

   ```bash
   sed -i.bak 's|EXAMPLEEXA"|EXAMPLEEXA"  # gitleaks:allow|' deploy.sh && rm deploy.sh.bak
   git commit -qam "lab2: inline allow"
   gitleaks git . --no-banner 2>&1 | tail -1
   ```

   Expected: still `2`. Why? Full-history mode scans the *old* commit where the line had no comment. Inline allows only help going forward. Confirm with a range scan:

   ```bash
   gitleaks git . --log-opts="HEAD~1..HEAD" --no-banner 2>&1 | tail -1
   ```

   Expected: `no leaks found` for that range (the only added line is allowlisted).

3. Suppress the historical findings precisely with fingerprints.

   ```bash
   gitleaks git . --report-format json --report-path lab2.json --no-banner || true
   {
     echo "# Historical lab fixtures; see SEC-101. Secrets are fake and were never valid."
     jq -r '.[].Fingerprint' lab2.json
   } > .gitleaksignore
   cat .gitleaksignore
   gitleaks git . --no-banner 2>&1 | tail -1
   ```

   Expected: `no leaks found`, exit `0`.

### Verify

`git add .gitleaks.toml .gitleaksignore && git commit -qm "lab2: suppressions"` then `gitleaks git .` exits `0`.

### Cleanup

None.

### What you learned

Regex allowlists suppress a *pattern* everywhere (good for documented fakes); inline allows only affect future commits; fingerprints suppress one historical finding exactly and break if history is rewritten. All three are reviewable in Git, which is the point.

---

## Lab 3: Write and Test a Custom Rule for an Internal Token

**Objective.** Detect the capstone's internal service token format `cap_<8 lowercase alnum>_<40 base64url>` and prove the rule has no false positives on the repository.

**Prerequisites.** Lab 2.

**Time.** 25 minutes.

### Steps

1. Generate a realistic fake token and a decoy that should not match.

   ```bash
   FAKE_TOKEN="cap_svc0auth_$(openssl rand -base64 30 | tr '+/' '-_' | cut -c1-40)"
   echo "$FAKE_TOKEN"
   cat > services/auth/client.py 2>/dev/null || mkdir -p services/auth && cat > services/auth/client.py <<EOF
   # FAKE internal token, generated locally for the lab
   SERVICE_TOKEN = "${FAKE_TOKEN}"
   EOF
   cat > docs/tokens.md 2>/dev/null || mkdir -p docs && cat > docs/tokens.md <<'EOF'
   Internal tokens look like `cap_<service>_<random>`; e.g. `cap_example_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx`.
   EOF
   git add . && git commit -qm "lab3: internal token fixture and docs"
   ```

2. Add the rule with a keyword prefilter, `secretGroup`, entropy, and a rule-scoped allowlist for documentation.

   ```bash
   cat >> .gitleaks.toml <<'EOF'

   [[rules]]
   id = "capstone-internal-service-token"
   description = "Capstone internal service-to-service token (cap_<svc>_<40 base64url>)"
   regex = '''\b(cap_[a-z0-9]{8}_[A-Za-z0-9_-]{40})\b'''
   secretGroup = 1
   entropy = 4.0
   keywords = ["cap_"]
   tags = ["internal", "capstone"]
     [rules.allowlist]
     paths = ['''(^|/)docs/''']
     regexes = ['''cap_example_''']
   EOF
   gitleaks git . --verbose --no-banner 2>&1 | grep -B2 -A6 capstone-internal
   ```

   Expected output:

   ```text
   Finding:     SERVICE_TOKEN = "cap_svc0auth_..."
   Secret:      cap_svc0auth_...
   RuleID:      capstone-internal-service-token
   Entropy:     4.9...
   File:        services/auth/client.py
   Line:        2
   ```

   The `docs/tokens.md` placeholder does not fire: it is under `docs/`, contains `cap_example_`, and `x` repeated has entropy 0.

3. Build a tiny rule test harness so the rule is regression-tested in CI.

   ```bash
   mkdir -p .gitleaks-tests
   cat > .gitleaks-tests/positive.txt <<EOF
   token = "${FAKE_TOKEN}"
   EOF
   cat > .gitleaks-tests/negative.txt <<'EOF'
   token = "cap_example_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
   captain = "cap_tooshort_abc"
   EOF
   cat > .gitleaks-tests/run.sh <<'EOF'
   #!/usr/bin/env bash
   set -euo pipefail
   cd "$(dirname "$0")/.."
   # Positive fixture MUST produce a finding
   if gitleaks dir .gitleaks-tests/positive.txt --config .gitleaks.toml --no-banner --exit-code 7 >/dev/null 2>&1; then
     echo "FAIL: positive fixture not detected"; exit 1
   fi
   # Negative fixture MUST be clean
   gitleaks dir .gitleaks-tests/negative.txt --config .gitleaks.toml --no-banner >/dev/null
   echo "rule tests passed"
   EOF
   chmod +x .gitleaks-tests/run.sh && .gitleaks-tests/run.sh
   ```

   Expected: `rule tests passed`.

### Verify

`gitleaks dir .gitleaks-tests/positive.txt --config .gitleaks.toml --verbose` shows the rule ID; the negative file yields `no leaks found`.

### Cleanup

Commit the rule and tests: `git add . && git commit -qm "lab3: custom rule + tests"`.

### What you learned

A good rule anchors on structure (`cap_`, fixed-length segments, character classes), sets `secretGroup` so entropy and redaction apply to the right thing, uses `keywords` for speed, and ships with positive and negative fixtures.

---

## Lab 4: Baseline a Legacy Repository

**Objective.** Onboard a repository with historical leaks so that only *new* leaks fail CI, while tracking the backlog.

**Prerequisites.** Lab 3.

**Time.** 15 minutes.

### Steps

1. Remove the `.gitleaksignore` from Lab 2 to simulate a repository with an existing backlog.

   ```bash
   git rm -q .gitleaksignore && git commit -qm "lab4: simulate legacy backlog"
   gitleaks git . --no-banner 2>&1 | tail -1      # leaks found again
   ```

2. Generate and commit the baseline after triage (in real life: triage means every entry is rotated or confirmed fake).

   ```bash
   gitleaks git . --report-format json --report-path .gitleaks-baseline.json --redact --no-banner || true
   jq 'length' .gitleaks-baseline.json
   git add .gitleaks-baseline.json && git commit -qm "lab4: gitleaks baseline (all entries triaged, SEC-101)"
   gitleaks git . --baseline-path .gitleaks-baseline.json --no-banner 2>&1 | tail -1
   ```

   Expected: `no leaks found`.

3. Introduce a new leak and prove the baseline does not hide it.

   ```bash
   echo 'STRIPE_KEY="sk_test_EXAMPLE0000000000000000000"  # FAKE' > billing.env
   git add billing.env && git commit -qm "lab4: new leak"
   gitleaks git . --baseline-path .gitleaks-baseline.json --verbose --no-banner 2>&1 | grep RuleID
   ```

   Expected: `RuleID: stripe-access-token` and exit `1`.

4. Burn down the backlog report.

   ```bash
   jq -r '.[] | "\(.RuleID)\t\(.File):\(.StartLine)\t\(.Date)"' .gitleaks-baseline.json | column -t
   ```

### Verify

The baselined run exits `0` before step 3 and `1` after.

### Cleanup

`git rm -q billing.env && git commit -qm "lab4: remove leak"` (and note: in a real incident the key is already compromised and must be rotated; removal is not remediation).

### What you learned

Baselines make the gate adoptable on brownfield repositories without hiding new leaks. Redact the baseline. Treat each entry as a rotation task.

---

## Lab 5: Pre-commit Hook

**Objective.** Block a secret at commit time with the pre-commit framework and observe the bypass.

**Prerequisites.** `pre-commit` installed (`pipx install pre-commit`).

**Time.** 15 minutes.

### Steps

1. Configure and install.

   ```bash
   cat > .pre-commit-config.yaml <<'EOF'
   repos:
     - repo: https://github.com/gitleaks/gitleaks
       rev: v8.21.2
       hooks:
         - id: gitleaks
   EOF
   pre-commit install
   git add .pre-commit-config.yaml && git commit -qm "lab5: pre-commit"
   ```

   The commit itself runs the hook (expected: `Passed`).

2. Attempt to commit a fake key.

   ```bash
   echo 'export AWS_SECRET_ACCESS_KEY="EXAMPLEexampleEXAMPLE0000example0000EXAMP"  # FAKE' > creds.sh
   git add creds.sh && git commit -m "lab5: should be blocked"; echo "exit=$?"
   ```

   Expected: `Detect hardcoded secrets....Failed`, exit `1`, and `git log -1 --oneline` still shows the lab5 pre-commit commit (nothing was committed).

3. Observe the bypass and why the PR gate matters.

   ```bash
   git commit --no-verify -qm "lab5: bypassed hook"
   gitleaks git . --log-opts="HEAD~1..HEAD" --no-banner 2>&1 | tail -1
   ```

   Expected: `leaks found: 1`. The hook was skipped, the server-side range scan still catches it.

### Verify

`pre-commit run gitleaks --all-files` prints `Failed` while `creds.sh` is present.

### Cleanup

`git reset -q --hard HEAD~1` to drop the bypassed commit (safe here because it was never pushed).

### What you learned

The hook is a developer convenience that stops 95% of leaks at zero cost; the PR gate is the control.

---

## Lab 6: CI Gate with SARIF Upload

**Objective.** Run Gitleaks on every PR, fail on new leaks, and surface findings in the GitHub Security tab.

**Prerequisites.** A GitHub repository (push `~/labs/gl-lab` to a private repo you own). Code scanning requires GitHub Advanced Security on private repos in organisations; on personal public repos it is free.

**Time.** 30 minutes.

### Steps

1. Add the workflow using the Docker image (no licence key needed) and SARIF upload.

   ```bash
   mkdir -p .github/workflows
   cat > .github/workflows/gate-1-gitleaks.yml <<'EOF'
   name: gate-1-gitleaks
   on:
     pull_request:
     push:
       branches: [main]
     schedule:
       - cron: "17 2 * * *"
   permissions:
     contents: read
     security-events: write
   jobs:
     gitleaks:
       runs-on: ubuntu-latest
       container:
         image: ghcr.io/gitleaks/gitleaks:v8.21.2
         options: --user 0            # container checkout needs root for actions/checkout file ownership
       steps:
         - uses: actions/checkout@v4
           with: { fetch-depth: 0 }
         - name: Mark workspace safe for git
           run: git config --global --add safe.directory "$GITHUB_WORKSPACE"
         - name: Rule regression tests
           run: ./.gitleaks-tests/run.sh
         - name: Compute range
           id: range
           run: |
             if [ "${{ github.event_name }}" = "pull_request" ]; then
               git fetch origin "${{ github.base_ref }}" --depth=0 || true
               MB=$(git merge-base "origin/${{ github.base_ref }}" HEAD)
               echo "opts=${MB}..HEAD" >> "$GITHUB_OUTPUT"
             else
               echo "opts=" >> "$GITHUB_OUTPUT"
             fi
         - name: Gitleaks
           run: |
             gitleaks git . \
               --config .gitleaks.toml \
               --baseline-path .gitleaks-baseline.json \
               --log-opts="${{ steps.range.outputs.opts }}" \
               --redact --no-banner \
               --report-format sarif --report-path results.sarif
         - uses: github/codeql-action/upload-sarif@v3
           if: always()
           with:
             sarif_file: results.sarif
             category: gitleaks
   EOF
   git add .github && git commit -qm "lab6: CI gate" && git push -u origin main
   ```

2. Open a PR that leaks a fake key.

   ```bash
   git checkout -qb leak-test
   echo 'TOKEN="glpat-EXAMPLE00000000000000"  # FAKE GitLab PAT' > ci.env
   git add ci.env && git commit -qm "test: fake leak" --no-verify && git push -u origin leak-test
   gh pr create --fill
   ```

3. Observe: the `gate-1-gitleaks` check fails; the PR's **Security > Code scanning** tab shows a `gitlab-pat` alert on `ci.env:1` under category `gitleaks`.

### Verify

`gh run list --workflow gate-1-gitleaks --limit 1` shows `failure`; `gh api repos/{owner}/{repo}/code-scanning/alerts --jq '.[].rule.id'` lists `gitlab-pat`.

### Cleanup

Close the PR, delete the branch, and delete the private repository when done.

### What you learned

Range scanning plus a baseline gives a gate that fails only on new leaks; SARIF puts findings where developers already look; rule tests keep custom rules honest.

---

## Stretch Labs

- **Scan a container image filesystem.** `docker create --name tmp ghcr.io/example/api:latest && docker export tmp | tar -xC /tmp/rootfs && gitleaks dir /tmp/rootfs`. Compare with `trufflehog docker --image`.
- **Scan Terraform state.** `terraform show -json | gitleaks stdin --redact`. Discuss why state should be encrypted and never committed.
- **Custom SARIF severity.** Use `--report-format template` with a Go template that sets `security-severity` per rule so GitHub buckets internal tokens as Critical.

Next: [05-production-patterns.md](05-production-patterns.md).
