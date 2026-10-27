# Argo CD: Interview Questions

| | |
|---|---|
| **Cluster** | 04 Containers & IDP |
| **Role** | Bridge |
| **Prerequisites** | The rest of this folder |
| **Target version** | Argo CD 2.12+ |
| **Capstone phase** | Phase 3 (Months 9-12) |

> Thirty questions from junior to principal level with model answers. Junior questions test vocabulary; mid-level test the reconciliation model and day-two operations; senior test multi-tenancy, scale and failure modes; principal test architecture judgement and trade-offs you can defend in an ADR.

---

## Junior

### Q1. What is GitOps, and how is it different from "CI/CD that deploys with kubectl"? (Junior)

**Answer.** GitOps is an operating model with four principles (OpenGitOps): the system is described declaratively, the desired state is versioned and immutable (Git), agents *pull* that state, and agents continuously reconcile actual state toward it. A CI pipeline running `kubectl apply` pushes once and stops watching; it needs cluster credentials on the CI runner and does nothing about drift. Argo CD runs inside the cluster, pulls from Git, shows the diff at all times, and can revert out-of-band changes.

### Q2. What are the main components of Argo CD and what does each do? (Junior)

**Answer.** `argocd-server` (API, UI, auth, RBAC), `argocd-repo-server` (clones repos and renders Helm/Kustomize/plain manifests), `argocd-application-controller` (watches clusters, compares live vs desired, performs syncs), Redis (cache of rendered manifests and cluster state), `argocd-applicationset-controller` (expands ApplicationSets into Applications), `argocd-notifications-controller` (sends events to Slack/GitHub/etc.), and optionally Dex (OIDC federation).

### Q3. What is an `Application`? (Junior)

**Answer.** A CRD in the `argoproj.io` group that binds a *source* (repo URL, revision, path or chart, plus Helm/Kustomize options) to a *destination* (cluster and namespace) under an *AppProject*, with a *sync policy* (manual or automated, prune, selfHeal, sync options). Its status holds sync status, health status, history and conditions.

### Q4. Explain the difference between sync status and health status. (Junior)

**Answer.** Sync status answers "do the live resources match the rendered manifests at the target revision?" (`Synced`, `OutOfSync`, `Unknown`). Health answers "are those resources working?" (`Healthy`, `Progressing`, `Degraded`, `Suspended`, `Missing`). They are independent: a Deployment with an image typo is `Synced` and `Degraded`; a manually scaled Deployment is `OutOfSync` and `Healthy`.

### Q5. What does `prune` do and why is it off by default? (Junior)

**Answer.** Prune deletes live resources that Argo CD tracks but that are no longer present in Git. It is off by default because an accidental deletion in Git, a wrong path, or an empty render could remove production workloads. Teams enable it after confidence, and pair it with `allowEmpty: false` and `Prune=false` annotations on critical resources like CRDs.

### Q6. What is `selfHeal`? (Junior)

**Answer.** With `syncPolicy.automated.selfHeal: true`, Argo CD triggers a sync whenever the live state diverges from the last synced revision, not only when Git changes. It reverts `kubectl edit`, `kubectl scale` and other out-of-band changes within seconds.

### Q7. How does Argo CD know which cluster resources belong to which Application? (Junior)

**Answer.** Through resource tracking: either the `app.kubernetes.io/instance` label (legacy) or the `argocd.argoproj.io/tracking-id` annotation (recommended, set via `application.resourceTrackingMethod: annotation`). The annotation method avoids collisions with Helm's use of the same label and supports longer names and apps in any namespace.

### Q8. How do you deploy a Helm chart with Argo CD, and what Helm feature does not work? (Junior)

**Answer.** Point `spec.source` at a Git path containing `Chart.yaml` or at a Helm/OCI repository with `chart` and `targetRevision`; supply `helm.valueFiles`, `values`, `valuesObject` or `parameters`. Argo CD runs `helm template` and applies the output, so there is no Helm release secret, `helm list` shows nothing, `helm test` is not run, and the `lookup` function returns empty because rendering happens without cluster access.

---

## Mid-Level

