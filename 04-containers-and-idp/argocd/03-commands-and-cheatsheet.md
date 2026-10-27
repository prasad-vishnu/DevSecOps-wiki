# Argo CD: Commands and Cheatsheet

| | |
|---|---|
| **Cluster** | 04 Containers & IDP |
| **Role** | Bridge |
| **Prerequisites** | `argocd` CLI logged in ([01-installation-and-setup.md](01-installation-and-setup.md)) |
| **Target version** | Argo CD 2.12+, Argo Rollouts kubectl plugin 1.7+ |
| **Capstone phase** | Phase 3 (Months 9-12) |

> A task-organised reference for the `argocd` CLI, the `kubectl` equivalents (everything the CLI does is a CRUD on CRDs or a gRPC call), the key ConfigMaps and Secrets, the annotations you will paste into manifests, and the `kubectl argo rollouts` plugin. Each command has a one-line explanation. Long and short flags are both shown where they exist.

---

## 1. Login, Context and Accounts

| Command | Explanation |
|---|---|
| `argocd login <HOST> --sso --grpc-web` | Log in via the browser-based SSO flow; `--grpc-web` for ingress controllers without HTTP/2 passthrough |
| `argocd login <HOST> --username admin --password <PW> --insecure` | Local account login (labs only); `--insecure` skips TLS verification |
| `argocd login <HOST> --core` | Bypass the API server and talk to Kubernetes directly with your kubeconfig (no RBAC enforcement, admin-only) |
| `argocd login --port-forward --port-forward-namespace argocd` | Auto port-forward to `argocd-server`; handy on kind |
| `argocd context` | List saved contexts (servers) |
| `argocd context <HOST>` | Switch context |
| `argocd logout <HOST>` | Remove the saved token |
| `argocd account get-user-info` | Show current user, groups and whether logged in |
| `argocd account list` | List local accounts (`argocd-cm` `accounts.<name>`) |
| `argocd account update-password --account <NAME>` | Change a local account password |
| `argocd account generate-token --account ci-bot --expires-in 720h` | Issue a JWT for a local account (API key); store it in CI secrets |
| `argocd account delete-token --account ci-bot <ID>` | Revoke a token |
| `argocd account can-i sync applications 'payments/*'` | Ask RBAC whether the current user may do an action |
| `argocd version` / `argocd version --client` / `--short` | Server+client versions |

Environment variables: `ARGOCD_SERVER`, `ARGOCD_AUTH_TOKEN`, `ARGOCD_OPTS="--grpc-web --insecure"` (used in CI).

---

## 2. Repositories and Credentials

| Command | Explanation |
|---|---|
| `argocd repo list` | List repositories with connection status |
| `argocd repo add https://github.com/<ORG>/<REPO>.git` | Add a public HTTPS repo |
| `argocd repo add git@github.com:<ORG>/<REPO>.git --ssh-private-key-path key` | Add via SSH deploy key |
| `argocd repo add https://... --username x --password <TOKEN>` | Add via HTTPS token (prefer GitHub App) |
| `argocd repo add https://charts.example.com --type helm --name stable` | Add a Helm HTTP repository |
| `argocd repo add ghcr.io/<ORG>/charts --type helm --enable-oci --name oci-charts` | Add an OCI Helm registry |
| `argocd repo add <URL> --project payments` | Project-scoped repo (only usable by that project) |
| `argocd repo get <URL> --refresh` | Re-test connectivity |
| `argocd repo rm <URL>` | Remove |
| `argocd repocreds add https://github.com/<ORG>/ --github-app-id ... --github-app-installation-id ... --github-app-private-key-path app.pem` | Credential template matched by URL prefix |
| `argocd repocreds list` | List credential templates |
| `argocd cert add-tls example.com --from cert.pem` | Trust a custom CA for HTTPS Git |
| `argocd cert add-ssh --batch < known_hosts` | Add SSH known hosts |
| `argocd gpg add --from pubkey.asc` | Add a GPG key for signed-commit verification (`spec.signatureKeys` in AppProject) |

---

## 3. Clusters

