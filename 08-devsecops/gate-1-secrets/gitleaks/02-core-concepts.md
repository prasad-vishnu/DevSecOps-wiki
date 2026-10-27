# Gitleaks: Core Concepts

| | |
|---|---|
| **Cluster** | 08 DevSecOps |
| **Role** | Gate 1 anchor |
| **Prerequisites** | [01-installation-and-setup.md](01-installation-and-setup.md) |
| **Target version** | Gitleaks 8.18+ |
| **Capstone phase** | Phase 5 (Months 17-20) |

> This file explains how Gitleaks turns a Git repository into findings: the scan sources, the rule engine, entropy, the allowlist layers, fingerprints and baselines, exit codes and report formats. By the end you can predict what a rule will and will not match before you run it.

---

## 1. Scan Sources

Gitleaks reads from exactly one source per invocation.

| Subcommand | Input | Mechanism | Typical use |
|---|---|---|---|
| `gitleaks git [path]` | A Git repository | Runs `git log -p --no-color --full-history --all` (plus `--log-opts`) and parses the unified diff stream. Each hunk added line is a candidate | Full-history audit, PR range scan |
| `gitleaks git --pre-commit --staged` | The index | Runs `git diff --staged -p` so only what is about to be committed is scanned | Pre-commit hook |
| `gitleaks git --pre-commit` | Working tree vs HEAD | `git diff -p` of unstaged changes | Editor integration |
| `gitleaks dir [path]` | Filesystem | Walks files (ignoring `.git/`), scans full file contents, no commit metadata | Build artefacts, extracted archives, non-Git projects |
| `gitleaks stdin` | Pipe | Reads text from stdin | `docker logs x \| gitleaks stdin`, `cat terraform.tfstate \| gitleaks stdin` |

Important consequences:

- **Git mode sees added lines only.** A secret present only in a removed line (`-` in the diff) was introduced in an earlier commit where it appears as `+`, so full-history mode still finds it there. In a *range* scan (`--log-opts="main..HEAD"`) a secret that was added before the range is invisible.
- **Directory mode sees the current state only.** It will not find a secret that was committed then deleted.
- **`--all` is on by default** in git mode (`--full-history --all`), so orphaned branches and tags are scanned too.

```mermaid
flowchart TD
    A["Which artefact?"] -->|"Git repo, want history"| B["gitleaks git ."]
    A -->|"Git repo, only this PR"| C["gitleaks git . --log-opts='origin/main..HEAD'"]
    A -->|"About to commit"| D["gitleaks git --pre-commit --staged"]
    A -->|"Folder / archive / build output"| E["gitleaks dir ./path"]
    A -->|"Log stream / tfstate / API response"| F["... | gitleaks stdin"]
```

---

## 2. Rule Anatomy

A rule is a TOML table in `[[rules]]`. Every field:

```toml
[[rules]]
id          = "internal-service-token"                 # unique, kebab-case; appears in reports
description = "Capstone internal service-to-service token"
regex       = '''(?i)\b(cap_[a-z0-9]{8}_[A-Za-z0-9_-]{40})\b'''
secretGroup = 1                                        # which capture group is the secret (default: whole match)
entropy     = 3.5                                      # min Shannon entropy of the secretGroup (optional)
keywords    = ["cap_"]                                 # case-insensitive prefilter; rule only runs on fragments containing one
path        = '''\.(py|java|yaml|yml|env|properties)$''' # optional: only files matching this regex
tags        = ["internal", "token", "capstone"]        # free-form; appear in JSON/SARIF
# Rule-scoped allowlist (v8.x): cancels matches of THIS rule only
[rules.allowlist]
regexes     = ['''cap_example_''']
paths       = ['''docs/''']
stopwords   = ["sample"]
```

### Regex notes

