# Helm: Interview Questions

| | |
|---|---|
| **Cluster** | 04 Containers & IDP |
| **Role** | Bridge (from Kubernetes) |
| **Prerequisites** | The rest of this folder |
| **Target version** | Helm 3.16+, Kubernetes 1.31+ |
| **Capstone phase** | Phase 3 (Months 9-12) |

> Thirty-two questions from junior to principal level with model answers. Use them to check your understanding after the labs, and to practise explaining Helm's trade-offs out loud. Principal-level answers are expected to cover the *why* and the failure modes, not just the mechanism.

## Junior

### Q1. What is a Helm chart and what are its minimum contents? (Junior)

**Answer.** A chart is a packaged set of Kubernetes manifests written as Go templates plus the metadata and default values needed to render them. The minimum is `Chart.yaml` (name, version, `apiVersion: v2`) and a `templates/` directory; in practice you also have `values.yaml` with defaults. Optional parts are `charts/` for dependencies, `crds/`, `values.schema.json`, `templates/_helpers.tpl` for named templates, `templates/NOTES.txt`, and `templates/tests/`.

### Q2. What is the difference between `version` and `appVersion` in `Chart.yaml`? (Junior)

**Answer.** `version` is the chart's own SemVer and must change whenever any chart file changes; it becomes the OCI tag and is what Argo CD pins. `appVersion` is the version of the software the chart deploys and is informational; charts commonly use it as the default image tag with `.Values.image.tag | default .Chart.AppVersion`. A chart-only fix bumps `version`; a new application release bumps both.

### Q3. How does `helm install` differ from `kubectl apply -f`? (Junior)

**Answer.** `kubectl apply` is stateless: it sends each manifest and remembers nothing beyond the `last-applied-configuration` annotation. `helm install` renders templates with values, applies the result, and records the chart, values and rendered manifest as a release Secret so that later `upgrade`, `rollback`, `diff` and `uninstall` can operate on the whole set of objects together. Helm also runs hooks and can wait for readiness.

### Q4. In what order are values merged? (Junior)

**Answer.** Chart `values.yaml` first, then for subcharts the parent's values for that subchart and `global`, then `-f` files left to right, then `--set`/`--set-string`/`--set-file`/`--set-json` in command-line order. Later wins. Maps deep-merge; lists are replaced whole.

### Q5. Why would you use `--set-string` instead of `--set`? (Junior)

**Answer.** `--set` parses the value: `1.4.2` becomes a float, `true` becomes a boolean, `0123` becomes an integer. An image tag or a version string must stay a string, so `--set-string image.tag=1.4.2`. The same applies in values files: quote anything that looks numeric.

### Q6. What does `helm template` do and when do you use it? (Junior)

**Answer.** It renders the chart locally and prints the manifests without touching a cluster. Use it to debug templates (`--debug` shows computed values), to validate output with `kubeconform`, to scan with Trivy/Checkov, and in GitOps pipelines. Limitations: `lookup` returns empty and `.Capabilities` are defaults unless you pass `--api-versions` and `--kube-version`.

### Q7. What is `_helpers.tpl` and why does its name start with an underscore? (Junior)

**Answer.** It holds named templates (`define`) such as `fullname` and `labels` that other templates `include`. Files whose names start with `_` are rendered for their definitions but produce no Kubernetes manifest output, so a helper file never accidentally becomes an object.

## Mid-level

### Q8. Explain the difference between `include` and `template`. (Mid-level)

**Answer.** `template` is a Go template *action* that writes the named template's output directly into the stream and cannot be used in a pipeline. `include` is a Helm *function* that returns the rendered string, so it can be piped: `{{ include "api.labels" . | nindent 4 }}`. Because almost every helper output needs indentation, `include` is the only one you should use.

### Q9. Why does `{{ toYaml .Values.resources | indent 12 }}` often break and what is the fix? (Mid-level)

