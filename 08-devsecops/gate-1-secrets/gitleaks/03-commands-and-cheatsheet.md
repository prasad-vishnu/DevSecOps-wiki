# Gitleaks: Commands and Cheatsheet

| | |
|---|---|
| **Cluster** | 08 DevSecOps |
| **Role** | Gate 1 anchor |
| **Prerequisites** | [02-core-concepts.md](02-core-concepts.md) |
| **Target version** | Gitleaks 8.18+ |
| **Capstone phase** | Phase 5 (Months 17-20) |

> Exhaustive reference organised by task. Long and short flags are both shown. Every command has a one-line explanation.

---

## 1. Global Flags (apply to all subcommands)

| Flag | Short | Meaning |
|---|---|---|
| `--config <file>` | `-c` | Config file path (default: `.gitleaks.toml` in target, else embedded default) |
| `--report-path <file>` | `-r` | Write report to file |
| `--report-format <fmt>` | `-f` | `json` (default), `csv`, `junit`, `sarif`, `template` |
| `--report-template <file>` | | Go template for `--report-format template` |
| `--baseline-path <file>` | `-b` | JSON report of findings to ignore |
| `--exit-code <n>` | | Exit code when leaks found (default `1`) |
| `--redact[=<pct>]` | | Redact secrets in output and report (optionally percentage, default 100) |
| `--verbose` | `-v` | Print each finding |
| `--no-banner` | | Suppress ASCII banner |
| `--no-color` | | Plain output |
| `--log-level <lvl>` | `-l` | `trace`, `debug`, `info` (default), `warn`, `error`, `fatal` |
| `--max-target-megabytes <n>` | | Skip files larger than this |
| `--enable-rule <id>` | | Only run these rules (repeatable) |
| `--gitleaks-ignore-path <path>` | `-i` | Path to `.gitleaksignore` (default: target root) |
| `--follow-symlinks` | | Follow symlinks in `dir` mode |
| `--ignore-gitleaks-allow` | | Ignore inline `gitleaks:allow` comments |
| `--max-decode-depth <n>` | | (v8.21+) Decode nested base64/percent/hex encodings up to n levels |
| `--max-archive-depth <n>` | | (v8.22+) Recurse into archives in `dir` mode |

---

## 2. Scanning Git History

```bash
gitleaks git .                                    # entire history, all refs, config auto-detected
gitleaks git /path/to/repo --verbose              # another repo, print findings
gitleaks git . --log-opts="--since=2026-01-01"    # commits since a date (any git log option works)
gitleaks git . --log-opts="origin/main..HEAD"     # only commits on this branch (PR scan)
gitleaks git . --log-opts="-n 50"                 # last 50 commits
gitleaks git . --log-opts="--no-merges --first-parent main"   # linear history of main only
gitleaks git . --log-opts="3f1c2b9^..3f1c2b9"     # a single commit
gitleaks git . --log-opts="--author=contractor@example.com"   # one author
gitleaks git . --redact --report-format sarif --report-path results.sarif   # SARIF for code scanning
gitleaks git . --baseline-path baseline.json      # only findings not in baseline
gitleaks git . --exit-code 0                      # report-only, never fail
gitleaks git . --enable-rule aws-access-token --enable-rule github-pat   # restrict to two rules
gitleaks git . --max-target-megabytes 5           # skip large blobs
```

Legacy equivalents: `gitleaks detect --source . [same flags]`.

### Computing the PR range robustly in CI

```bash
# In GitHub Actions on pull_request with fetch-depth: 0
BASE="origin/${GITHUB_BASE_REF}"
MERGE_BASE=$(git merge-base "$BASE" HEAD)
gitleaks git . --log-opts="${MERGE_BASE}..HEAD" --redact --report-format sarif --report-path results.sarif
```

---

## 3. Pre-commit and Working-Tree Scans

```bash
gitleaks git --pre-commit --staged                # what `git commit` is about to include
gitleaks git --pre-commit                         # unstaged working-tree changes vs HEAD
gitleaks git --pre-commit --staged --redact --verbose    # what the pre-commit framework hook runs
```

Legacy: `gitleaks protect --staged`.

---

## 4. Scanning Directories and Streams

```bash
gitleaks dir .                                    # current folder, full file contents, .git ignored
gitleaks dir ./dist --max-target-megabytes 20     # build output
gitleaks dir /mnt/extracted-image-rootfs          # an unpacked container image filesystem
gitleaks dir . --follow-symlinks
gitleaks dir ./artifacts --max-archive-depth 2    # look inside zip/tar (v8.22+)
cat terraform.tfstate | gitleaks stdin            # scan a stream
kubectl logs deploy/api --since=1h | gitleaks stdin --redact
docker run --rm ghcr.io/example/api:1.2.3 env | gitleaks stdin
```

Legacy: `gitleaks detect --no-git --source .`.

---

## 5. Docker Invocations

```bash
# Full history of the mounted repo
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/repo:ro" \
  ghcr.io/gitleaks/gitleaks:v8.21.2 git /repo --redact --verbose

# With a repo config and SARIF written to a writable mount
docker run --rm --user "$(id -u):$(id -g)" \
  -v "$PWD:/repo:ro" -v "$PWD/out:/out" \
  ghcr.io/gitleaks/gitleaks:v8.21.2 git /repo \
  --config /repo/.gitleaks.toml --report-format sarif --report-path /out/results.sarif

# Directory mode on a build context
docker run --rm -v "$PWD/dist:/scan:ro" ghcr.io/gitleaks/gitleaks:v8.21.2 dir /scan
```

