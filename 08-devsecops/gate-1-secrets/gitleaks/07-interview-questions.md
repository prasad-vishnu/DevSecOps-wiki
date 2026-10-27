# Gitleaks: Interview Questions

| | |
|---|---|
| **Cluster** | 08 DevSecOps |
| **Role** | Gate 1 anchor |
| **Prerequisites** | Whole folder |
| **Target version** | Gitleaks 8.18+ |
| **Capstone phase** | Phase 5 (Months 17-20) |

> 28 questions from junior to principal level with model answers. Use them to check understanding and to rehearse explaining gate 1 to an interviewer.

---

## Junior

### Q1. What does Gitleaks do, in one sentence? (Junior)

**Answer.** It scans Git history, staged changes, directories or stdin for strings that look like credentials, using regex rules with keyword prefilters and Shannon-entropy thresholds, and exits non-zero when it finds one so a hook or CI job can block the change.

### Q2. Why is secret scanning the first gate in a DevSecOps pipeline? (Junior)

**Answer.** A leaked credential is compromised the moment it reaches a remote, regardless of whether the code is built or deployed. No other gate can run earlier than a pre-commit hook, the check costs milliseconds, and a fix commit does not undo the exposure, so catching it before push is worth more than catching anything else later.

### Q3. What is the difference between `gitleaks git .` and `gitleaks dir .`? (Junior)

**Answer.** `git` parses `git log -p` and scans every added line in every commit on every ref, with commit metadata; `dir` walks the filesystem and scans current file contents only, so it misses secrets that were committed and later deleted but works on non-Git inputs like build outputs or extracted images.

### Q4. What exit codes does Gitleaks return? (Junior)

**Answer.** `0` for no leaks (or all suppressed), `1` for leaks found, `126` for errors such as a bad config or not a Git repository. `--exit-code` overrides the leaks code, for example `--exit-code 0` for report-only runs.

### Q5. How do you install the pre-commit hook? (Junior)

**Answer.** Add `repo: https://github.com/gitleaks/gitleaks` with a pinned `rev` and `hooks: - id: gitleaks` to `.pre-commit-config.yaml`, then run `pre-commit install`. The hook runs `gitleaks git --pre-commit --redact --staged --verbose` on each commit.

### Q6. What does `--redact` do and why always use it in CI? (Junior)

**Answer.** It masks the `Secret` and `Match` fields in output and reports. Without it, the scan log and SARIF artefact are a nicely formatted list of your credentials visible to everyone with read access to the CI run.

---

## Mid-level

### Q7. Explain the role of `keywords` in a rule. (Mid)

**Answer.** `keywords` is a case-insensitive prefilter: the rule's regex only runs on fragments that contain at least one keyword. It is the main performance mechanism; a rule without keywords runs its regex against every fragment in the repository, which is what makes scans slow enough that developers disable the hook.

### Q8. What is `secretGroup` and what goes wrong if you omit it? (Mid)

**Answer.** It selects which regex capture group is the secret. Entropy, redaction and the reported `Secret` all use that group. If omitted, the whole match (for example `password = "abc..."`) is treated as the secret, which dilutes entropy with low-entropy prefix text and causes entropy-based rules to miss real values.

### Q9. Shannon entropy: what threshold would you set for a 32-character hex API key and why? (Mid)

**Answer.** Hex has at most 4.0 bits per character and a random 32-char hex string sits around 3.8-4.0; English text and placeholders sit around 3.0. A threshold of 3.5 separates them with margin. For base62 or base64 keys you can go higher (4.5+). For precise-prefix formats like `AKIA...` entropy is unnecessary because the structure is the evidence.

### Q10. Compare the four suppression mechanisms. (Mid)

**Answer.** Global/rule `[allowlist]` regexes and paths suppress a *pattern* everywhere (right for documented fakes and fixture directories); `.gitleaksignore` fingerprints suppress exactly one `(commit, file, rule, line)` finding and break when history is rewritten (right for triaged historical findings); `--baseline-path` subtracts a previous report in bulk (right for onboarding legacy repositories, time-boxed); inline `gitleaks:allow` suppresses one line going forward only (discouraged, easy to abuse).

### Q11. Why does an inline `# gitleaks:allow` not fix a finding in full-history mode? (Mid)