**Answer.** `indent` indents every line including the first, but the first line is being emitted right after `resources:` on the same line, so you get `resources:            requests:`. `nindent` prepends a newline and then indents, producing valid YAML. Combine with `{{-` to remove the whitespace before the action: `resources: {{- toYaml .Values.resources | nindent 12 }}`.

### Q10. Inside a `range` block, how do you access `.Release.Name`? (Mid-level)

**Answer.** `range` and `with` rebind `.` to the current element, so `.Release` no longer exists. Use the root variable `$`: `{{ $.Release.Name }}`. Likewise pass `$` into `include` calls inside loops. Alternatively capture a variable before the loop: `{{- $fullName := include "api.fullname" . }}`.

### Q11. What is a Helm release physically, and where is it stored? (Mid-level)

**Answer.** A Secret of type `helm.sh/release.v1` named `sh.helm.release.v1.<release>.v<revision>` in the release namespace. Its `data.release` field is base64(gzip(base64(JSON))) containing the chart, user-supplied values, rendered manifest, hooks and status. Each upgrade or rollback adds a new Secret; `--history-max` prunes old ones. `HELM_DRIVER` can switch to ConfigMaps or an SQL backend.

### Q12. What does `helm rollback` actually do? (Mid-level)

**Answer.** It reads the stored manifest of the target revision, performs a three-way merge against the live cluster state, applies it, and records the result as a *new* revision with status `deployed` and description `Rollback to N`. It does not undo database migrations or hooks; `pre-rollback`/`post-rollback` hooks run if defined. Values are restored too because they are part of the stored revision.

### Q13. Describe the three-way merge Helm performs on upgrade and one trap it causes. (Mid-level)

**Answer.** Helm compares the previous revision's manifest, the new rendered manifest and the live object. Fields the chart stops setting are removed; fields set live by other actors that the chart does not touch are preserved. Trap: if the chart sets `spec.replicas` while an HPA is scaling the Deployment, each upgrade resets replicas to the chart value, causing a scale-down then scale-up. Fix: omit `spec.replicas` when `autoscaling.enabled`.

### Q14. How do you make Pods restart when only a ConfigMap changes? (Mid-level)

**Answer.** Kubernetes does not restart Pods on ConfigMap changes. Put a hash of the rendered ConfigMap into a Pod template annotation: `checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}`. The annotation changes, the Pod template changes, the Deployment rolls. Alternatives are Reloader or Argo CD's `argocd.argoproj.io/sync-options` with a restart hook, but the checksum pattern is self-contained.

### Q15. What are hooks, how are they ordered, and what does `hook-delete-policy` control? (Mid-level)

**Answer.** Hooks are manifests annotated with `helm.sh/hook` (pre/post-install, upgrade, rollback, delete, test). Helm removes them from the normal apply set and runs them at the named lifecycle point, waiting for Jobs/Pods to complete. Ordering is by `helm.sh/hook-weight` (string, ascending, can be negative) then by kind. `helm.sh/hook-delete-policy` (`before-hook-creation`, `hook-succeeded`, `hook-failed`) controls when the hook object is deleted; `before-hook-creation,hook-succeeded` is the usual combination so failed Jobs stay for debugging but never block the next run with an immutable-field error.

### Q16. How do subcharts receive values, and what is `global`? (Mid-level)

**Answer.** A dependency named `redis` reads the parent's `redis:` block as its own `.Values`. Values flow down only; the subchart cannot read the parent's other keys. `global:` is visible to every chart in the tree and is used for registry mirrors, pull secrets and shared labels. `condition:` and `tags:` in `Chart.yaml` enable or disable a dependency from values.

### Q17. Compare Helm and Kustomize. When would you choose each? (Mid-level)

