# Helm: Commands and Cheatsheet

| | |
|---|---|
| **Cluster** | 04 Containers & IDP |
| **Role** | Bridge (from Kubernetes) |
| **Prerequisites** | [02-core-concepts.md](02-core-concepts.md) |
| **Target version** | Helm 3.16+, helm-diff 3.9+, helm-secrets 4.6+, ct 3.11+ |
| **Capstone phase** | Phase 3 (Months 9-12) |

> An exhaustive, task-organised reference for the Helm CLI, its most-used plugins, chart-testing, and the template functions you reach for daily. Every command has a one-line explanation. Long-form flags are shown with short aliases in brackets.

## Global flags (work on every command)

| Flag | Meaning |
|---|---|
| `--namespace <ns>` `[-n]` | Target namespace (default: kubeconfig context namespace) |
| `--kube-context <ctx>` | Kubeconfig context to use |
| `--kubeconfig <path>` | Alternative kubeconfig |
| `--debug` | Verbose output; with `template`/`install` prints computed values and rendered hooks |
| `--kube-as-user`, `--kube-as-group` | Impersonate (useful to test RBAC) |
| `--burst-limit`, `--qps` | API client throttling for huge charts |
| `--registry-config`, `--repository-config`, `--repository-cache` | Override XDG paths |

```bash
helm env                                       # print all HELM_* variables and paths
helm version --short                           # v3.16.3+gcfd0749
helm help <command>                            # built-in docs
helm completion bash|zsh|fish|powershell       # shell completion script
```

## 1. Repositories

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami         # register an HTTP repo
helm repo add private https://charts.example.com --username <USER> --password-stdin < token.txt   # authenticated repo
helm repo add internal https://charts.internal.example.com --ca-file ./ca.crt                     # private CA
helm repo list                                                   # show registered repos
helm repo update                                                 # refresh every index.yaml
helm repo update bitnami                                         # refresh one repo
helm repo remove bitnami                                         # unregister
helm repo index ./dist --url https://charts.example.com          # generate index.yaml for a static repo
helm repo index ./dist --url https://charts.example.com --merge ./dist/index.yaml   # append to an existing index
```

## 2. Searching and inspecting charts

```bash
helm search repo redis                                   # search registered repos by name/keyword
helm search repo bitnami/redis --versions                # list every chart version
helm search repo redis --version '^20.0.0'               # SemVer constraint
helm search repo redis --devel                           # include pre-releases
helm search hub prometheus                               # search Artifact Hub
helm show chart bitnami/redis                            # print Chart.yaml
helm show values bitnami/redis --version 20.2.1          # print default values.yaml
helm show readme bitnami/redis                           # print README
helm show crds bitnami/redis                             # print bundled CRDs
helm show all oci://ghcr.io/example/charts/api --version 1.4.2   # everything, from an OCI ref
```

## 3. OCI registries

```bash
echo "$TOKEN" | helm registry login ghcr.io --username <USER> --password-stdin   # log in (token from env/stdin, never a flag)
helm registry login localhost:5000 --insecure                                     # plain-HTTP local registry
helm registry logout ghcr.io                                                      # remove stored credential
helm push api-1.4.2.tgz oci://ghcr.io/<ORG>/charts                                # push; name:version appended automatically
helm pull oci://ghcr.io/<ORG>/charts/api --version 1.4.2                           # download .tgz
helm pull oci://ghcr.io/<ORG>/charts/api --version 1.4.2 --untar --untardir ./vendor   # download and extract
helm pull bitnami/redis --version 20.2.1 --verify --keyring ~/.gnupg/pubring.gpg  # verify PGP provenance on pull
helm show values oci://ghcr.io/<ORG>/charts/api --version 1.4.2                   # inspect without pulling
```

## 4. Creating and developing a chart

```bash
helm create api                                 # scaffold from the built-in starter
helm create api --starter ./starters/platform   # scaffold from a custom starter in $HELM_DATA_HOME/starters
helm lint ./api                                 # static checks: Chart.yaml, YAML validity, best practices
helm lint ./api --strict                        # warnings become errors
helm lint ./api -f values-prod.yaml             # lint with a specific values file
helm lint ./api --with-subcharts                # also lint dependencies
helm template api ./api                         # render to stdout, no cluster needed
helm template api ./api -f values-prod.yaml --namespace api          # render with release name + namespace set
helm template api ./api --show-only templates/deployment.yaml        # one file only
helm template api ./api --set autoscaling.enabled=true --debug       # include computed values (stderr)
helm template api ./api --api-versions monitoring.coreos.com/v1 --kube-version 1.31.0   # emulate cluster capabilities
helm template api ./api --validate                                   # validate against the connected cluster's OpenAPI
helm template api ./api --output-dir ./rendered                      # write one file per template
helm template api ./api --no-hooks                                   # exclude hook manifests
helm template api ./api --include-crds                               # also print crds/
helm template api ./api | kubeconform -strict -summary -schema-location default \
  -schema-location 'https://raw.githubusercontent.com/datreeio/CRDs-catalog/main/{{.Group}}/{{.ResourceKind}}_{{.ResourceAPIVersion}}.json'   # schema-validate rendered output incl. CRDs
