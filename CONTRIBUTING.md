# Contributing to the DevSecOps Wiki

This wiki is a personal learning repository that is written to a professional standard so it can double as team documentation and a Backstage TechDocs source.

## Ground rules

1. **Follow the conventions.** Read [templates/doc-conventions.md](templates/doc-conventions.md) before adding or editing anything. Every tool folder uses the same eight-file kit and the same file names.
2. **Every command must run.** If you add a command, you ran it, or you label it clearly as pseudo-code. Include expected output where it helps.
3. **No secrets, no personal data.** Use `<PLACEHOLDER>` values, `123456789012` for AWS account ids, `example.com`, and obviously fake tokens such as `ghp_EXAMPLE000...`. Gitleaks runs in pre-commit; do not add allow-list entries to bypass it.
4. **Security by default.** Examples use non-root containers, least privilege, OIDC rather than static keys, and encryption in transit and at rest. If a lab takes a shortcut, say so in a callout.
5. **Prefer relative links.** They work on GitHub, in TechDocs and in local viewers.
6. **Write an ADR for tool choices.** Any time the capstone picks between sibling tools, add an ADR in [capstone/adrs/](capstone/adrs/).

## Local tooling

```bash
# Markdown lint and link check
npm install --global markdownlint-cli2 markdown-link-check
markdownlint-cli2 "**/*.md" "#node_modules"
find . -name "*.md" -not -path "./.git/*" -print0 | xargs -0 -n1 markdown-link-check -q

# Secret scan before commit
pre-commit install
pre-commit run --all-files
```

A starter `.pre-commit-config.yaml`:

```yaml
repos:
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.21.2
    hooks:
      - id: gitleaks
  - repo: https://github.com/DavidAnson/markdownlint-cli2
    rev: v0.14.0
    hooks:
      - id: markdownlint-cli2
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v5.0.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
```

## Adding a new tool

1. Create `NN-cluster/<tool>/` and copy [templates/tool-readme-template.md](templates/tool-readme-template.md) to `README.md`.
2. Create the seven numbered files with the exact names from the conventions.
3. Add the tool to the cluster `README.md` comparison matrix and learning order.
4. Add a line to the root `README.md` index.
5. If the tool is a bridge, include a "Coming from <anchor>" section mapping concepts one-to-one.

## Commit messages

Use Conventional Commits:

```text
docs(kubernetes): add Gateway API lab and troubleshooting for HTTPRoute
docs(adr): ADR-0005 Argo CD vs Flux
chore: fix broken relative links in 08-devsecops
```

## Review checklist

- [ ] File names match the kit exactly
- [ ] Front-matter table present (cluster, role, prerequisites, version, phase)
- [ ] Every code block declares a language
- [ ] Mermaid diagrams render on GitHub
- [ ] Labs have objective, prerequisites, steps, verify, cleanup
- [ ] No real credentials, hostnames, emails or personal data
- [ ] Links resolve