**Answer.** Helm generates manifests from templates and values; it has logic, packaging, distribution and release history, at the cost of template readability. Kustomize patches fixed base YAML with overlays; it is always valid YAML and built into kubectl, but has no loops, defaults, packaging or releases. Choose Helm for reusable, distributable, parameterised software (your service charts, third-party installs). Choose Kustomize for small per-environment patches on manifests you own. They compose: Kustomize's `helmCharts` field and Argo CD can render a chart and patch the output.

## Senior

### Q18. How do HTTP chart repositories and OCI registries differ, and why did Helm move to OCI? (Senior)

**Answer.** An HTTP repository is a static `index.yaml` listing chart versions plus tarballs; it needs `helm repo add/update`, has no immutability, and signing is a separate PGP `.prov` file. An OCI registry stores the chart as an OCI artifact with a content digest; the chart version becomes the tag, auth reuses registry credentials, charts live next to images, and Cosign signs the digest exactly as for images. OCI gives one distribution and signing story for images and charts, supports tag immutability, and removes the index-file race conditions. Downsides: no search/discovery, and you must pass `--version` explicitly.

### Q19. Your chart uses `lookup` to preserve a generated password. Argo CD shows the app perpetually OutOfSync. Why, and what is the correct design? (Senior)

**Answer.** Argo CD renders with `helm template`, where `lookup` returns empty, so the template falls into the `randAlphaNum` branch and produces a different password on every render; the diff never settles and a sync would rotate the credential. Correct design: never generate secrets in templates. Expose `existingSecret` (fail with `required` in prod) and source it from the External Secrets Operator, or accept a pre-created Secret. Rendering must be deterministic for GitOps.

### Q20. How does Helm handle CRDs, and how do you ship a chart that installs an operator plus its custom resources? (Senior)

**Answer.** Files in `crds/` are applied before templates, only on install, never upgraded, never deleted. This is deliberate because CRD deletion removes every CR cluster-wide. For an operator chart, either publish a separate `<name>-crds` chart with `helm.sh/resource-policy: keep` and install it first (Argo CD sync wave -1), or template CRDs behind `.Values.crds.install` accepting that Helm will then upgrade and (without `keep`) delete them. Custom resources that depend on the CRD cannot be in the same `helm install` as a `crds/` CRD unless the CRD is in `crds/` (applied first) rather than in `templates/`; with templates you need a post-install hook or a second chart.

### Q21. Walk through a CI pipeline for a chart repository in a regulated environment. (Senior)

**Answer.** On PR: `ct lint` (helm lint --strict, yamllint, `Chart.yaml` schema, mandatory version bump), `helm unittest` for security invariants (non-root, read-only FS, dropped caps), `helm template` per environment piped to `kubeconform -strict`, then `trivy config`/`checkov`/`kube-linter` on the rendered output, `ct install --upgrade` into a kind cluster to catch immutable-field breaks and run `helm test`, and a `helm-docs` freshness check. On merge/tag: `helm package`, `helm push oci://`, `cosign sign` keyless with the workflow's OIDC identity, `cosign attest` with an SPDX SBOM of the chart, and a promotion PR that bumps `targetRevision` in the GitOps repo and posts a `helm diff` against prod. Nothing runs `helm upgrade` against prod; Argo CD does.

### Q22. What are the limits of Helm's release storage and how would you mitigate them? (Senior)

**Answer.** Each revision is a Secret capped at 1 MiB (etcd object limit), containing the gzipped manifest and values. Charts that embed many dashboards via `.Files.Glob` or ship large CRDs can exceed it, failing with `Too long: must have at most 1048576 bytes`. Mitigations: split the chart, move large static files to their own ConfigMap-only chart, lower `--history-max`, or use `HELM_DRIVER=sql` with PostgreSQL. Under Argo CD the problem disappears because no release Secrets are written.

### Q23. How does Argo CD change the semantics of a Helm chart? (Senior)