helm package ./api                              # produce api-<version>.tgz
helm package ./api --version 1.4.3 --app-version 1.4.3   # override versions at package time
helm package ./api --destination ./dist         # output directory
helm package ./api --dependency-update          # run dependency update first
helm package ./api --sign --key 'Platform Team' --keyring ~/.gnupg/secring.gpg   # produce .tgz.prov (PGP)
helm verify api-1.4.2.tgz --keyring ~/.gnupg/pubring.gpg   # verify provenance file
```

## 5. Dependencies

```bash
helm dependency list ./api        # show declared deps and their status (ok / missing / wrong version)
helm dependency update ./api      # resolve constraints, write Chart.lock, download to charts/
helm dependency build ./api       # download exactly what Chart.lock pins (CI should use this)
helm dependency update ./api --skip-refresh   # do not hit repo indexes
```

## 6. Installing and upgrading releases

```bash
helm install api ./api --namespace api --create-namespace                 # install from a directory
helm install api bitnami/redis --version 20.2.1 --namespace cache         # from a repo
helm install api oci://ghcr.io/<ORG>/charts/api --version 1.4.2 -n api    # from OCI
helm install api ./api --generate-name                                    # auto-generate a release name
helm install api ./api -f values.yaml -f values-prod.yaml                 # multiple values files, later wins
helm install api ./api --set replicaCount=3 --set-string image.tag=1.4.2  # inline overrides
helm install api ./api --set-file config.appToml=./app.toml               # file contents into a value
helm install api ./api --set-json 'resources={"limits":{"cpu":"500m"}}'   # JSON literal
helm install api ./api --dry-run=client                                   # render locally, print, do nothing
helm install api ./api --dry-run=server                                   # render with cluster lookups, do nothing
helm install api ./api --wait --timeout 5m                                # block until workloads are Ready
helm install api ./api --wait-for-jobs                                    # also wait for Jobs
helm install api ./api --atomic                                           # roll back automatically on failure (implies --wait)
helm install api ./api --skip-crds                                        # do not install crds/
helm install api ./api --no-hooks                                         # skip hooks
helm install api ./api --disable-openapi-validation                       # skip client-side schema check (rarely wise)
helm install api ./api --description "initial deploy from CI run 1234"    # free text stored in release
helm install api ./api --post-renderer ./kustomize-wrapper.sh             # pipe rendered YAML through a program
helm install api ./api --labels team=platform,tier=backend                # labels on the release Secret (3.13+)
helm install api ./api --enable-dns                                       # allow getHostByName in templates