| Command | Explanation |
|---|---|
| `argocd cluster list` | Registered clusters, versions, status |
| `argocd cluster add <KUBECONFIG_CONTEXT> --name prod-a --label tier=workload --yes` | Create SA + ClusterRoleBinding in target, store Secret in Argo CD |
| `argocd cluster add <CTX> --namespace team-a --namespace team-b` | Namespace-scoped registration (Role instead of ClusterRole) |
| `argocd cluster add <CTX> --aws-cluster-name prod-a --aws-role-arn arn:aws:iam::123456789012:role/argocd-deployer` | EKS with IAM auth (exec plugin) instead of a static token |
| `argocd cluster add <CTX> --exec-command ...` | Generic exec-credential plugin (GKE, AKS) |
| `argocd cluster get <NAME_OR_SERVER>` | Details, including connection state and cache stats |
| `argocd cluster set <NAME> --label region=eu-west-1` | Update labels (used by cluster generator) |
| `argocd cluster rotate-auth <SERVER>` | Rotate the bearer token |
| `argocd cluster rm <SERVER>` | Remove (does not delete workloads) |

Declarative cluster Secret:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: cluster-prod-a
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: cluster
    tier: workload
    region: eu-west-1
stringData:
  name: prod-a
  server: https://<EKS_ENDPOINT>.eks.amazonaws.com
  config: |
    {
      "awsAuthConfig": { "clusterName": "prod-a", "roleARN": "arn:aws:iam::123456789012:role/argocd-deployer" },
      "tlsClientConfig": { "insecure": false, "caData": "<BASE64_CA>" }
    }
```

---

## 4. Projects

| Command | Explanation |
|---|---|
| `argocd proj list` | List AppProjects |
| `argocd proj create payments --description "Payments" --dest workload-dev,payments-* --src https://github.com/<ORG>/capstone.git` | Create with initial destination and source |
| `argocd proj add-source payments https://github.com/<ORG>/payments-*` | Allow another source repo (globs OK) |
| `argocd proj add-destination payments workload-prod payments --name` | Allow a destination by cluster name |
| `argocd proj allow-cluster-resource payments "" Namespace` | Permit a cluster-scoped kind |
| `argocd proj deny-namespace-resource payments networking.k8s.io NetworkPolicy` | Blacklist a namespaced kind |
| `argocd proj role create payments developer` | Create a project role |
| `argocd proj role add-policy payments developer --action sync --permission allow --object 'payments/*'` | Attach a policy |
| `argocd proj role add-group payments developer payments-developers` | Bind an OIDC group |
| `argocd proj role create-token payments ci --expires-in 168h` | Issue a project-scoped JWT for CI |
| `argocd proj windows add payments --kind deny --schedule "0 22 * * 5" --duration 58h --applications '*-prod'` | Add a sync window |
| `argocd proj windows list payments` | List windows |
| `argocd proj set payments --orphaned-resources --orphaned-resources-warn` | Enable orphaned resource warnings |
| `argocd proj get payments -o yaml` | Export as YAML (commit it to Git) |

---

## 5. Applications: Create and Inspect