**Answer.** Full-history mode scans the commit where the line was *added*, and in that commit the line did not have the comment. The allow only affects commits made after it. Range scans of the new commit are clean; the historical finding needs a fingerprint or rotation.

### Q12. How do you scan only the commits in a pull request in GitHub Actions? (Mid)

**Answer.** Check out with `fetch-depth: 0`, fetch the base branch, compute `MB=$(git merge-base origin/$GITHUB_BASE_REF HEAD)` and run `gitleaks git . --log-opts="${MB}..HEAD"`. Shallow clones break this because `merge-base` cannot be found.

### Q13. What is SARIF and how does Gitleaks use it? (Mid)

**Answer.** SARIF is the OASIS JSON standard for static analysis results. `--report-format sarif` emits it; `github/codeql-action/upload-sarif` publishes it to GitHub code scanning so findings appear in the PR Security tab with a `category` (set to `gitleaks` so other tools do not overwrite it). DefectDojo and VS Code also consume SARIF.

### Q14. A developer runs `git commit --no-verify`. What stops the leak? (Mid)

**Answer.** Nothing on the laptop; the PR workflow does. The hook is a convenience, the PR check configured as a required status check on the protected branch is the control. The nightly full-history scan is the audit that catches admin direct pushes and force-pushes.

### Q15. Why is Gitleaks paired with TruffleHog rather than replaced by it? (Mid)

**Answer.** Gitleaks is fast, deterministic and easy to extend with internal token rules, which makes it ideal for hooks and PR gates. It cannot tell a live key from a revoked one. TruffleHog verifies candidates against provider APIs and scans non-Git sources (S3, images, org-wide issues), but is slower and network-bound. Each covers the other's weakness.

---

## Senior

### Q16. Design the organisation-wide rollout of Gitleaks for 400 repositories. (Senior)

**Answer.** One `org-security-config` repo with `base.gitleaks.toml` (defaults + internal token rules + global allowlists) and positive/negative fixtures; distribute via Renovate-managed vendored copy or a custom image with `GITLEAKS_CONFIG`. A reusable GitHub workflow invoked from each repo, enforced as a required check via an organisation ruleset. Pre-commit config in the golden-path template (Backstage). Nightly tier-3 full-history matrix scan feeding DefectDojo via reimport, paired with TruffleHog `--only-verified`. Baselines time-boxed to 90 days for brownfield repos. Metrics: findings per week, false-positive rate, time to rotate, hook adoption.

### Q17. A real AWS access key was committed to a public repository 3 days ago. Walk through remediation. (Senior)

**Answer.** Assume compromise. (1) Triage: confirm with `trufflehog --only-verified` or IAM console; find first commit with `git log --all -S KEY`. (2) Rotate: create a new key or, better, move the workload to an IAM role via OIDC; update consumers through the secret store. (3) Revoke: `aws iam update-access-key --status Inactive` then delete. (4) Audit CloudTrail by `AccessKeyId` from the first-commit time; unexplained calls mean incident response. (5) Purge with `git filter-repo --replace-text`, force-push, re-clone collaborators, delete forks, purge CI caches and registries, ask GitHub Support to purge cached views. (6) Prevent: required PR check, hook in template, rule if format was missed, blameless postmortem. Steps 5-6 are hygiene; step 2-3 is the fix.

### Q18. How would you keep the false-positive rate of the gate below 10%? (Senior)

**Answer.** Measure suppressed/total monthly. Anchor custom rules on structure with `secretGroup` and tuned entropy; add path allowlists with `(^|/)` for fixtures, lockfiles and generated code; use `disabledRules` for inherited rules that another tool covers better; require tickets on `.gitleaksignore` entries via a CI check; review top noisy rules monthly; add regression fixtures for every rule tweak so precision does not regress.

### Q19. Explain `regexTarget` and when you need `"line"`. (Senior)

**Answer.** Allowlist regexes test against the `secret` by default. `"match"` tests the entire regex match, `"line"` tests the whole source line. You need `"line"` when the decision depends on context outside the secret, such as a trailing `# test-fixture -- SEC-101` comment or a variable name like `EXAMPLE_KEY =`.

### Q20. Gitleaks finds nothing in CI but the same command finds a leak locally. Diagnose. (Senior)