helm upgrade api ./api -n api -f values-prod.yaml                         # upgrade; fails if release missing
helm upgrade --install api ./api -n api --create-namespace -f values-prod.yaml   # idempotent install-or-upgrade (use this in CI)
helm upgrade api ./api --reuse-values --set image.tag=1.4.3               # start from previous user values
helm upgrade api ./api --reset-values                                     # start from chart defaults (default behaviour)
helm upgrade api ./api --reset-then-reuse-values                          # defaults, then previous user values (3.14+)
helm upgrade api ./api --force                                            # delete/recreate on conflict (dangerous: downtime)
helm upgrade api ./api --cleanup-on-fail                                  # delete new resources created during a failed upgrade
helm upgrade api ./api --history-max 5                                    # keep at most 5 revision Secrets
helm upgrade api ./api --version 1.4.3                                    # from repo/OCI: specific chart version
helm upgrade api ./api --devel                                            # allow pre-release chart versions
helm upgrade api ./api --take-ownership                                   # adopt existing objects not managed by this release (3.17+)
```

## 7. Inspecting releases

```bash
helm list -n api                                 # releases in a namespace
helm list --all-namespaces [-A]                  # every release
helm list -A --all                               # include failed / pending / uninstalled-with-history
helm list -A --failed | --pending | --superseded | --uninstalled | --deployed   # filter by status
helm list -A --filter '^api'                     # regex on name
helm list -A --output json | jq '.[] | {name, namespace, chart, app_version, status}'   # machine-readable
helm list -A --date --reverse                    # newest first
helm list -A --selector team=platform            # by release labels (3.13+)
helm status api -n api                           # status, notes, last deployed
helm status api -n api --show-resources          # also list managed objects
helm status api -n api --revision 2              # status of a past revision
helm history api -n api                          # revision table
helm history api -n api --max 3 --output yaml    # last three, as YAML
helm get all api -n api                          # everything below in one go
helm get manifest api -n api                     # rendered manifest stored in the release
helm get manifest api -n api --revision 2        # a past revision's manifest
helm get values api -n api                       # user-supplied values
helm get values api -n api --all --output yaml   # merged with defaults
helm get hooks api -n api                        # hook manifests
helm get notes api -n api                        # NOTES.txt output
helm get metadata api -n api                     # name, chart, version, appVersion, namespace, revision, status
```

## 8. Rollback, test, uninstall

```bash
helm rollback api -n api                 # roll back to the previous revision
helm rollback api 2 -n api               # roll back to revision 2 (creates a new revision)
helm rollback api 2 -n api --wait --timeout 3m --cleanup-on-fail
helm rollback api 2 -n api --recreate-pods   # deprecated; restart pods (prefer checksum annotations)
helm test api -n api                     # run test hooks
helm test api -n api --logs              # and print their logs
helm test api -n api --filter name=api-test-connection   # run selected tests
helm uninstall api -n api                # delete every managed object and the release Secrets
helm uninstall api -n api --keep-history # keep release Secrets with status uninstalled (enables rollback)
helm uninstall api -n api --wait         # wait until objects are gone
helm uninstall api -n api --no-hooks     # skip pre/post-delete hooks
helm uninstall api -n api --dry-run      # show what would be deleted
helm uninstall api -n api --cascade=foreground|background|orphan   # deletion propagation
```

## 9. Plugins

```bash
helm plugin list
helm plugin install https://github.com/databus23/helm-diff --version v3.9.11
helm plugin update diff
helm plugin uninstall diff
```

### helm-diff

```bash
helm diff upgrade api ./api -n api -f values-prod.yaml                  # diff live release vs proposed upgrade
helm diff upgrade api ./api -n api -f values-prod.yaml --allow-unreleased   # also works when not yet installed
helm diff upgrade api ./api -n api --three-way-merge                    # include live-object drift in the diff
helm diff upgrade api ./api -n api --detailed-exitcode                  # exit 2 when there are changes (CI gate)
helm diff upgrade api ./api -n api --context 3 --no-color               # readable in CI logs
helm diff upgrade api ./api -n api --show-secrets                       # do not mask Secret data (default masks)
helm diff revision api 2 3 -n api                                       # diff two stored revisions
helm diff rollback api 2 -n api                                         # what a rollback would change
helm diff release api-dev api-stage -n api                              # diff two releases
```

### helm-secrets (SOPS)

```bash
helm secrets encrypt secrets.yaml > secrets.enc.yaml           # encrypt (SOPS config from .sops.yaml)
helm secrets decrypt secrets.enc.yaml                          # print plaintext to stdout
helm secrets edit secrets.enc.yaml                             # edit in $EDITOR, re-encrypt on save
helm secrets upgrade --install api ./api -n api -f values.yaml -f secrets://secrets.enc.yaml   # transparently decrypt during upgrade
helm secrets template api ./api -f secrets.enc.yaml            # same for template
helm secrets --backend vals upgrade ... -f values.yaml         # vals backend: ref+awssecrets://... placeholders in values
```

### helm-unittest

```bash
helm unittest ./api                                   # run tests/*_test.yaml
helm unittest ./api --color --output-type JUnit --output-file junit.xml   # CI report
helm unittest ./api --update-snapshot                 # refresh snapshot files
```

## 10. Chart testing (`ct`) and docs

```bash
ct lint --config ct.yaml                              # lint changed charts: helm lint, yamllint, Chart.yaml schema, version bump check
ct lint --config ct.yaml --all                        # all charts, not just changed
ct lint --config ct.yaml --charts charts/api          # explicit chart
ct install --config ct.yaml                           # install each changed chart into the current cluster with every ci/*-values.yaml, run helm test, uninstall
ct install --config ct.yaml --upgrade                 # also test upgrade from the previous released version
ct lint-and-install --config ct.yaml                  # both
ct list-changed --config ct.yaml                      # which charts changed vs target branch
helm-docs --chart-search-root charts                  # regenerate README.md tables from values.yaml comments
```

## 11. Environment variables

| Variable | Purpose |
|---|---|
| `HELM_NAMESPACE` | Default namespace |
| `HELM_DRIVER` | `secret` (default), `configmap`, `sql`, `memory` |
| `HELM_DRIVER_SQL_CONNECTION_STRING` | DSN for the sql driver |
| `HELM_MAX_HISTORY` | Default for `--history-max` |
| `HELM_DEBUG` | Same as `--debug` |
| `HELM_KUBECONTEXT`, `KUBECONFIG` | Cluster selection |
| `HELM_REGISTRY_CONFIG` | Registry credentials file |
| `HELM_EXPERIMENTAL_OCI` | Obsolete since 3.8; OCI is always on |
| `HELM_BURST_LIMIT`, `HELM_QPS` | API client throttling |
| `HELM_PLUGINS`, `HELM_CACHE_HOME`, `HELM_CONFIG_HOME`, `HELM_DATA_HOME` | Paths |
| `SOPS_AGE_KEY_FILE`, `SOPS_AGE_RECIPIENTS` | helm-secrets with age |

## 12. Template function quick reference

```yaml
# Strings
{{ .Values.name | upper | quote }}                 # "API"
{{ printf "%s-%s" .Release.Name .Chart.Name }}     # api-api
{{ .Values.host | trunc 63 | trimSuffix "-" }}     # DNS-safe
{{ .Values.s | replace "." "-" }}                   # replace
{{ .Values.s | trimPrefix "v" }}                    # 1.4.2 from v1.4.2
{{ regexReplaceAll "[^a-z0-9-]" (lower .Values.s) "-" }}   # sanitise
{{ .Values.csv | splitList "," | join "\n" }}
{{ .Values.s | b64enc }} / {{ .Values.s | b64dec }}
{{ .Values.s | sha256sum }}