- Go's RE2 syntax: no lookahead/lookbehind, no backreferences. Use `\b` and explicit character classes instead.
- Triple single quotes `'''...'''` are TOML literal strings; backslashes are not escaped, which keeps regexes readable.
- Make the regex **anchor on structure** (prefix, fixed length, character class), not on variable names. `password\s*=\s*(.+)` matches everything; `(?i)password\s*[=:]\s*['"]([^'"]{12,})['"]` with `entropy = 3.5` is usable.
- `secretGroup` matters for redaction, entropy and fingerprinting. If omitted, the whole match (including `password =`) is treated as the secret and entropy is computed on that, diluting it.

### Built-in rules

`gitleaks` embeds `config/gitleaks.toml` with ~170 rules. Representative IDs you will see in reports:

| RuleID | Pattern summary |
|---|---|
| `aws-access-token` | `(A3T[A-Z0-9]\|AKIA\|ASIA\|ABIA\|ACCA)[A-Z0-9]{16}` |
| `github-pat` | `ghp_[0-9a-zA-Z]{36}` |
| `github-fine-grained-pat` | `github_pat_\w{82}` |
| `github-oauth` | `gho_[0-9a-zA-Z]{36}` |
| `gitlab-pat` | `glpat-[0-9a-zA-Z\-_]{20}` |
| `slack-bot-token` | `xoxb-[0-9]{10,13}-[0-9]{10,13}[a-zA-Z0-9-]*` |
| `slack-webhook-url` | `hooks.slack.com/services/T.../B.../...` |
| `stripe-access-token` | `(sk\|rk)_(test\|live\|prod)_[0-9a-zA-Z]{10,99}` |
| `private-key` | `-----BEGIN (RSA\|EC\|OPENSSH\|PGP\|DSA)? ?PRIVATE KEY` |
| `jwt` | `ey[a-zA-Z0-9]{17,}\.ey[a-zA-Z0-9\/\\_-]{17,}\.[a-zA-Z0-9\/\\_-]{10,}` |
| `generic-api-key` | Keyword-driven (`key`, `api`, `token`, `secret`, `password`...) followed by a 10-150 char value with entropy >= 3.5 |
| `hashicorp-tf-api-token` | `[a-z0-9]{14}\.atlasv1\.[a-z0-9\-_=]{60,70}` |
| `openai-api-key` | `sk-[a-zA-Z0-9]{20}T3BlbkFJ[a-zA-Z0-9]{20}` |
| `anthropic-api-key` | `sk-ant-api03-[a-zA-Z0-9_\-]{93}AA` |

Dump the embedded config to study it:

```bash
# The default config ships in the source tree; fetch the version matching your binary
curl -sSL "https://raw.githubusercontent.com/gitleaks/gitleaks/v$(gitleaks version)/config/gitleaks.toml" | less
```

---

## 3. Entropy

Shannon entropy of a string `s` with character frequencies `p_i` is `H = -sum(p_i * log2(p_i))`, in bits per character. Intuition:

| String | Entropy (bits/char) |
|---|---|
| `aaaaaaaaaaaaaaaa` | 0.0 |
| `password123456` | ~3.0 |
| English sentence | ~3.0-3.5 |
| `AKIAEXAMPLEEXAMPLE00` | ~3.1 (low because of repeated `EXAMPLE`) |
| 32-char hex (`[0-9a-f]`) random | ~3.9-4.0 (max 4.0) |
| 40-char base62 random | ~5.0-5.3 (max ~5.95) |
| 44-char base64 random | ~5.3-5.6 |

Rules of thumb:

- A precise-prefix rule (`AKIA...`) should have **no** entropy requirement or a low one; the structure is the evidence. (The default `aws-access-token` rule sets `entropy = 3`, which is why the fake key with repeated `EXAMPLE` still fires.)
- A generic rule (`generic-api-key`) **needs** entropy >= 3.5 or it matches every `password = "changeme"`.
- Entropy is computed on `secretGroup` only. Set it.
- Entropy does not distinguish a real key from a random test fixture. That is what allowlists and stopwords are for.