| Command | Explanation |
|---|---|
| `argocd app list` / `-o wide` / `-o name` / `-p payments` / `-l env=dev` | List apps; filter by project or label |
| `argocd app create fastapi-dev --repo <URL> --path gitops/workloads/dev/fastapi-service --dest-name workload-dev --dest-namespace payments --project payments` | Imperative create (prefer committing YAML) |
| `argocd app create --file app.yaml` / `--upsert` | Create from manifest; `--upsert` updates if exists |
| `argocd app create fastapi-dev --repo <URL> --path chart --helm-set image.tag=1.4.2 --values values-dev.yaml --release-name fastapi` | Helm-specific flags |
| `argocd app create ... --kustomize-image ghcr.io/x/api:1.4.2 --nameprefix dev-` | Kustomize-specific flags |
| `argocd app create ... --sync-policy automated --auto-prune --self-heal --sync-option CreateNamespace=true` | Sync policy at create time |
| `argocd app get fastapi-dev` | Status, resources, health, conditions |
| `argocd app get fastapi-dev --refresh` / `--hard-refresh` | Force a refresh before printing |
| `argocd app get fastapi-dev -o yaml` / `-o json` / `-o tree` / `-o tree=detailed` | Alternative views |
| `argocd app get fastapi-dev --show-operation` | Include current/last operation details |
| `argocd app get fastapi-dev --show-params` | Show Helm parameters |
| `argocd app resources fastapi-dev` | Table of managed resources |
| `argocd app resources fastapi-dev --orphaned` | Resources in namespace not managed by any app |
| `argocd app manifests fastapi-dev` | Print rendered target manifests (what Git says) |
| `argocd app manifests fastapi-dev --source live` | Print live manifests |
| `argocd app manifests fastapi-dev --revision <SHA>` | Render a specific revision |
| `argocd app diff fastapi-dev` | Diff live vs target; exit code 1 if different (use in CI) |
| `argocd app diff fastapi-dev --local ./chart --local-repo-root .` | Diff live vs a local uncommitted directory (preview before pushing) |
| `argocd app diff fastapi-dev --revision feature-branch` | Diff live vs another revision |
| `argocd app history fastapi-dev` | Deployment history (ID, date, revision) |
| `argocd app logs fastapi-dev --kind Deployment --name fastapi-service --follow --tail 100` | Stream pod logs via Argo CD (needs `logs` RBAC) |
| `argocd app logs fastapi-dev --container api --since-seconds 300` | Filter by container/time |

---

## 6. Applications: Sync, Rollback, Modify

| Command | Explanation |
|---|---|
| `argocd app sync fastapi-dev` | Sync to target revision |
| `argocd app sync fastapi-dev --prune` | Also delete extra resources |
| `argocd app sync fastapi-dev --dry-run` | Server-side dry run |
| `argocd app sync fastapi-dev --revision v1.4.2` | Sync a specific revision without changing spec (temporary) |
| `argocd app sync fastapi-dev --resource apps:Deployment:fastapi-service` | Sync one resource only |
| `argocd app sync fastapi-dev --force` | Delete and recreate (`kubectl replace --force`); last resort for immutable fields |
| `argocd app sync fastapi-dev --replace` | Use `kubectl replace` instead of apply |
| `argocd app sync fastapi-dev --server-side` | Force server-side apply for this sync |
| `argocd app sync fastapi-dev --apply-out-of-sync-only` | Skip resources already synced |
| `argocd app sync fastapi-dev --strategy hook` / `apply` | Choose whether to run hooks |
| `argocd app sync fastapi-dev --retry-limit 3 --retry-backoff-duration 10s` | Retry on failure |
| `argocd app sync fastapi-dev --async` | Return immediately |
| `argocd app sync fastapi-dev --timeout 300` | Wait up to 300 s |
| `argocd app sync fastapi-dev --preview-changes` | Print diff and prompt before syncing |
| `argocd app sync -l env=dev` | Sync all apps with a label |
| `argocd app sync fastapi-dev --local ./chart` | Sync from a local directory (bypasses Git; **never in prod**, sets `local` revision) |
| `argocd app wait fastapi-dev --sync --health --timeout 600` | Block until Synced and Healthy |
| `argocd app wait fastapi-dev --operation` | Wait for the current operation to finish |
| `argocd app wait fastapi-dev --degraded` | Return when Degraded (useful in test harnesses) |
| `argocd app rollback fastapi-dev <HISTORY_ID>` | Re-apply a previous history entry; disables auto-sync until re-enabled |
| `argocd app terminate-op fastapi-dev` | Abort a stuck sync |
| `argocd app set fastapi-dev --sync-policy automated --auto-prune --self-heal` | Enable automation |
| `argocd app set fastapi-dev --sync-policy none` | Disable automation (freeze) |
| `argocd app set fastapi-dev --helm-set image.tag=1.4.3` | Override a Helm parameter (an *imperative* change; prefer Git) |
| `argocd app set fastapi-dev --revision release/1.5` | Change target branch |
| `argocd app set fastapi-dev --sync-option ServerSideApply=true` | Add a sync option |
| `argocd app unset fastapi-dev --parameter image.tag` | Remove an override |
| `argocd app edit fastapi-dev` | Edit spec in `$EDITOR` |
| `argocd app patch fastapi-dev --patch '{"spec":{"syncPolicy":null}}' --type merge` | JSON merge patch |
| `argocd app patch-resource fastapi-dev --kind Deployment --resource-name fastapi-service --patch '...'` | Patch a live child resource |
| `argocd app actions list fastapi-dev` | Available Lua actions |
| `argocd app actions run fastapi-dev restart --kind Deployment --resource-name fastapi-service` | Run an action |
| `argocd app delete-resource fastapi-dev --kind Pod --resource-name api-xyz --force` | Delete one live resource (it will be recreated if synced) |
| `argocd app delete fastapi-dev` | Delete the Application (cascades if finalizer present) |
| `argocd app delete fastapi-dev --cascade=false` | Delete the Application, keep workloads |
| `argocd app delete fastapi-dev --propagation-policy background` | Choose deletion propagation |
| `argocd app delete fastapi-dev --yes` | Skip confirmation |

