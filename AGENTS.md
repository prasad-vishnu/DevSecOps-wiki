# AGENTS.md

Markdown-only learning wiki that documents a 24-month Cloud / Platform / DevSecOps / SRE / AI-engineering roadmap. There is no application code, build or test suite here: the "product" is documentation, also intended as a Backstage TechDocs source. The capstone monorepo it describes lives elsewhere.

## Scope and structure

- [roadmap/README.md](roadmap/README.md) is the source of truth for scope: 12 clusters, anchor vs bridge tools, 6 phases. Folders `01-`..`12-` map 1:1 to roadmap clusters; `00-foundations` is a prerequisite layer.
- [capstone/](capstone/) describes the single "Unified Production Capstone" repo every tool ties back to (FastAPI + Spring Boot incident API, later an AI agent). Tool choices between siblings are recorded as ADRs in [capstone/adrs/](capstone/adrs/) (`NNNN-short-title.md`, from [templates/adr-template.md](templates/adr-template.md)).
- `08-devsecops/` is nested one level deeper: `gate-N-<name>/<tool>/` plus a gate `README.md`.
- [GLOSSARY.md](GLOSSARY.md) holds shared terms; link to it rather than redefining.
- Many tool folders are still empty scaffolding and several kits are partial. Check what exists before linking to a file; do not create links to files that are not there.
- The root `README.md` referenced by the conventions and CONTRIBUTING does not exist yet.

## Conventions (read before writing)

Follow [templates/doc-conventions.md](templates/doc-conventions.md) and [CONTRIBUTING.md](CONTRIBUTING.md); do not restate them. Key points that are easy to get wrong:

- **Eight-file kit per tool**, exact names: `README.md`, `01-installation-and-setup.md`, `02-core-concepts.md`, `03-commands-and-cheatsheet.md`, `04-hands-on-labs.md`, `05-production-patterns.md`, `06-troubleshooting.md`, `07-interview-questions.md`. New tool READMEs start from [templates/tool-readme-template.md](templates/tool-readme-template.md).
- **Practice (non-tool) topics** in `11-sre/` and `12-enterprise-ops/` use `01-getting-started.md` and `03-reference-and-cheatsheet.md` instead. Match the sibling folders in the same cluster.
- Large concept files may be split as `02a-`, `02b-`, `02c-` (see [04-containers-and-idp/kubernetes/](04-containers-and-idp/kubernetes/)).
- **Front-matter table** on every file. In practice READMEs include rows: Cluster, Role (Anchor / Bridge with link to anchor), Category, Prerequisites, Target version, Capstone phase, Official docs. Follow it with a `>` summary paragraph. Example: [11-sre/sli-slo-error-budgets/README.md](11-sre/sli-slo-error-budgets/README.md).
- Bridge tools need a "Coming from <anchor>" mapping; every tool README ends with "Role in the Capstone".
- Every fenced code block declares a language; diagrams are Mermaid (`flowchart`, `sequenceDiagram`, `stateDiagram-v2`).
- Relative links only. Placeholders like `<ACCOUNT_ID>`, `123456789012`, `example.com`, `ghp_EXAMPLE000...`; never realistic secrets (Gitleaks runs in pre-commit).
- Security-by-default examples (non-root, least privilege, OIDC, short-lived creds); flag any lab shortcut in a callout.
- Respect target lengths per file from the conventions table; no marketing tone.

## When adding or changing content

1. Adding a tool: create the full kit, then update the cluster `README.md` comparison matrix and learning order.
2. Picking between sibling tools for the capstone: add an ADR and list it in [capstone/adrs/README.md](capstone/adrs/README.md).
3. Commits use Conventional Commits with the tool as scope, e.g. `docs(kubernetes): ...`.

## Validation

Environment is Windows (PowerShell) and the repo sits under a OneDrive path with spaces, so quote paths. The `find | xargs` link-check in CONTRIBUTING is bash-only; on PowerShell use:

```powershell
markdownlint-cli2 "**/*.md" "#node_modules"
Get-ChildItem -Recurse -Filter *.md | ForEach-Object { markdown-link-check -q $_.FullName }
```