### Q9. Walk through what happens between a `git push` and pods running. (Mid-level)

**Answer.** A webhook (or the 3-minute poll) triggers a refresh. The controller asks the repo-server to render the source at the new revision; the repo-server fetches, renders (Helm/Kustomize/plugin), and caches the manifests in Redis. The controller compares them with live objects from its informer cache, normalising known differences and applying `ignoreDifferences`. If different, status becomes `OutOfSync`. With automated sync, the controller starts an operation: PreSync hooks, then Sync-phase resources wave by wave (waiting for health between waves), then PostSync hooks, pruning last if `PruneLast`. Each apply is `kubectl apply` (client- or server-side). Health is then recomputed from the live objects, and notifications fire on the resulting state.

### Q10. What are sync waves and hooks, and when do you use each? (Mid-level)

**Answer.** Waves (`argocd.argoproj.io/sync-wave`) order tracked resources within a phase: negative waves first. Use them for CRDs before CRs, Namespaces before contents, databases before apps. Hooks (`argocd.argoproj.io/hook: PreSync|Sync|PostSync|SyncFail|PostDelete`) are resources run during a phase but not tracked afterwards: schema migrations as PreSync Jobs, smoke tests as PostSync, cleanup as PostDelete. `hook-delete-policy` controls lifetime; `BeforeHookCreation` avoids "already exists" on re-runs.

### Q11. Production is flapping between Synced and OutOfSync every few minutes. Diagnose. (Mid-level)

**Answer.** Run `argocd app diff` to see the offending field. Typical culprits: an HPA changing `spec.replicas`, a mutating webhook injecting sidecars or defaults, cert-manager rotating `caBundle`, or a controller writing status-like fields into spec. Fix with `ignoreDifferences` (jsonPointers, jqPathExpressions, or `managedFieldsManagers` when using server-side apply), remove the field from Git if a controller owns it, and add `RespectIgnoreDifferences=true` so sync does not re-apply it.

### Q12. What is App-of-Apps and how does it compare with ApplicationSets? (Mid-level)

**Answer.** App-of-Apps is an Application whose source directory contains other Application manifests; syncing the root creates the children. It is the idiomatic way to bootstrap a cluster from one `kubectl apply`. ApplicationSets use generators (list, cluster, git directory/file, matrix, merge, pull request, SCM provider) to stamp out Applications from a template and keep them in sync with generator output. Use App-of-Apps for bootstrap and hand-curated sets; ApplicationSets for fan-out across environments and clusters.

### Q13. How do you handle secrets in a GitOps repo? (Mid-level)

**Answer.** Never commit Kubernetes Secret manifests: base64 is not encryption, and rotation would mean rewriting history. Options: External Secrets Operator (commit `ExternalSecret` CRs pointing at Vault/cloud secret managers; recommended), Sealed Secrets (asymmetric encryption, cluster-held key), SOPS via a Config Management Plugin or KSOPS (encrypted files in Git, decrypted in repo-server), or runtime injection (Vault Agent, CSI driver). Argo CD's own credentials follow the same rule and are populated by ESO.

### Q14. What does an AppProject control? (Mid-level)

**Answer.** Allowed source repositories, allowed destinations (cluster and namespace patterns), allowed cluster-scoped and namespaced resource kinds (whitelists/blacklists), project roles with RBAC policies and JWT tokens, OIDC group bindings, sync windows (allow/deny schedules), orphaned-resource monitoring, signature keys for signed commits, and (with apps-in-any-namespace) which namespaces may hold Applications. It is Argo CD's tenancy boundary.

### Q15. Explain Argo CD RBAC syntax and how you would give a team sync rights only on their project. (Mid-level)

**Answer.** Casbin CSV in `argocd-rbac-cm`: `p, <subject>, <resource>, <action>, <object>, allow|deny` and `g, <subject>, <role>`. Set `policy.default: ""` (deny). For a team: define a project role in the AppProject with `p, proj:payments:developer, applications, sync, payments/*, allow` and `groups: [payments-developers]`; the group claim comes from OIDC (`scopes: "[groups]"`). Test with `argocd admin settings rbac can`.