### kubectl equivalents

```bash
# Refresh (normal / hard)
kubectl annotate application fastapi-dev -n argocd argocd.argoproj.io/refresh=normal --overwrite
kubectl annotate application fastapi-dev -n argocd argocd.argoproj.io/refresh=hard --overwrite

# Trigger a sync
kubectl patch application fastapi-dev -n argocd --type merge \
  -p '{"operation":{"initiatedBy":{"username":"kubectl"},"sync":{"revision":"HEAD","prune":true}}}'

# Remove the finalizer (Application stuck deleting)
kubectl patch application fastapi-dev -n argocd --type json \
  -p '[{"op":"remove","path":"/metadata/finalizers"}]'

# Read status fields
kubectl get application fastapi-dev -n argocd -o jsonpath='{.status.sync.status}{"\t"}{.status.health.status}{"\n"}'
kubectl get applications -n argocd -o custom-columns='NAME:.metadata.name,SYNC:.status.sync.status,HEALTH:.status.health.status,REV:.status.sync.revision'
```

---

## 7. ApplicationSets

| Command | Explanation |
|---|---|
| `argocd appset list` | List ApplicationSets |
| `argocd appset create appset.yaml` / `--upsert` | Create or update |
| `argocd appset get fastapi-service-envs` / `-o yaml` | Inspect; shows generated apps and conditions |
| `argocd appset generate appset.yaml` | Dry-run: print the Applications the generators would produce (2.11+) |
| `argocd appset delete fastapi-service-envs` | Delete (children deleted unless `preserveResourcesOnDeletion`) |
| `kubectl get applicationset -n argocd fastapi-service-envs -o jsonpath='{.status.conditions}'` | Read error conditions |

---

## 8. Admin and Maintenance

| Command | Explanation |
|---|---|
| `argocd admin initial-password -n argocd` | Print the generated admin password |
| `argocd admin export -n argocd > argocd-backup.yaml` | Dump all Argo CD config (apps, projects, repos, clusters, settings) |
| `argocd admin import -n argocd - < argocd-backup.yaml` | Restore from a dump; `--prune` removes objects not in the dump |
| `argocd admin settings validate --argocd-cm-path argocd-cm.yaml` | Validate ConfigMap syntax |
| `argocd admin settings rbac validate --policy-file policy.csv` | Validate RBAC CSV |
| `argocd admin settings rbac can <SUBJECT> <ACTION> <RESOURCE> <OBJECT> --policy-file policy.csv` | Test a policy decision |
| `argocd admin settings resource-overrides health deployment.yaml --argocd-cm-path argocd-cm.yaml` | Test a Lua health script against a manifest |
| `argocd admin settings resource-overrides ignore-differences ...` | Test ignoreDifferences |
| `argocd admin app generate-spec fastapi-dev --repo ... --path ...` | Emit an Application YAML without creating it |
| `argocd admin app get-reconcile-results` | Dump reconciliation results for performance analysis |
| `argocd admin cluster stats` | Per-cluster resource counts, API calls, shard assignment |
| `argocd admin cluster shards` | Show cluster-to-shard mapping |
| `argocd admin cluster kubeconfig <SERVER> /tmp/kc` | Build a kubeconfig from a cluster Secret (debugging) |
| `argocd admin repo generate-spec <URL> --ssh-private-key-path key` | Emit a repo Secret manifest |
| `argocd admin proj generate-spec payments ...` | Emit an AppProject manifest |
| `argocd admin redis-initial-password -n argocd` | Redis auth password (2.8+) |
| `argocd admin dashboard -n argocd` | Run a local UI against the cluster without the API server (core mode) |
| `argocd admin notifications template notify app-deployed fastapi-dev` | Test a notification template |
| `argocd admin notifications trigger get` | List triggers |