Compute it yourself to calibrate thresholds:

```python
import math, collections, sys
def shannon(s: str) -> float:
    counts = collections.Counter(s)
    return -sum(c/len(s) * math.log2(c/len(s)) for c in counts.values())
for s in sys.argv[1:]:
    print(f"{shannon(s):.3f}  {s}")
```

```bash
python3 entropy.py AKIAEXAMPLEEXAMPLE00 ghp_EXAMPLE000000000000000000000000000000 "$(openssl rand -hex 16)"
```

---

## 4. Allowlist Layers

Allowlists cancel matches. There are four scopes, evaluated after the regex matches.

| Scope | Where | Fields | Cancels |
|---|---|---|---|
| Global allowlist | `[allowlist]` top-level | `regexes`, `paths`, `commits`, `stopwords`, `regexTarget` | Any rule's match |
| Rule allowlist | `[rules.allowlist]` under a rule | same | Only that rule's match |
| Fingerprint ignore | `.gitleaksignore` file | one fingerprint per line | Exactly one historical finding |
| Baseline | `--baseline-path report.json` | previous findings | Everything already in the baseline |

### `regexTarget`

By default `regexes` are tested against the **secret** (the `secretGroup`). Set `regexTarget = "match"` to test against the whole regex match, or `regexTarget = "line"` to test against the entire line. Use `line` when you want to allowlist based on a trailing comment:

```toml
[allowlist]
regexTarget = "line"
regexes = ['''#\s*gitleaks:allow''']      # any line ending in "# gitleaks:allow" is skipped
```

(`gitleaks:allow` inline comments are also supported natively: a line containing `gitleaks:allow` is skipped regardless of config.)

### Stopwords

`stopwords` are substrings; if the secret contains any of them (case-insensitive) the match is dropped. The default config ships a large stopword list (`example`, `test`, `sample`, `fake`, `dummy`, ...) applied to the `generic-api-key` rule. Stopwords are blunt: `"test"` also drops a real key that happens to contain `test`. Prefer `paths` for fixture directories and `regexes` for specific known fakes.

### Commits

```toml
[allowlist]
commits = ["3f1c2b9e4d5a6b7c8d9e0f1a2b3c4d5e6f7a8b9c"]
```

Useful after a history purge or for a vendored import you have reviewed. Hashes are exact.

### Paths

Regexes on the repository-relative path. Always use `(^|/)` anchoring so `tests/` does not also match `contests/`:

```toml
paths = ['''(^|/)tests?/fixtures/''', '''\.(png|jpg|pdf|woff2?)$''', '''(^|/)go\.sum$''']
```

---

## 5. Fingerprints and `.gitleaksignore`

Every finding has a fingerprint:

```text
<commit-sha>:<file-path>:<rule-id>:<line-number>      # git mode
<file-path>:<rule-id>:<line-number>                   # dir mode
```

Put fingerprints in `.gitleaksignore` at the repo root to suppress exactly that finding:

```text
# .gitleaksignore
# Lab fixture: documented fake AWS key used in gate-1 exercises. Ticket SEC-101.
3f1c2b9e4d5a6b7c8d9e0f1a2b3c4d5e6f7a8b9c:docs/labs/fake-config.py:aws-access-token:2
```

Properties:

- Rewriting history changes the SHA and invalidates the entry.
- Moving the file or shifting the line changes the fingerprint. It is a precise, brittle tool by design: it suppresses *this* finding, not the pattern.
- Comments (`#`) are permitted. Require a ticket reference in review.

---

## 6. Baselines

A baseline is a previous JSON report. Findings whose fingerprints appear in the baseline are not reported, so only *new* leaks fail the gate.

```bash
# Day 0: accept the current state (after triage!)
gitleaks git . --report-format json --report-path gitleaks-baseline.json || true
git add gitleaks-baseline.json && git commit -m "chore(security): gitleaks baseline"

# Every subsequent run
gitleaks git . --baseline-path gitleaks-baseline.json --report-format sarif --report-path results.sarif
```