**Answer.** Most likely a shallow clone (`fetch-depth: 1`) so `--log-opts` range is empty or `merge-base` failed and the scan covered zero commits; check `git rev-parse --is-shallow-repository` and the debug log's commit count. Other causes: a different config picked up (`--log-level debug` prints which), a baseline in CI, or the finding being in a commit outside the PR range.

### Q21. What are the limits of entropy-based detection and how do you compensate? (Senior)

**Answer.** Entropy cannot distinguish a random secret from a random test fixture, hash or UUID (false positives), and misses low-entropy real passwords (`Summer2026!`) and encoded secrets (false negatives). Compensate with structure-anchored rules, stopwords and path allowlists for fixtures, `--max-decode-depth` for encodings, Semgrep rules for assignments to password-like identifiers, and policy: configuration belongs in a secret store, not in Git.

### Q22. How do you version and test custom rules so they do not regress? (Senior)

**Answer.** Keep rules in a versioned config repo with a CHANGELOG; for each rule maintain a positive fixture that must produce a finding and a negative fixture that must be clean; run `gitleaks dir <fixture> --config` in CI with a non-default `--exit-code` to assert both; semantic-version the config and let Renovate propose updates to consuming repositories.

---

## Principal

### Q23. Argue for or against making the pre-commit hook mandatory and un-bypassable. (Principal)

**Answer.** Against: client-side controls cannot be enforced (`--no-verify`, fresh clones, non-Git tooling), attempts to make them mandatory create friction and workarounds, and the real control already exists server-side (required PR check plus GitHub push protection). For: hooks stop most leaks before they exist, which avoids rotation entirely. Resolution: make the hook the default via the golden-path template, measure adoption, and put the enforcement budget into the PR gate, branch protection and nightly audit.

### Q24. How does gate 1 interact with Vault (gate 7) and OIDC in a well-designed platform? (Principal)

**Answer.** Gate 1 detects the symptom; gates 7 and OIDC remove the cause. If workloads obtain credentials dynamically from Vault (short TTL) and CI authenticates to clouds with OIDC federation, there are almost no long-lived secrets to leak, and a leaked dynamic credential expires in minutes. Gitleaks then becomes a guardrail for the residual static secrets (third-party SaaS keys) and a detector of process failures, and its finding rate is a KPI for how far the platform has moved to identity-based access.

### Q25. Design the metrics and SLOs for the secrets gate. (Principal)

**Answer.** Leading: hook adoption percentage, percentage of repos with required check, rule-test coverage. Lagging: new findings per 1,000 commits, verified-live findings per month (TruffleHog), time from detection to rotation (SLO: Critical < 1 h), time to purge, false-positive rate (< 10%), baseline burn-down. Report in DefectDojo dashboards; page on verified-live only.

### Q26. A regulated customer forbids any tool from making outbound calls during scans. How does that shape gate 1? (Principal)

**Answer.** Gitleaks becomes the primary detector because it is offline and deterministic; TruffleHog is run with verification disabled (`--no-verification`) or only in an isolated environment where egress is permitted and logged. Compensate for missing verification with strict rules, fast rotation runbooks and provider-side detection (AWS GuardDuty credential-exfiltration findings, GitHub secret scanning partner alerts).

### Q27. How would you handle a monorepo where full-history scans take 40 minutes? (Principal)

**Answer.** Split tiers: PR gate uses merge-base range (seconds); tier-3 full scan runs nightly on a self-hosted runner with the repo cached, parallelised by commit ranges (`--log-opts` with date windows) across a matrix, with `--max-target-megabytes` and path allowlists for generated content; results reimported into DefectDojo. Consider a one-time full scan to establish a redacted baseline and then scan only commits since the last nightly (`--log-opts="--since=<last-run>"`).

### Q28. What would make you choose GitHub's native secret scanning and push protection over Gitleaks, or vice versa? (Principal)

**Answer.** GitHub native: zero maintenance, partner-verified patterns, server-side push protection that cannot be bypassed, but limited to GitHub-hosted repos, limited custom patterns (GHAS required for private repos) and no local hook or non-Git sources. Gitleaks: works anywhere (GitLab, Bitbucket, Azure DevOps, laptops, images, tfstate), fully customisable, free, but client/CI-side only. Mature organisations run both: push protection as the un-bypassable floor, Gitleaks for custom internal formats, hooks and non-GitHub sources, TruffleHog for verification and org-wide sweeps.

Back to [README](README.md). Sibling: [TruffleHog](../trufflehog/README.md).