### Q16. What is server-side apply and why would you enable it in Argo CD? (Mid-level)

**Answer.** `ServerSideApply=true` makes Argo CD send manifests as `kubectl apply --server-side`, so the API server merges fields by *field manager* instead of relying on the `last-applied-configuration` annotation. Benefits: no 262 KB annotation limit (large CRDs such as Prometheus's fail otherwise), explicit ownership so multiple controllers can share an object, and more accurate diffs. Cost: conflicts surface as errors unless `Force` is used, and diffs become manager-aware, hence `managedFieldsManagers` in `ignoreDifferences`.

### Q17. How do you register a remote cluster securely? (Mid-level)

**Answer.** Avoid the default `argocd cluster add`, which creates a cluster-admin ServiceAccount with a long-lived token stored in Argo CD. Instead, create a cluster Secret whose `config` uses the provider's exec/IAM mechanism: `awsAuthConfig` with an IAM role assumed via IRSA or Pod Identity for EKS, `execProviderConfig` for GKE/AKS with workload identity. Grant the role only the Kubernetes permissions Argo CD needs, and register namespace-scoped (`--namespace`) where tenancy allows.

---

## Senior

### Q18. Design a repository layout and promotion flow for dev, stage and prod across three clusters. (Senior)

**Answer.** One GitOps repo separate from application code. `bootstrap/root.yaml` applied by Terraform; `platform/projects`, `platform/addons` (cluster-generator ApplicationSets for cert-manager, ESO, ingress, monitoring, Kyverno, Rollouts), `platform/apps/*.appset.yaml` (git directory generators over `workloads/<env>/<service>`); `workloads/<env>/<service>/values.yaml` holding only environment deltas; shared charts in an OCI registry pinned by version. Promotion is a Git commit copying `image.tag` (digest-pinned in prod) from one directory to the next, authored by a GitHub Actions workflow after the image is published, auto-merged for dev, reviewed for stage/prod. Prod Applications have `automated` without `selfHeal` disabled only during incidents, a weekend deny sync window, and Argo Rollouts canaries. Avoid branch-per-environment because it diverges and fits generators poorly.

### Q19. Argo CD is slow: refreshes take minutes and the UI times out. What do you check and change? (Senior)

**Answer.** Measure: `argocd_git_request_duration_seconds`, `argocd_repo_pending_request_total`, `argocd_app_reconcile` histogram, controller workqueue depth, Redis health, `argocd admin cluster stats`. Common causes: a huge monorepo re-rendered on every refresh, too few repo-server replicas, controller under-provisioned for the number of watched resources, high-churn kinds filling the cache. Fixes in order: webhooks and `manifest-generate-paths` to stop needless renders; scale repo-server and raise `reposerver.parallelism.limit`; shard the controller across clusters and raise `controller.status.processors`; `resource.exclusions` for Events/Leases/noisy CRDs; move charts to OCI; redis-ha. Also check the API server rate limits on target clusters.

### Q20. A sync pruned every resource in a namespace. How did it happen and how do you prevent recurrence? (Senior)

**Answer.** Likely a render that produced zero or few manifests (bad Kustomize path, chart condition flipping everything off, wrong branch) with `prune: true` and no `allowEmpty: false` guard, or a renamed ApplicationSet template that deleted and recreated Applications with finalizers. Prevent: `allowEmpty: false` (default, keep it), `Prune=false` annotations on stateful and CRD resources, `PruneLast=true`, PR checks that run `argocd app diff --local` and fail on large deletions, `preserveResourcesOnDeletion` and `applicationsSync: create-update` on ApplicationSets, and `FailOnSharedResource`. Recovery is `git revert` and sync; data recovery depends on PV reclaim policy and backups, which is why `Retain` on prod StorageClasses matters.

### Q21. Explain how Argo Rollouts integrates with Argo CD and how a canary is gated by Prometheus. (Senior)

**Answer.** Argo CD syncs a `Rollout` resource (a Deployment replacement) and `AnalysisTemplate`s like any manifest; Argo Rollouts' controller then executes the strategy. For canary with traffic routing (nginx, ALB, Istio, Gateway API), each `setWeight` step shifts traffic; `pause` steps wait; an `AnalysisRun` created from the template queries Prometheus on an interval (for example success rate from ingress metrics) and compares against `successCondition` with `failureLimit`/`inconclusiveLimit`. Failure aborts, routing 100% to stable and marking the Rollout `Degraded`, which Argo CD surfaces as app health. Success promotes the canary ReplicaSet to stable. Argo CD exposes `promote-full`/`abort` as resource actions. Query hygiene matters: `clamp_min` denominators so no-traffic windows do not produce NaN.

### Q22. How would you make Argo CD highly available and recoverable? (Senior)

**Answer.** HA: 3 argocd-server replicas with HPA and PDB, 3 repo-servers, controller StatefulSet sharded across clusters, redis-ha with HAProxy, anti-affinity across zones, NetworkPolicies, resource requests tuned to observed metrics. Recoverability: everything (Applications, Projects, ConfigMaps via Helm values, repo/cluster Secret templates) in Git; secret material via ESO; nightly `argocd admin export` to an encrypted versioned bucket as a secondary; a documented Git-first runbook (Terraform recreates the install, apply root app, wait for waves) drilled quarterly with measured RTO. Note that Redis is a cache and Application `.status`/history is disposable.

### Q23. Compare Argo CD and Flux for a 200-cluster fleet. (Senior)

**Answer.** Argo CD: hub-and-spoke with a UI, SSO/RBAC and ApplicationSets built in; strong for central visibility and PR-based preview environments; the hub becomes a scaling and blast-radius concern at 200 clusters, mitigated by sharding, per-region hubs or the agent architecture. Flux: per-cluster controllers with Kubernetes-native CRDs, true Helm releases (`helm upgrade`, `lookup`, tests), image automation built in, no UI; scales naturally with cluster count and has smaller blast radius but weaker central visibility and tenancy ergonomics (relies on Kubernetes RBAC and impersonation). Decision hinges on whether central UI/governance or per-cluster autonomy matters more; both are CNCF graduated and can coexist (Flux for cluster add-ons, Argo CD for app teams).

### Q24. How do you implement multi-tenancy so teams can self-serve Applications without platform tickets? (Senior)

**Answer.** Enable apps-in-any-namespace (`application.namespaces`) and set `sourceNamespaces` on each team's AppProject so teams create Applications in their own namespace using ordinary Kubernetes RBAC. Platform owns AppProjects with tight `sourceRepos`, `destinations`, empty `clusterResourceWhitelist`, blacklists for quota/NetworkPolicy kinds, and `orphanedResources.warn`. Argo CD RBAC grants teams `applications, *` within their project and `logs, get`, denies `exec` and `projects, update`. Kyverno enforces image signatures and PSS independently. Namespaces are provisioned by platform with labels, quotas and NetworkPolicy, or via `CreateNamespace=true` with `managedNamespaceMetadata` enforcing PSA labels.

---

## Principal

### Q25. Argue for or against Argo CD managing itself. (Principal)

**Answer.** For: Argo CD's configuration (ConfigMaps, RBAC, projects, notifications) is then versioned, reviewed and reconciled like everything else, and upgrades become chart version bumps in Git. Against: a bad config push can take down the thing that would fix it; `selfHeal` can fight generated secrets (`argocd-secret`, redis password) and the install's own CRDs. Recommended compromise: Terraform installs the Helm chart and owns `argocd-cmd-params-cm` and CRDs; an Argo CD Application manages the *rest* of Argo CD's config with `ignoreDifferences` for generated secrets, `prune: false`, and `Prune=false` on CRDs; a break-glass runbook recovers via Terraform if Argo CD wedges itself.

### Q26. A regulator asks for evidence of what was deployed to production on a given date and who approved it. How does your GitOps design answer? (Principal)

**Answer.** The GitOps repo's history is the deployment ledger: each prod change is a merged PR with reviewers (CODEOWNERS-enforced), signed commits (AppProject `signatureKeys` make Argo CD refuse unsigned revisions), and digest-pinned images. Argo CD notifications post sync results as GitHub commit statuses and to an append-only channel (Loki/S3 with object lock), supplementing the ephemeral Application history. Image provenance comes from Cosign attestations and SBOMs verified by Kyverno at admission. For "who approved": PR reviews plus Argo CD audit logs for any manual sync (`initiatedBy.username`). This chain of evidence is why `argocd app set` and `--local` syncs are forbidden in prod RBAC.

### Q27. When would you *not* use Argo CD? (Principal)

**Answer.** Non-Kubernetes targets (Argo CD deploys only to Kubernetes API servers; use Terraform/Crossplane/Pulumi for cloud resources, though Crossplane CRs can themselves be GitOps'd). Environments where in-cluster pull is impossible (locked-down clusters with no egress to Git and no mirror). Teams that need true Helm release semantics (`lookup`, `helm test`, release secrets) and no UI, where Flux fits better. Tiny single-cluster setups where a pipeline step is genuinely sufficient. Workflows requiring orchestration of ordered steps across systems, which is Argo Workflows' or a CI engine's job, not Argo CD's.

### Q28. Describe an incident caused by Argo CD behaviour and the systemic fix. (Principal)

**Answer.** Example: a chart upgrade changed a Deployment's selector labels. Argo CD's apply failed with `field is immutable`; an engineer added `Replace=true` globally to the Application to unblock it. The next sync replaced a PVC-backed StatefulSet, deleting the PVC references and causing data loss. Systemic fixes: scope `Replace=true`/`Force=true` to the single resource via annotation and only in a reviewed PR; `Retain` reclaim policy on prod StorageClasses; Velero backups; chart CI (`ct`, `helm diff`) that flags selector changes; policy (`conftest`) blocking `Force=true` at the Application level; a runbook that prefers blue-green chart renames over in-place immutable changes.

### Q29. How do you evolve a 500-Application single-hub Argo CD toward 5,000 Applications and 300 clusters without a big-bang migration? (Principal)

**Answer.** Phase 1: optimise the hub (webhooks, path filters, OCI charts, sharded controller, repo-server HPA, exclusions) and instrument it so you know the ceiling. Phase 2: split by blast radius before by scale: a separate prod hub and non-prod hub, each bootstrapped from the same Git repo with different cluster labels driving the cluster generator; AppProjects and RBAC are identical so users notice nothing beyond a URL. Phase 3: regional hubs or Argo CD Agent/agent-based control planes for far clusters; Applications move by relabelling cluster Secrets, which the ApplicationSets react to. Throughout, Backstage's Argo CD plugin aggregates across instances so developers keep a single view. Guardrails: `applicationsSync: create-update` during moves, `preserveResourcesOnDeletion`, and dry-runs via `argocd appset generate`.

### Q30. What are the security boundaries of Argo CD, and what is the realistic worst case if `argocd-server` is compromised? (Principal)

**Answer.** Boundaries: Argo CD RBAC (users to projects), AppProjects (projects to repos/destinations/kinds), cluster credentials (Argo CD to clusters), repo credentials (Argo CD to Git), and the policy engine at admission (what any applier may create). Worst case with a compromised API server: the attacker holds whatever the server can reach, which includes every cluster credential in the `argocd` namespace. If those are cluster-admin tokens, that is total fleet compromise. Mitigations that bound the damage: workload identity with least-privilege roles per cluster, namespace-scoped registration, Kyverno/Gatekeeper refusing unsigned images and privileged pods regardless of who applies, signed-commit enforcement so the attacker cannot inject manifests via Git, network policies limiting egress, disabled `exec`, short-lived sessions, and separate hubs for prod. The design goal is that Argo CD can only apply what a reviewed, signed Git revision says, and only within pre-approved destinations.

---

## Quick Self-Check

If you can answer these in one sentence each, you are ready for the senior bar: What is the difference between refresh and hard refresh? Why does `lookup` not work? What does `allowEmpty` protect against? Why prefer annotation tracking? What does a negative sync wave mean? Where should `Prune=false` always be set? What does `argocd admin export` include that Git does not?

Back to [README.md](README.md).
