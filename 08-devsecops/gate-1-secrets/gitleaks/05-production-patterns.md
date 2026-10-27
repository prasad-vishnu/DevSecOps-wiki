# Gitleaks: Production Patterns

| | |
|---|---|
| **Cluster** | 08 DevSecOps |
| **Role** | Gate 1 anchor |
| **Prerequisites** | [04-hands-on-labs.md](04-hands-on-labs.md) |
| **Target version** | Gitleaks 8.18+ |
| **Capstone phase** | Phase 5 (Months 17-20) |

> How to run Gitleaks across an organisation: shared configuration, the three-tier scan strategy, CI integration on GitHub Actions / Jenkins / Azure DevOps, SARIF and DefectDojo publishing, false-positive governance, performance on monorepos, the remediation workflow when a secret leaks, anti-patterns and a decision checklist.

---

## 1. Three-Tier Scan Strategy

| Tier | Where | Scope | Blocking | Cadence |
|---|---|---|---|---|
| 1 Developer | pre-commit hook | Staged diff | Yes (bypassable) | Every commit |
| 2 Gate | PR workflow | Merge-base..HEAD range + rule tests | Yes (authoritative) | Every PR push |
| 3 Audit | Scheduled workflow / security team runner | Full history, all refs, all repos | No (alerts) | Nightly / weekly |

Tier 2 is the control; tiers 1 and 3 are convenience and assurance. Tier 3 is also where you catch leaks introduced by force-pushes, rebases and direct pushes to protected branches by admins.

```mermaid
flowchart LR
    subgraph T1["Tier 1: laptop"]
        H["pre-commit: gitleaks git --pre-commit --staged"]
    end
    subgraph T2["Tier 2: PR gate"]
        R["rule tests"] --> S["gitleaks git --log-opts=merge-base..HEAD --baseline"]
        S --> U["SARIF -> code scanning"]
    end
    subgraph T3["Tier 3: audit"]
        N["nightly: full history, every repo"] --> D["DefectDojo reimport"]
        N --> T["trufflehog verification"]
    end
    H --> R
    U --> D
```

---

## 2. Shared Organisation Config

Keep one canonical ruleset and let repositories extend it.

```text
org-security-config/            # a dedicated repo, protected branch, CODEOWNERS = security team
└── gitleaks/
    ├── base.gitleaks.toml      # useDefault + internal token rules + global allowlists
    ├── tests/                  # positive/negative fixtures per custom rule
    └── CHANGELOG.md
```

Repository-level `.gitleaks.toml`:

```toml
title = "service-x"
[extend]
path = "./.security/base.gitleaks.toml"   # vendored copy (git subtree / Renovate-managed) of the org base
[allowlist]
paths = ['''(^|/)tests/fixtures/''']
```

Distribution options, in order of preference:

1. **Renovate/Dependabot-managed vendored copy** of `base.gitleaks.toml` in each repo (`.security/`). Deterministic, offline, reviewable.
2. **Download at CI time** from the config repo at a pinned tag (`curl -sSL https://raw.githubusercontent.com/<ORG>/org-security-config/v3.2.0/gitleaks/base.gitleaks.toml`). Simple, but CI depends on network and the hook does not benefit.
3. **Custom Docker image** `ghcr.io/<ORG>/gitleaks:<ver>` with the config baked at `/etc/gitleaks/base.toml` and `GITLEAKS_CONFIG` set. Best for Jenkins and self-hosted runners.

Internal token formats worth a rule in any organisation: internal API gateway keys, service mesh tokens, Vault wrapping tokens (`hvs.`/`hvb.` prefixes are in the default ruleset already), internal JWT issuer (`"iss":"https://auth.example.internal"` in a decoded payload), database connection strings with embedded passwords (`postgres://user:pass@host`), Kubernetes service account tokens (`eyJhbGciOiJSUzI1NiIsImtpZCI6` + `kubernetes.io/serviceaccount`).

---

## 3. CI Integration Patterns

### GitHub Actions: Docker-based, no licence

See Lab 6 for the full workflow. Key production additions:

```yaml
    - name: Gitleaks
      run: |
        gitleaks git . --config .gitleaks.toml --baseline-path .gitleaks-baseline.json \
          --log-opts="${{ steps.range.outputs.opts }}" --redact --no-banner \
          --report-format sarif --report-path results.sarif \
          --max-target-megabytes 10
      # Do NOT use continue-on-error here; secrets block.
    - uses: github/codeql-action/upload-sarif@v3
      if: always()
      with: { sarif_file: results.sarif, category: gitleaks }
    - uses: actions/upload-artifact@v4
      if: always()
      with: { name: gitleaks-sarif, path: results.sarif, retention-days: 30 }
```

### GitHub Actions: official Action (organisation licence)

The `gitleaks/gitleaks-action@v2` Action requires `GITLEAKS_LICENSE` for repositories owned by an organisation (keys are issued free for OSS and small teams via gitleaks.io). It handles range computation, summaries and PR comments for you. Choose it when you want zero maintenance; choose the Docker approach when you want full control or cannot add a third-party licence secret.

### Reusable workflow for the whole organisation

```yaml
# org/.github/.github/workflows/gitleaks-reusable.yml
on:
  workflow_call:
    inputs:
      config: { type: string, default: .gitleaks.toml }
jobs:
  gitleaks:
    runs-on: ubuntu-latest
    container: { image: ghcr.io/gitleaks/gitleaks:v8.21.2, options: --user 0 }
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 0 }
      - run: git config --global --add safe.directory "$GITHUB_WORKSPACE"
      - run: gitleaks git . --config "${{ inputs.config }}" --redact --report-format sarif --report-path results.sarif
      - uses: github/codeql-action/upload-sarif@v3
        if: always()
        with: { sarif_file: results.sarif, category: gitleaks }
```

Consumers: `jobs: secrets: uses: <ORG>/.github/.github/workflows/gitleaks-reusable.yml@v1`. Combine with a GitHub ruleset that requires the `gitleaks` check on all repositories.

### Jenkins (declarative)

```groovy
stage('Gate 1: Gitleaks') {
  agent { docker { image 'ghcr.io/gitleaks/gitleaks:v8.21.2'; args '--entrypoint=""' } }
  steps {
    sh 'git config --global --add safe.directory "$WORKSPACE"'
    sh 'gitleaks git . --redact --no-banner --report-format junit --report-path gitleaks.xml'
  }
  post { always { junit 'gitleaks.xml' } }
}
```

### Azure DevOps

```yaml
- task: Bash@3
  displayName: Gate 1 Gitleaks
  inputs:
    targetType: inline
    script: |
      docker run --rm -v "$(Build.SourcesDirectory):/repo:ro" -v "$(Build.ArtifactStagingDirectory):/out" \
        ghcr.io/gitleaks/gitleaks:v8.21.2 git /repo --redact --no-banner \
        --report-format sarif --report-path /out/gitleaks.sarif
- task: PublishBuildArtifacts@1
  condition: always()
  inputs: { PathtoPublish: $(Build.ArtifactStagingDirectory)/gitleaks.sarif, ArtifactName: CodeAnalysisLogs }
```

The `CodeAnalysisLogs` artefact name is what the SARIF SAST Scans Tab extension reads.

---

## 4. Publishing to DefectDojo

DefectDojo has a native "Gitleaks Scan" parser for the JSON report. Reimport (rather than import) so findings are deduplicated and auto-closed when they disappear:

```bash
curl -sS -X POST "${DD_URL}/api/v2/reimport-scan/" \
  -H "Authorization: Token ${DD_TOKEN}" \
  -F "scan_type=Gitleaks Scan" \
  -F "file=@gitleaks.json" \
  -F "product_name=capstone" \
  -F "engagement_name=ci-${GITHUB_REF_NAME}" \
  -F "auto_create_context=true" \
  -F "close_old_findings=true" \
  -F "minimum_severity=Info" \
  -F "tags=gate-1,gitleaks"
```

Generate JSON **unredacted only in memory**, never as an artefact: if you must give DefectDojo the secret value for deduplication, restrict the DefectDojo product to the security team. Prefer `--redact` and let DefectDojo dedupe on file + line + rule. See [DefectDojo](../../aspm-and-siem/defectdojo/README.md).