---

## 9. Argo Rollouts kubectl Plugin

Install: `curl -sSL -o kubectl-argo-rollouts https://github.com/argoproj/argo-rollouts/releases/download/v1.7.2/kubectl-argo-rollouts-linux-amd64 && sudo install -m 555 kubectl-argo-rollouts /usr/local/bin/` (or `brew install argoproj/tap/kubectl-argo-rollouts`).

| Command | Explanation |
|---|---|
| `kubectl argo rollouts list rollouts -n payments` | List Rollouts |
| `kubectl argo rollouts get rollout fastapi-service -n payments --watch` | Live step/weight/ReplicaSet view |
| `kubectl argo rollouts status fastapi-service -n payments` | Block until Healthy/Degraded; exit code for CI |
| `kubectl argo rollouts promote fastapi-service -n payments` | Advance past a pause step |
| `kubectl argo rollouts promote fastapi-service --full` | Skip all remaining steps |
| `kubectl argo rollouts abort fastapi-service` | Route all traffic to stable; Rollout becomes Degraded |
| `kubectl argo rollouts retry rollout fastapi-service` | Retry an aborted rollout |
| `kubectl argo rollouts undo fastapi-service --to-revision 3` | Roll back to a revision |
| `kubectl argo rollouts pause fastapi-service` / `resume` | Manual pause/resume |
| `kubectl argo rollouts restart fastapi-service` | Rolling restart of pods |
| `kubectl argo rollouts set image fastapi-service api=ghcr.io/x/api:1.4.3` | Imperative image change (lab only; Git in production) |
| `kubectl argo rollouts get analysisrun -n payments` | Inspect analysis results and metric values |
| `kubectl argo rollouts lint -f rollout.yaml` | Validate a Rollout manifest |
| `kubectl argo rollouts dashboard` | Local web dashboard on :3100 |

---

## 10. Annotations Cheatsheet

```yaml
metadata:
  annotations:
    # Ordering and hooks
    argocd.argoproj.io/sync-wave: "-1"
    argocd.argoproj.io/hook: PreSync                          # PreSync|Sync|PostSync|SyncFail|PostDelete|Skip
    argocd.argoproj.io/hook-delete-policy: HookSucceeded      # HookSucceeded|HookFailed|BeforeHookCreation
    # Per-resource behaviour
    argocd.argoproj.io/sync-options: Prune=false,Delete=false,Validate=false,ServerSideApply=true,Replace=true,Force=true,SkipDryRunOnMissingResource=true
    argocd.argoproj.io/compare-options: IgnoreExtraneous
    # Application-level
    argocd.argoproj.io/manifest-generate-paths: .;../shared   # webhook path filtering (on the Application)
    argocd.argoproj.io/refresh: hard                          # one-shot trigger (on the Application)
    # Notifications (on the Application)
    notifications.argoproj.io/subscribe.on-sync-succeeded.slack: deploys
    notifications.argoproj.io/subscribe.on-health-degraded.github: ""
    # Image Updater (on the Application)
    argocd-image-updater.argoproj.io/image-list: api=ghcr.io/<ORG>/fastapi-service
    argocd-image-updater.argoproj.io/api.update-strategy: semver
    argocd-image-updater.argoproj.io/api.allow-tags: regexp:^v?[0-9]+\.[0-9]+\.[0-9]+$
    argocd-image-updater.argoproj.io/write-back-method: git
    argocd-image-updater.argoproj.io/write-back-target: helmvalues:values-dev.yaml
    argocd-image-updater.argoproj.io/api.helm.image-tag: image.tag
```

---

## 11. Configuration Objects Map