Baseline vs `.gitleaksignore`: the baseline is bulk and generated; the ignore file is curated and reviewed. Use the baseline to onboard a legacy repository, then burn it down to zero and delete it. A baseline is **not** remediation: every entry in it is a secret that must be rotated.

---

## 7. Exit Codes and Report Formats

| Exit code | Meaning |
|---|---|
| `0` | No leaks (or all allowlisted / baselined) |
| `1` | Leaks found |
| `126` | Error (bad config, not a git repo, git failure) |

Override with `--exit-code <n>` when you want findings to be non-blocking in a particular stage:

```bash
gitleaks git . --exit-code 0 --report-format sarif --report-path results.sarif   # report-only
```

| Format | Flag | Use |
|---|---|---|
| JSON | `--report-format json` | Baselines, scripting with `jq` |
| CSV | `--report-format csv` | Spreadsheets |
| JUnit | `--report-format junit` | Test-report tabs in Jenkins / Azure DevOps |
| SARIF | `--report-format sarif` | GitHub code scanning, DefectDojo, VS Code SARIF viewer |
| Template | `--report-format template --report-template custom.tmpl` | Go `text/template` custom output |

A JSON finding:

```json
[
  {
    "Description": "AWS",
    "StartLine": 2,
    "EndLine": 2,
    "StartColumn": 22,
    "EndColumn": 41,
    "Match": "AKIAEXAMPLEEXAMPLE00",
    "Secret": "AKIAEXAMPLEEXAMPLE00",
    "File": "config.py",
    "SymlinkFile": "",
    "Commit": "3f1c2b9e4d5a6b7c8d9e0f1a2b3c4d5e6f7a8b9c",
    "Entropy": 3.1219282,
    "Author": "demo",
    "Email": "demo@example.com",
    "Date": "2026-10-27T06:31:14Z",
    "Message": "add config",
    "Tags": [],
    "RuleID": "aws-access-token",
    "Fingerprint": "3f1c2b9e4d5a6b7c8d9e0f1a2b3c4d5e6f7a8b9c:config.py:aws-access-token:2"
  }
]
```

With `--redact` (or `--redact=50` for percentage) the `Secret` and `Match` fields are masked. Always redact in CI logs and artefacts; the report itself is otherwise a nicely formatted list of your secrets.

---

## 8. Performance Model

Time is roughly proportional to (bytes of added lines across scanned commits) x (rules whose keywords appear). Levers:

| Lever | Effect |
|---|---|
| `--log-opts="origin/main..HEAD"` | Scan only the PR range instead of all history |
| `fetch-depth` in CI | Shallow clones make range scans impossible; use `fetch-depth: 0` for PR jobs or compute the merge-base |
| `keywords` on custom rules | A rule without keywords runs its regex on every fragment |
| `--max-target-megabytes 10` | Skip giant files (minified JS, binaries, SQL dumps) |
| `[allowlist] paths` for lockfiles and vendored dirs | Skips scanning them entirely |
| `--no-banner`, `--redact` | Trivial, but reduce log noise |

A 50k-commit monorepo typically scans its full history in 1-3 minutes and a PR range in under 5 seconds.

---

## 9. What Gitleaks Does Not Do

- It does not verify that a secret is live. Pair with [TruffleHog](../trufflehog/README.md).
- It does not scan binary files meaningfully (regex over bytes; use `--max-target-megabytes` and path allowlists).
- It does not decode base64 or decrypt; a base64-encoded key is only caught by entropy rules.
- It does not find secrets whose shape you have not described and whose entropy is low (`password = "Summer2026!"`). That is a code-review and policy problem.
- It does not rewrite history or rotate anything. See [05-production-patterns.md](05-production-patterns.md).

Next: [03-commands-and-cheatsheet.md](03-commands-and-cheatsheet.md).