---

## 5. False-Positive Governance

| Situation | Mechanism | Review requirement |
|---|---|---|
| Documented fake used in tests/docs | Global `[allowlist] regexes` with a comment | Security CODEOWNERS approve the config change |
| Whole fixture directory | `[allowlist] paths` with `(^|/)` anchor | Same |
| One historical finding (rotated or confirmed fake) | `.gitleaksignore` fingerprint with ticket in comment | PR template requires ticket link |
| Legacy backlog during onboarding | `--baseline-path` with redacted JSON | Time-boxed (90 days), tracked in DefectDojo |
| Noisy inherited rule | `[extend] disabledRules = ["..."]` | Justify why TruffleHog or another rule covers it |
| A single line, forever | `# gitleaks:allow` | Discouraged: easy to abuse; CI greps for allows added without a trailing reason |

CI check that every new `gitleaks:allow` and every `.gitleaksignore` line carries a reason:

```bash
# scripts/check-suppressions.sh
set -euo pipefail
BASE=$(git merge-base origin/main HEAD)
if git diff "$BASE"..HEAD -U0 | grep -E '^\+.*gitleaks:allow' | grep -vqE 'gitleaks:allow\s+--\s+\S+'; then
  echo "::error::gitleaks:allow without justification ('gitleaks:allow -- <reason/ticket>')"; exit 1
fi
if git diff "$BASE"..HEAD -U0 -- .gitleaksignore | grep -E '^\+[0-9a-f]' >/dev/null; then
  git diff "$BASE"..HEAD -U0 -- .gitleaksignore | grep -E '^\+#.*(SEC|JIRA|TICKET)-[0-9]+' >/dev/null \
    || { echo "::error::new .gitleaksignore entry without ticket comment"; exit 1; }
fi
```

Measure: false-positive rate = suppressed / total findings per month. Above 30% means rules need tuning; below 5% with many `generic-api-key` hits means you are probably missing things and should lower entropy or add rules.

---

## 6. Performance on Monorepos

| Problem | Pattern |
|---|---|
| Full-history scans take 20+ minutes | Move full scans to tier 3 nightly; PR gate uses merge-base range |
| Giant generated files | `--max-target-megabytes 5` and allowlist `(^|/)dist/`, `\.min\.js$`, `(^|/)__snapshots__/` |
| Lockfiles with hashes trigger entropy rules | Allowlist `\.(lock|sum)$`, `package-lock\.json$`, `poetry\.lock$`, `uv\.lock$` |
| Shallow clone in CI breaks range | `fetch-depth: 0`, or `git fetch --deepen` until `merge-base` succeeds |
| Many repos, one runner | Parallelise with a matrix over repositories; cache the Docker image |
| Rules without `keywords` | Add keywords; a keyword-less regex runs on every fragment |

Benchmark before and after a config change:

```bash
time gitleaks git . --no-banner --exit-code 0 >/dev/null
```

---

## 7. Remediation Workflow

When a finding is a real credential (as opposed to a fake fixture), follow this order. **Rotation is the remediation; everything else is hygiene.**

```mermaid
stateDiagram-v2
    [*] --> Detected
    Detected --> Triaged: within 1 hour (Critical SLA)
    Triaged --> Rotated: new credential issued to all consumers
    Rotated --> Revoked: old credential disabled
    Revoked --> Audited: provider logs reviewed from first-commit time
    Audited --> Purged: git filter-repo + force push + caches
    Purged --> Prevented: rule/allowlist/hook fixes, postmortem
    Prevented --> [*]
```