| Object | Namespace `argocd` | Holds |
|---|---|---|
| ConfigMap `argocd-cm` | general settings | `url`, `admin.enabled`, `oidc.config`, `dex.config`, `accounts.*`, `resource.customizations.*`, `resource.exclusions`, `timeout.reconciliation`, `application.resourceTrackingMethod`, `kustomize.buildOptions`, `helm.valuesFileSchemes`, `application.instanceLabelKey` |
| ConfigMap `argocd-rbac-cm` | RBAC | `policy.csv`, `policy.default`, `scopes`, `policy.matchMode` |
| ConfigMap `argocd-cmd-params-cm` | component flags | `server.insecure`, `controller.status.processors`, `controller.operation.processors`, `reposerver.parallelism.limit`, `application.namespaces`, `controller.sharding.algorithm`, `applicationsetcontroller.enable.progressive.syncs` |
| ConfigMap `argocd-tls-certs-cm` | custom CA certs for Git/Helm HTTPS | PEM per host |
| ConfigMap `argocd-ssh-known-hosts-cm` | SSH host keys | `ssh_known_hosts` |
| ConfigMap `argocd-gpg-keys-cm` | GPG public keys | for signed-commit verification |
| ConfigMap `argocd-notifications-cm` | notifications | `service.*`, `template.*`, `trigger.*`, `subscriptions`, `context` |
| Secret `argocd-secret` | server secrets | `admin.password`, `server.secretkey`, `webhook.github.secret`, `oidc.*.clientSecret` |
| Secret `argocd-notifications-secret` | notification creds | `slack-token`, `github-privateKey` |
| Secrets labelled `argocd.argoproj.io/secret-type=repository` | repo creds | `url`, `type`, `sshPrivateKey` / `password` / GitHub App fields |
| Secrets labelled `argocd.argoproj.io/secret-type=repo-creds` | credential templates | `url` prefix + creds |
| Secrets labelled `argocd.argoproj.io/secret-type=cluster` | clusters | `name`, `server`, `config` JSON |

Apply changes with Git (the capstone manages these ConfigMaps via the Helm chart values) and restart the affected component if it does not hot-reload (`argocd-cmd-params-cm` changes require a rollout restart):

```bash
kubectl rollout restart deployment argocd-server argocd-repo-server -n argocd
kubectl rollout restart statefulset argocd-application-controller -n argocd
```

---

## 12. Webhook Setup

Point a GitHub webhook at `https://argocd.example.com/api/webhook` with content type `application/json`, events `push` (and `pull_request` if using the PR generator). Store the shared secret in `argocd-secret` as `webhook.github.secret`. Verify delivery:

```bash
kubectl logs deployment/argocd-server -n argocd | grep -i webhook
```

```text
time="..." level=info msg="Received push event repo: https://github.com/<ORG>/capstone, revision: main, touchedHead: true"
time="..." level=info msg="Refreshing app 'fastapi-dev' for change in cached revision ..."
```

For ApplicationSets, the webhook also goes to the applicationset-controller via the same API server endpoint (2.4+).

---

## 13. One-Liners Worth Memorising

```bash
# Everything that is OutOfSync or unhealthy, across all projects
argocd app list -o json | jq -r '.[] | select(.status.sync.status!="Synced" or .status.health.status!="Healthy") | "\(.metadata.name)\t\(.status.sync.status)\t\(.status.health.status)"'

# Which revision is each prod app on?
argocd app list -l env=prod -o json | jq -r '.[] | "\(.metadata.name)\t\(.status.sync.revision[0:7])"'

# Pause all automation during an incident (then re-enable from Git)
for a in $(argocd app list -l env=prod -o name); do argocd app set "$a" --sync-policy none; done

# Rendered manifest of a single resource
argocd app manifests fastapi-dev | yq 'select(.kind=="Deployment" and .metadata.name=="fastapi-service")'

# CI gate: fail the pipeline if the live cluster is not what Git says
argocd app diff fastapi-dev --exit-code && echo "in sync"

# Check controller queue depth (prometheus metric via port-forward)
kubectl port-forward -n argocd statefulset/argocd-application-controller 8082:8082 &
curl -s localhost:8082/metrics | grep -E 'argocd_app_reconcile_count|workqueue_depth'
```

Next: [04-hands-on-labs.md](04-hands-on-labs.md).