**Answer.** Argo CD runs `helm template` (with `--api-versions` and `--kube-version` from the destination), then applies the output itself. Consequences: no release Secrets (`helm list` is empty), `lookup` is empty, hooks map to Argo CD hooks and `hook-weight` to sync waves (not all hook types map; `test` is skipped), CRDs in `crds/` *are* upgraded, health is assessed per resource kind rather than via `--wait`, rollback is `argocd app rollback` or `git revert`, and `--set` becomes `helm.parameters`. Design rule: if the chart renders correctly and deterministically under `helm template`, it works under Argo CD.

### Q24. A chart upgrade fails with `spec.selector: field is immutable`. What happened and what are your options? (Senior)

**Answer.** The Deployment's `spec.selector.matchLabels` changed between chart versions, usually because someone edited the `selectorLabels` helper or `nameOverride`. Kubernetes forbids selector changes. Options: revert the label change (correct fix; selector labels must be frozen for a chart's lifetime), or accept downtime by deleting the Deployment and upgrading (`helm upgrade --force` does delete-and-recreate), or blue-green by installing a new release name and switching the Service. Prevent with `ct install --upgrade` in CI, which installs the previous release then upgrades.

## Principal

### Q25. Design the chart strategy for a platform team supporting 60 microservices across dev/stage/prod on EKS. (Principal)

**Answer.** One library chart (`platform-lib`) owns the hardened Deployment/Service/HPA/PDB/NetworkPolicy/ServiceMonitor/ExternalSecret skeletons and standard labels; each service has a thin application chart that depends on it and overrides only what differs. Service charts live next to the service code so a PR changes code and chart together; charts are versioned independently of images and published to a private OCI registry, Cosign-signed with keyless OIDC. Environment deltas are values files in a separate GitOps repo consumed by Argo CD via multi-source Applications (chart from OCI, values from Git); promotion is a PR moving `image.digest` and `targetRevision`. Guardrails: `values.schema.json` generated from the library, PSS `restricted` on namespaces, Kyverno `verifyImages` and a policy requiring the standard labels, `ct`/unittest/kubeconform/trivy in CI, Renovate for pins, and a rule that cluster-scoped objects are forbidden in service charts. Backstage's golden-path template scaffolds the thin chart so teams never start from `helm create`. Third-party charts are mirrored into the registry and installed through Argo CD App-of-Apps with CRD charts in earlier sync waves.

### Q26. Argue for and against vendoring a database as a subchart of an application chart. (Principal)

**Answer.** For: one `helm install` gives a complete dev environment; `condition: postgresql.enabled` lets prod disable it; versions are locked in `Chart.lock`. Against: the database's lifecycle is coupled to the application's, so an app upgrade can trigger a StatefulSet change on the database; the umbrella grows toward the 1 MiB release limit; the subchart's values namespace leaks into the app's API surface and its breaking changes become yours; backups, PVC retention and major version upgrades are operator concerns the app chart cannot express. Recommendation: subchart for dev and ephemeral preview environments only, with `enabled: false` in stage/prod and a managed service (RDS/Aurora) referenced through an `ExternalSecret`. Record it as an ADR.

### Q27. How would you migrate a fleet from push-based `helm upgrade` in Jenkins to GitOps without downtime? (Principal)

**Answer.** First make every chart GitOps-safe: deterministic rendering (no `lookup`/`randAlphaNum`), hooks annotated with Argo CD equivalents, CRDs handled, `helm template --api-versions` equal to install output. Introduce a GitOps repo with values files extracted from `helm get values` for each live release. Install Argo CD with Applications in manual sync and `selfHeal: false`; Argo CD adopts objects by label matching, with `argocd app diff` showing zero drift when the migration is correct (Helm's `meta.helm.sh/*` annotations differ from Argo CD's tracking label, so set `application.resourceTrackingMethod: annotation` and add the annotation via a one-off `kubectl annotate`, or accept a one-time no-op diff). Freeze Jenkins deploys per service as each Application reaches `Synced`, then enable automated sync and `selfHeal`. Delete Helm release Secrets last, after a bake period, since they are harmless. Roll out per environment, dev first, and keep the Jenkins job as a break-glass with a documented procedure.