1. **Triage (minutes).** Confirm it is real: `trufflehog git file://. --only-verified` on the same repo, or check the provider console. Determine the exposure window: `git log --all -S '<secret>' --format='%H %aI %ae' | tail -1` gives the first commit. Determine visibility: public repo? forks? which tags/releases contain it (`git tag --contains <commit>`)?
2. **Rotate (minutes to an hour).** Issue the new credential. Update consumers through the secret store (Vault KV/dynamic secret, Kubernetes Secret via External Secrets, GitHub Actions secret), **not** by committing the new value. Verify consumers are healthy.
3. **Revoke.** Disable the old key (`aws iam update-access-key --status Inactive` then `delete-access-key`; GitHub: delete the PAT; Slack: revoke the token).
4. **Audit.** CloudTrail: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=AccessKeyId,AttributeValue=<KEY>` from the first-commit timestamp. GitHub audit log for the PAT. Treat unexplained use as a breach and escalate to incident response.
5. **Purge.**

   ```bash
   pipx install git-filter-repo
   git clone --mirror git@github.com:<ORG>/<REPO>.git && cd <REPO>.git
   printf '%s\n' '<THE_SECRET>==>REDACTED' > /tmp/expressions.txt
   git filter-repo --replace-text /tmp/expressions.txt
   git push --force --all && git push --force --tags
   ```

   Then: ask collaborators to re-clone (rebasing stale clones re-introduces the secret), delete and recreate forks, purge CI caches and artefacts, contact GitHub Support to purge cached views of the old commits, and invalidate any package or image that embedded the secret.
6. **Prevent.** If the format was not detected, add a rule (with tests). If it was detected but the hook was absent, enforce the PR gate as a required status check. Write a blameless postmortem; the person who committed it is rarely the root cause.

The audit entry for a confirmed fake (lab fixture) is simply the allowlist/fingerprint change with a ticket reference.

---

## 8. Security Hardening of the Scan Itself

- **Redact everywhere.** `--redact` in hooks, CI logs, artefacts. A scan report is a list of secrets.
- **Least-privilege tokens.** The CI job needs `contents: read` and `security-events: write`; nothing else.
- **Pin versions.** Image tag `v8.21.2`, pre-commit `rev`, Action SHA. Verify release checksums for binaries.
- **Protect the config.** `.gitleaks.toml`, `.gitleaksignore` and the baseline are security controls: add them to `CODEOWNERS` for the security team.
- **Scan the scanner's inputs.** Terraform state, Helm values, `.env` files for review environments; the scan is only as good as what you point it at.
- **Do not log the finding line in Slack notifications.** Send file, rule, commit and author; never the secret.

---

## 9. Anti-patterns

| Anti-pattern | Why it hurts | Instead |
|---|---|---|
| `--exit-code 0` in the PR gate "temporarily" | Gate becomes decoration; stays that way | Baseline the backlog, keep exit code 1 |
| Deleting the secret in a new commit and calling it fixed | It is still in history and already compromised | Rotate, then purge |
| One huge `[allowlist] regexes` of real past secrets | You just committed your secrets to the config | Use fingerprints or baseline, and rotate them |
| `stopwords = ["test"]` globally | Drops real keys containing `test` | Path allowlist for fixture dirs |
| Rules without `keywords` | Slow scans, developers disable the hook | Always add keywords |
| Shallow clone + range scan | Scans nothing or everything | `fetch-depth: 0` |
| Relying on the hook alone | `--no-verify` exists | PR gate as required check |
| Scanning only code repos | Secrets live in wikis, issues, tfstate, images, logs | TruffleHog org/S3/docker scans in tier 3 |

---

## 10. Decision Checklist

- [ ] Pre-commit hook configured via `.pre-commit-config.yaml` and documented in the repo README.
- [ ] PR workflow is a **required status check** on protected branches; uses merge-base range; `fetch-depth: 0`.
- [ ] `.gitleaks.toml` extends the organisation base; custom rules have positive and negative fixtures run in CI.
- [ ] Suppressions carry a ticket; CI enforces it.
- [ ] Baseline (if any) is redacted, time-boxed and tracked in DefectDojo.
- [ ] SARIF uploaded under category `gitleaks`; JSON reimported into DefectDojo nightly.
- [ ] Nightly full-history scan across all repos, paired with TruffleHog verification.
- [ ] Rotation runbook exists at `runbooks/secret-leak.md` and has been exercised with a fake.
- [ ] `--redact` in every invocation that writes a log or artefact.
- [ ] Versions pinned; config protected by CODEOWNERS.

Next: [06-troubleshooting.md](06-troubleshooting.md).