# Defaults, required, fail, coalesce, ternary
{{ .Values.image.tag | default .Chart.AppVersion }}
{{ required "ingress.host is required when ingress.enabled" .Values.ingress.host }}
{{ if gt (int .Values.autoscaling.minReplicas) (int .Values.autoscaling.maxReplicas) }}{{ fail "minReplicas > maxReplicas" }}{{ end }}
{{ coalesce .Values.a .Values.b "fallback" }}       # first non-empty
{{ ternary "yes" "no" .Values.flag }}

# Types and conversion
{{ .Values.port | int }} / {{ .Values.n | int64 }} / {{ .Values.f | float64 }} / {{ .Values.x | toString }}
{{ kindIs "string" .Values.x }} / {{ typeOf .Values.x }} / {{ empty .Values.x }}
{{ .Values.obj | toYaml | nindent 4 }} / {{ .Values.obj | toJson }} / {{ .Values.s | fromYaml }} / {{ .Values.s | fromJson }}

# Lists
{{ has "x" .Values.list }} / {{ first .Values.list }} / {{ last .Values.list }} / {{ len .Values.list }}
{{ .Values.list | uniq | sortAlpha | join "," }}
{{ append .Values.list "new" }} / {{ prepend .Values.list "first" }} / {{ concat .Values.a .Values.b }}
{{ slice .Values.list 1 3 }} / {{ without .Values.list "x" }} / {{ compact .Values.list }}
{{ until 3 }}                                       # [0 1 2] for range loops
{{ range $i, $e := untilStep 0 10 2 }}...{{ end }}