If git complains about `dubious ownership` inside the container, add `-e GIT_CONFIG_COUNT=1 -e GIT_CONFIG_KEY_0=safe.directory -e GIT_CONFIG_VALUE_0=/repo`.

---

## 6. Config File Reference (`.gitleaks.toml`)

```toml
title = "my config"

[extend]
useDefault = true                     # start from embedded rules
# path = "../shared/base.gitleaks.toml"   # or extend another file (mutually exclusive with useDefault in older versions)
# disabledRules = ["generic-api-key"]      # (v8.19+) drop noisy inherited rules

[[rules]]
id = "..."
description = "..."
regex = '''...'''
secretGroup = 1
entropy = 3.5
keywords = ["..."]
path = '''...'''
tags = ["..."]
  [rules.allowlist]
  regexTarget = "match"               # "secret" (default) | "match" | "line"
  regexes = ['''...''']
  paths = ['''...''']
  stopwords = ["..."]
  commits = ["..."]

[allowlist]
description = "global"
regexTarget = "secret"
regexes = ['''...''']
paths = ['''...''']
commits = ["..."]
stopwords = ["..."]
```

Validation: there is no `--validate` flag; run a scan with `--log-level debug` and look for `ERR` lines, or scan a known-positive fixture and confirm the rule fires.

---

## 7. `.gitleaksignore`

```text
# fingerprint per line; comments allowed
<commit>:<path>:<rule-id>:<line>
<path>:<rule-id>:<line>                # dir-mode fingerprint
```

```bash
# Generate fingerprints from a JSON report after triage
jq -r '.[].Fingerprint' report.json
```

---

## 8. Report Post-processing with `jq`

```bash
# Count findings by rule
jq 'group_by(.RuleID) | map({rule: .[0].RuleID, n: length}) | sort_by(-.n)' report.json

# Unique files containing findings
jq -r '.[].File' report.json | sort -u

# Findings by author (who to talk to)
jq -r '.[] | "\(.Email)\t\(.RuleID)\t\(.File):\(.StartLine)"' report.json | sort | uniq -c

# Earliest commit per secret (where the exposure window starts)
jq -r 'group_by(.Secret) | map(min_by(.Date) | {rule: .RuleID, first: .Date, commit: .Commit})' report.json

# Convert to .gitleaksignore after manual review
jq -r '.[] | select(.File | test("^tests/")) | .Fingerprint' report.json >> .gitleaksignore
```

---

## 9. Pre-commit Framework

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.21.2
    hooks:
      - id: gitleaks
```

```bash
pre-commit install                                # install hook into .git/hooks
pre-commit run gitleaks --all-files               # run against whole tree (dir-like semantics)
pre-commit autoupdate --repo https://github.com/gitleaks/gitleaks   # bump rev
SKIP=gitleaks git commit -m "..."                 # skip this hook once (visible in shell history; PR check still runs)
```

---

## 10. GitHub Action Inputs

```yaml
- uses: gitleaks/gitleaks-action@v2
  env:
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
    GITLEAKS_LICENSE: ${{ secrets.GITLEAKS_LICENSE }}      # org repos only
    GITLEAKS_CONFIG: .gitleaks.toml                          # optional
    GITLEAKS_ENABLE_UPLOAD_ARTIFACT: "true"                  # results.sarif artefact
    GITLEAKS_ENABLE_SUMMARY: "true"                          # job summary table
    GITLEAKS_ENABLE_COMMENTS: "true"                         # PR comments on findings
    GITLEAKS_NOTIFY_USER_LIST: "@security-team"
    GITLEAKS_VERSION: "8.21.2"                               # pin CLI version
```

Event handling: on `pull_request` the Action scans `base..head`; on `push` it scans the pushed commits; on `schedule`/`workflow_dispatch` it scans full history.

---

## 11. Git Hygiene Commands Used Alongside Gitleaks

```bash
git log --all --oneline -- path/to/file          # when was this file touched (exposure window)
git log --all -S 'AKIAEXAMPLEEXAMPLE00' --oneline # commits that add/remove the string (pickaxe)
git log --all -G 'AKIA[A-Z0-9]{16}' --oneline     # regex pickaxe
git show <commit>:path/to/file                    # see the file at that commit
git branch -r --contains <commit>                 # which remote branches contain the leak
git tag --contains <commit>                       # which tags (releases!) contain it

# History purge (after rotation)
pipx install git-filter-repo
cat > /tmp/expressions.txt <<'EOF'
AKIAEXAMPLEEXAMPLE00==>REDACTED_AWS_KEY
regex:ghp_[0-9a-zA-Z]{36}==>REDACTED_GITHUB_PAT
EOF
git filter-repo --replace-text /tmp/expressions.txt --force
git push --force --all && git push --force --tags
```

---

## 12. Quick Reference Card

| Task | Command |
|---|---|
| Install | `brew install gitleaks` |
| Version | `gitleaks version` |
| Scan all history | `gitleaks git .` |
| Scan PR range | `gitleaks git . --log-opts="origin/main..HEAD"` |
| Scan staged | `gitleaks git --pre-commit --staged` |
| Scan folder | `gitleaks dir ./path` |
| Scan stream | `cmd \| gitleaks stdin` |
| SARIF out | `--report-format sarif --report-path results.sarif` |
| Redact | `--redact` |
| Baseline | `--baseline-path baseline.json` |
| Non-blocking | `--exit-code 0` |
| One rule only | `--enable-rule <id>` |
| Debug config | `--log-level debug` |
| Ignore one finding | add fingerprint to `.gitleaksignore` |
| Ignore one line | append `# gitleaks:allow` |

Next: [04-hands-on-labs.md](04-hands-on-labs.md).