### Q28. Explain `--atomic`, `--wait`, `--cleanup-on-fail` and `--timeout` and how they interact. (Senior)

**Answer.** `--wait` makes Helm block until all Deployments, StatefulSets, DaemonSets, Services with load balancers and PVCs report ready (plus Jobs with `--wait-for-jobs`), up to `--timeout` (default 5m). `--atomic` implies `--wait` and rolls back to the previous revision if the install/upgrade fails or times out; on a first install it uninstalls. `--cleanup-on-fail` deletes only the *new* resources created during a failed upgrade without rolling back the rest. In CI, `--atomic --timeout 5m` with a job timeout longer than 5 minutes avoids both failed releases and stuck `pending-upgrade` locks. Under Argo CD none of these apply; health checks and sync retries replace them.

### Q29. How does `.Capabilities` help a chart work on heterogeneous clusters, and what is its limitation? (Senior)

**Answer.** `.Capabilities.KubeVersion` enables version gating (`semverCompare ">=1.29-0"`) and `.Capabilities.APIVersions.Has "monitoring.coreos.com/v1"` lets a chart skip a `ServiceMonitor` when the Prometheus Operator is absent. The limitation is that under `helm template` these reflect the Helm binary's defaults, not any cluster, so CI renders and Argo CD must pass `--api-versions`/`--kube-version` (Argo CD does so automatically from the destination). A chart whose output depends on capabilities must be tested with both settings.

### Q30. A teammate proposes a single umbrella chart that installs the whole platform (ingress, cert-manager, Prometheus, Argo CD, every service). Evaluate. (Principal)

**Answer.** It gives one-command bootstrap and one version number, which is attractive for demos. In production it fails on several axes: a single failing resource fails the whole release; the release Secret approaches the 1 MiB limit; CRD ordering between cert-manager, Prometheus Operator and their CRs cannot be expressed inside one install (`crds/` are install-only, hooks are untracked); every subchart's values namespace becomes part of one enormous values file; and the blast radius of an upgrade is the entire platform. The alternative is an Argo CD App-of-Apps (or ApplicationSet) with each component as its own Application, sync waves for ordering, and per-component values; bootstrapping remains one command (`kubectl apply -f root-app.yaml`). Keep umbrella charts for local dev environments only.

### Q31. How would you expose a chart-level "environment" switch without duplicating templates? (Principal)

**Answer.** Avoid an `environment: prod` key that templates branch on; it spreads environment logic through the chart and makes the values file an incomplete description of the deployment. Instead, keep the chart environment-agnostic and express every difference as a value (`replicaCount`, `autoscaling.*`, `resources`, `ingress.hosts`, `image.digest`, `podDisruptionBudget.minAvailable`). Environment files then fully describe the deployment and `helm template -f envs/prod/values.yaml` is the complete truth. Where a profile is genuinely useful (for example "preview environment" defaults), ship it as a *values file* in the chart (`values-preview.yaml`) that users layer with `-f`, not as template logic.

### Q32. What would make you *not* use Helm for a given workload? (Principal)

**Answer.** When there is no variation to parameterise and no distribution need, plain manifests or Kustomize are simpler and more readable. When the lifecycle requires in-cluster reconciliation and domain logic (databases, message brokers, anything with ordered upgrades, backup coordination or failover), an operator is the right abstraction and Helm should only install the operator. When the configuration surface is so large that the values file reinvents the Kubernetes API (`extraManifests: []` passthroughs everywhere), the chart adds no value over raw YAML. When the consumer is a single team that also writes the manifests, Helm's templating cost outweighs its reuse benefit; a Kustomize base per service with environment overlays is enough. Helm's strengths are packaging, distribution and release bookkeeping; if none of the three is needed, skip it.

---

Back to the [Helm README](README.md) or continue to [Argo CD](../argocd/README.md).