# Maps
{{ hasKey .Values.m "k" }} / {{ keys .Values.m | sortAlpha }} / {{ values .Values.m }}
{{ get .Values.m "k" }} / {{ set .Values.m "k" "v" }} / {{ unset .Values.m "k" }}
{{ pick .Values.m "a" "b" }} / {{ omit .Values.m "secret" }}
{{ merge .Values.override .Values.base }}           # left wins; mutates first arg -> deepCopy first
{{ mergeOverwrite (deepCopy .Values.base) .Values.override }}   # right wins
{{ dict "key" "value" "k2" "v2" }} / {{ list 1 2 3 }}

# Comparison and logic (functions, not operators)
{{ if and .Values.a (or .Values.b .Values.c) }}
{{ if eq .Values.env "prod" }} / {{ if ne }} / {{ if lt }} / {{ if le }} / {{ if gt }} / {{ if ge }} / {{ if not }}
{{ semverCompare ">=1.29-0" .Capabilities.KubeVersion.Version }}
{{ .Capabilities.APIVersions.Has "monitoring.coreos.com/v1/ServiceMonitor" }}

# Math
{{ add 1 2 }} / {{ sub 5 1 }} / {{ mul 2 3 }} / {{ div 10 2 }} / {{ mod 7 3 }} / {{ max 1 5 }} / {{ min 1 5 }} / {{ ceil 1.2 }} / {{ floor 1.8 }}

# Dates and random (both non-deterministic; avoid in manifests)
{{ now | date "2006-01-02" }} / {{ randAlphaNum 16 }} / {{ uuidv4 }}

# Files
{{ .Files.Get "config/app.toml" }}
{{ (.Files.Glob "dashboards/*.json").AsConfig | nindent 2 }}
{{ range $path, $_ := .Files.Glob "scripts/*.sh" }}{{ $path }}: {{ $.Files.Get $path | quote }}{{ end }}

# Helm-specific
{{ include "api.fullname" . }}
{{ tpl .Values.extraManifest . }}
{{ lookup "v1" "Secret" .Release.Namespace "existing" }}      # map or empty dict
{{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
```

## 13. One-liners you will use every week

```bash
# Which image tag is prod running?
helm get values api -n api -o json | jq -r '.image.tag'

# Diff prod against what CI is about to apply, fail the job if anything changed
helm diff upgrade api oci://ghcr.io/<ORG>/charts/api --version "$CHART_VERSION" -n api -f envs/prod/values.yaml --detailed-exitcode

# List every release and its chart version across the cluster, sorted
helm list -A -o json | jq -r '.[] | "\(.namespace)\t\(.name)\t\(.chart)\t\(.status)"' | column -t | sort

# Find releases stuck in pending
helm list -A --pending

# Export the live values of every release to Git (poor man's backup)
for r in $(helm list -n api -q); do helm get values "$r" -n api -o yaml > "backup/$r.values.yaml"; done

# Render and validate every env in one go
for env in dev stage prod; do helm template api ./charts/api -f "envs/$env/values.yaml" | kubeconform -strict -summary || exit 1; done

# Show the resources a release owns
helm status api -n api --show-resources

# Find the release that owns an object
kubectl get deployment api -n api -o jsonpath='{.metadata.annotations.meta\.helm\.sh/release-name}{"\n"}'
```

Continue to [04-hands-on-labs.md](04-hands-on-labs.md).
