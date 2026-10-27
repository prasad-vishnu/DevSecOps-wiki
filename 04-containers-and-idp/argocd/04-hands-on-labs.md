# Argo CD: Hands-On Labs

| | |
|---|---|
| **Cluster** | 04 Containers & IDP |
| **Role** | Bridge |
| **Prerequisites** | Argo CD installed on kind per [01-installation-and-setup.md](01-installation-and-setup.md); a GitHub account; the FastAPI chart from [../helm/04-hands-on-labs.md](../helm/04-hands-on-labs.md) or the stand-in chart in Lab 2 |
| **Target version** | Argo CD 2.12+, Argo Rollouts 1.7+, Kubernetes 1.31+ (kind) |
| **Capstone phase** | Phase 3 (Months 9-12) |

> Eight progressive labs. You start with a manual sync of a plain-YAML app, finish with a canary release gated by Prometheus analysis, and along the way build the `gitops/` repository layout that the capstone uses for real. Every lab ends with cleanup so the kind cluster never accumulates junk. Budget roughly eight hours.

Create a GitHub repository named `capstone-gitops` (private is fine) and clone it. Replace `<GITHUB_ORG>` with your user or org throughout. All labs assume `kubectl` context `kind-argocd-lab` and `argocd` logged in.

```bash
export GITOPS_REPO="https://github.com/<GITHUB_ORG>/capstone-gitops.git"
git clone "$GITOPS_REPO" && cd capstone-gitops
```

---

## Lab 1: Manual Sync, Drift and Self-Heal

**Objective.** Deploy a plain-YAML app, break it by hand, watch Argo CD detect drift, then let `selfHeal` fix it.

**Prerequisites.** Argo CD running; repo registered (public repo needs no credentials).

**Time.** 30 minutes.

### Steps

1. Create the manifests.

   ```bash
   mkdir -p lab1/hello
   cat > lab1/hello/deployment.yaml <<'EOF'
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: hello
     labels: { app: hello }
   spec:
     replicas: 2
     selector:
       matchLabels: { app: hello }
     template:
       metadata:
         labels: { app: hello }
       spec:
         securityContext:
           runAsNonRoot: true
           runAsUser: 65532
           seccompProfile: { type: RuntimeDefault }
         containers:
           - name: web
             image: ghcr.io/nginxinc/nginx-unprivileged:1.27-alpine
             ports: [{ containerPort: 8080 }]
             resources:
               requests: { cpu: 10m, memory: 32Mi }
               limits: { memory: 64Mi }
             securityContext:
               allowPrivilegeEscalation: false
               capabilities: { drop: ["ALL"] }
   EOF
   cat > lab1/hello/service.yaml <<'EOF'
   apiVersion: v1
   kind: Service
   metadata:
     name: hello
   spec:
     selector: { app: hello }
     ports: [{ port: 80, targetPort: 8080 }]
   EOF
   git add lab1 && git commit -m "lab1: hello app" && git push
   ```

2. Register the repo (SSH deploy key or GitHub App per the install file) and create the Application with **manual** sync.

   ```bash
   argocd app create hello \
     --repo "$GITOPS_REPO" --path lab1/hello --revision main \
     --dest-server https://kubernetes.default.svc --dest-namespace lab1 \
     --sync-option CreateNamespace=true
   argocd app get hello
   ```

   Expected output (abridged):

   ```text
   Sync Status:        OutOfSync from main (a1b2c3d)
   Health Status:      Missing
   ```

3. Sync and wait.

   ```bash
   argocd app sync hello && argocd app wait hello --health --timeout 120
   ```

   ```text
   Sync Status:        Synced to main (a1b2c3d)
   Health Status:      Healthy
   ```

4. Introduce drift by hand.

   ```bash
   kubectl scale deployment hello -n lab1 --replicas=5
   sleep 5
   argocd app get hello --refresh | grep -E "Sync Status|Deployment"
   ```

   ```text
   Sync Status:        OutOfSync from main (a1b2c3d)
   apps   Deployment  lab1  hello  OutOfSync  Healthy
   ```

5. Inspect the diff.

   ```bash
   argocd app diff hello
   ```

   ```text
   ===== apps/Deployment lab1/hello ======
   ...
   -  replicas: 5
   +  replicas: 2
   ```

6. Enable self-heal and watch the drift disappear within seconds.

   ```bash
   argocd app set hello --sync-policy automated --self-heal
   sleep 10
   kubectl get deployment hello -n lab1 -o jsonpath='{.spec.replicas}{"\n"}'
   ```

   ```text
   2
   ```

### Verify

`argocd app get hello` shows `Synced` and `Healthy`; `argocd app history hello` shows two entries (manual sync, self-heal sync).

### Cleanup

```bash
argocd app delete hello --yes
kubectl delete namespace lab1 --ignore-not-found
```

### What you learned

The difference between sync status and health; how drift surfaces as `OutOfSync`; that `selfHeal` is just automated sync triggered by live-state change.

---

## Lab 2: Helm Source, Values Files and Parameter Overrides

**Objective.** Deploy the FastAPI chart from Git with environment-specific values, override a parameter from the Application, and understand precedence.

**Prerequisites.** Lab 1. If you have not built the FastAPI chart yet, generate a stand-in with `helm create fastapi-service` and set the image to `ghcr.io/nginxinc/nginx-unprivileged:1.27-alpine` with `containerPort: 8080`.

**Time.** 40 minutes.

### Steps

1. Lay out the chart and per-environment values.

   ```bash
   mkdir -p charts gitops/workloads/dev/fastapi-service
   cp -r <PATH_TO_CHART>/fastapi-service charts/
   cat > gitops/workloads/dev/fastapi-service/values.yaml <<'EOF'
   replicaCount: 1
   image:
     repository: ghcr.io/<GITHUB_ORG>/fastapi-service
     tag: "1.0.0"
   resources:
     requests: { cpu: 50m, memory: 128Mi }
     limits: { memory: 256Mi }
   EOF
   git add charts gitops && git commit -m "lab2: fastapi chart + dev values" && git push
   ```

2. Create a multi-source Application: chart from one path, values from another (same repo, but the pattern generalises to OCI chart + Git values).

   ```yaml
   # gitops/apps/fastapi-service-dev.yaml
   apiVersion: argoproj.io/v1alpha1
   kind: Application
   metadata:
     name: fastapi-service-dev
     namespace: argocd
     finalizers: [resources-finalizer.argocd.argoproj.io]
   spec:
     project: default
     sources:
       - repoURL: https://github.com/<GITHUB_ORG>/capstone-gitops.git
         targetRevision: main
         path: charts/fastapi-service
         helm:
           releaseName: fastapi-service
           valueFiles:
             - $values/gitops/workloads/dev/fastapi-service/values.yaml
           parameters:
             - name: replicaCount
               value: "2"
       - repoURL: https://github.com/<GITHUB_ORG>/capstone-gitops.git
         targetRevision: main
         ref: values
     destination:
       server: https://kubernetes.default.svc
       namespace: payments-dev
     syncPolicy:
       automated: { prune: true, selfHeal: true }
       syncOptions: [CreateNamespace=true, ServerSideApply=true]
   ```

   ```bash
   kubectl apply -f gitops/apps/fastapi-service-dev.yaml
   argocd app wait fastapi-service-dev --health --timeout 180
   kubectl get deployment -n payments-dev -o jsonpath='{.items[0].spec.replicas}{"\n"}'
   ```

   ```text
   2
   ```

   `parameters` beat `valueFiles` (the file said 1). Precedence: chart `values.yaml` < `valueFiles` in order < `values`/`valuesObject` < `parameters`.

3. Compare what Argo CD rendered with what `helm template` renders locally.

   ```bash
   argocd app manifests fastapi-service-dev | yq 'select(.kind=="Deployment") | .spec.replicas'
   helm template fastapi-service charts/fastapi-service -f gitops/workloads/dev/fastapi-service/values.yaml --set replicaCount=2 | yq 'select(.kind=="Deployment") | .spec.replicas'
   ```

   Both print `2`. Argo CD is running the same `helm template`.

4. Change the tag in Git and watch the automated sync.

   ```bash
   sed -i 's/tag: "1.0.0"/tag: "1.0.1"/' gitops/workloads/dev/fastapi-service/values.yaml
   git commit -am "dev: bump fastapi-service to 1.0.1" && git push
   argocd app get fastapi-service-dev --refresh | grep "Sync Status"
   ```

   Within one reconciliation period (or immediately if you configured a webhook) the app syncs. The image may not exist, which is fine: the Application goes `Synced` but `Degraded` (`ImagePullBackOff`), demonstrating that sync and health are independent.

### Verify

`argocd app get fastapi-service-dev --show-params` lists `replicaCount=2`; `argocd app history` shows the revision bump.

### Cleanup

Leave this Application in place for Lab 3. (If stopping here: `argocd app delete fastapi-service-dev --yes`.)

### What you learned

Multi-source Applications, Helm value precedence inside Argo CD, and that `helm template` is the whole story.

---

## Lab 3: Sync Waves and a PreSync Migration Hook

**Objective.** Add a Namespace-level ConfigMap at wave -1, a PreSync Job that must succeed before the Deployment rolls, and observe ordering in the UI.

**Prerequisites.** Lab 2.

**Time.** 30 minutes.

### Steps

1. Add the hook and a wave-ordered ConfigMap to the chart templates.

   ```yaml
   # charts/fastapi-service/templates/presync-migrate.yaml
   apiVersion: batch/v1
   kind: Job
   metadata:
     name: {{ include "fastapi-service.fullname" . }}-migrate
     annotations:
       argocd.argoproj.io/hook: PreSync
       argocd.argoproj.io/hook-delete-policy: BeforeHookCreation,HookSucceeded
   spec:
     backoffLimit: 1
     template:
       spec:
         restartPolicy: Never
         securityContext:
           runAsNonRoot: true
           runAsUser: 65532
           seccompProfile: { type: RuntimeDefault }
         containers:
           - name: migrate
             image: busybox:1.36
             command: ["sh", "-c", "echo 'alembic upgrade head (simulated)'; sleep 15; echo done"]
             securityContext:
               allowPrivilegeEscalation: false
               capabilities: { drop: ["ALL"] }
   ```

   ```yaml
   # charts/fastapi-service/templates/config.yaml
   apiVersion: v1
   kind: ConfigMap
   metadata:
     name: {{ include "fastapi-service.fullname" . }}-config
     annotations:
       argocd.argoproj.io/sync-wave: "-1"
   data:
     LOG_LEVEL: {{ .Values.logLevel | default "info" | quote }}
   ```

   ```bash
   git add charts && git commit -m "lab3: presync hook + wave -1 configmap" && git push
   ```

2. Trigger a sync and watch the phases.

   ```bash
   argocd app sync fastapi-service-dev --async
   watch -n 2 'argocd app get fastapi-service-dev --show-operation | sed -n "/GROUP/,\$p"'
   ```

   You will see the Job appear with `HOOK PreSync` and status `Running`, the ConfigMap `Synced` first, and the Deployment only updated after the Job reaches `Succeeded`.

   ```text
   GROUP  KIND        NAMESPACE     NAME                      STATUS     HEALTH   HOOK     MESSAGE
   batch  Job         payments-dev  fastapi-service-migrate   Succeeded  Healthy  PreSync  job.batch/fastapi-service-migrate created
          ConfigMap   payments-dev  fastapi-service-config    Synced              configmap/fastapi-service-config created
   apps   Deployment  payments-dev  fastapi-service           Synced     Healthy           deployment.apps/fastapi-service configured
   ```

3. Make the hook fail and confirm the Deployment is not touched.

   ```bash
   sed -i 's/echo done/exit 1/' charts/fastapi-service/templates/presync-migrate.yaml
   git commit -am "lab3: break migration" && git push
   argocd app sync fastapi-service-dev; echo "exit=$?"
   ```

   ```text
   FATA[0020] Operation has completed with phase: Failed
   exit=20
   ```

   `argocd app get` shows the Job `Failed` and the Deployment still at the previous revision. Revert the change and push.

### Verify

`argocd app history fastapi-service-dev` shows the failed operation; `kubectl get jobs -n payments-dev` shows the hook Job retained (because the delete policy only removes on success).

### Cleanup

```bash
git revert --no-edit HEAD && git push
kubectl delete job -n payments-dev --all
```

### What you learned

How hooks gate a sync, how waves order resources, and that a failed PreSync leaves the previous version running.

---

## Lab 4: App-of-Apps Bootstrap and AppProjects

**Objective.** Build the capstone's `gitops/` layout: a root Application, a platform project, a payments project with restricted destinations, and child Applications discovered from a directory.

**Prerequisites.** Lab 2 (delete `fastapi-service-dev` first; the root app will recreate it).

**Time.** 45 minutes.

### Steps

1. Delete the hand-made Application without cascading.

   ```bash
   argocd app delete fastapi-service-dev --cascade=false --yes
   ```

2. Create projects.

   ```yaml
   # gitops/projects/platform.yaml
   apiVersion: argoproj.io/v1alpha1
   kind: AppProject
   metadata:
     name: platform
     namespace: argocd
   spec:
     description: Cluster add-ons managed by the platform team
     sourceRepos: ["*"]
     destinations:
       - server: "*"
         namespace: "*"
     clusterResourceWhitelist:
       - group: "*"
         kind: "*"
   ---
   # gitops/projects/payments.yaml
   apiVersion: argoproj.io/v1alpha1
   kind: AppProject
   metadata:
     name: payments
     namespace: argocd
   spec:
     description: Payments team workloads
     sourceRepos:
       - https://github.com/<GITHUB_ORG>/capstone-gitops.git
     destinations:
       - server: https://kubernetes.default.svc
         namespace: payments-*
     clusterResourceWhitelist: []
     namespaceResourceBlacklist:
       - group: ""
         kind: ResourceQuota
       - group: networking.k8s.io
         kind: NetworkPolicy
     orphanedResources:
       warn: true
   ```

3. Move the child Application under `gitops/platform/apps/` and set its project to `payments`.

   ```bash
   mkdir -p gitops/platform/apps gitops/platform/projects
   mv gitops/projects/*.yaml gitops/platform/projects/
   git mv gitops/apps/fastapi-service-dev.yaml gitops/platform/apps/
   sed -i 's/project: default/project: payments/' gitops/platform/apps/fastapi-service-dev.yaml
   ```

   Give the projects a negative wave so they exist before Applications reference them (edit both AppProject manifests):

   ```yaml
     annotations:
       argocd.argoproj.io/sync-wave: "-10"
   ```

4. Create the root app.

   ```yaml
   # gitops/bootstrap/root.yaml
   apiVersion: argoproj.io/v1alpha1
   kind: Application
   metadata:
     name: root
     namespace: argocd
   spec:
     project: platform
     source:
       repoURL: https://github.com/<GITHUB_ORG>/capstone-gitops.git
       targetRevision: main
       path: gitops/platform
       directory:
         recurse: true
     destination:
       server: https://kubernetes.default.svc
       namespace: argocd
     syncPolicy:
       automated: { prune: true, selfHeal: true }
   ```

   The root references project `platform`, which is itself defined under `gitops/platform/projects`. Bootstrap it once by applying the project manually; afterwards Git owns it.

   ```bash
   git add gitops && git commit -m "lab4: app-of-apps layout" && git push
   kubectl apply -f gitops/platform/projects/platform.yaml
   kubectl apply -f gitops/bootstrap/root.yaml
   argocd app wait root --sync --timeout 120
   argocd app list
   ```

   ```text
   NAME                        CLUSTER                         NAMESPACE     PROJECT   STATUS  HEALTH   SYNCPOLICY  CONDITIONS
   argocd/fastapi-service-dev  https://kubernetes.default.svc  payments-dev  payments  Synced  Healthy  Auto-Prune  <none>
   argocd/root                 https://kubernetes.default.svc  argocd        platform  Synced  Healthy  Auto-Prune  <none>
   ```

5. Prove the project fence works. Add a second Application that targets `kube-system` under project `payments`, push, and read the condition.

   ```bash
   sed 's/fastapi-service-dev/fastapi-escape/; s/namespace: payments-dev/namespace: kube-system/' \
     gitops/platform/apps/fastapi-service-dev.yaml > gitops/platform/apps/fastapi-escape.yaml
   git add . && git commit -m "lab4: attempt to escape project" && git push
   sleep 60
   argocd app get fastapi-escape | grep -A2 CONDITION
   ```

   ```text
   CONDITION       MESSAGE
   InvalidSpecError  application destination server 'https://kubernetes.default.svc' and namespace 'kube-system' do not match any of the allowed destinations in project 'payments'
   ```

   Remove the file and push.

### Verify

`argocd proj get payments` lists the single allowed destination; the UI shows `root` with children in a tree.

### Cleanup

```bash
git rm gitops/platform/apps/fastapi-escape.yaml && git commit -m "lab4: remove escape" && git push
```

Keep `root` for the following labs.

### What you learned

Bootstrapping a cluster from one Application, project-based tenancy, and how spec violations surface as conditions rather than silent failures.

---

## Lab 5: ApplicationSet Across Environments with a Git Directory Generator

**Objective.** Replace the hand-written child Application with an ApplicationSet that creates one Application per environment directory, with prod set to manual sync.

**Prerequisites.** Lab 4.

**Time.** 40 minutes.

### Steps

1. Create stage and prod value directories.

   ```bash
   for env in stage prod; do
     mkdir -p gitops/workloads/$env/fastapi-service
     sed "s/replicaCount: 1/replicaCount: 2/" gitops/workloads/dev/fastapi-service/values.yaml \
       > gitops/workloads/$env/fastapi-service/values.yaml
   done
   ```

2. Write the ApplicationSet; delete the single Application manifest.

   ```yaml
   # gitops/platform/apps/fastapi-service.appset.yaml
   apiVersion: argoproj.io/v1alpha1
   kind: ApplicationSet
   metadata:
     name: fastapi-service
     namespace: argocd
   spec:
     goTemplate: true
     goTemplateOptions: ["missingkey=error"]
     generators:
       - git:
           repoURL: https://github.com/<GITHUB_ORG>/capstone-gitops.git
           revision: main
           directories:
             - path: gitops/workloads/*/fastapi-service
     template:
       metadata:
         name: 'fastapi-service-{{ index .path.segments 2 }}'
         labels:
           env: '{{ index .path.segments 2 }}'
           team: payments
       spec:
         project: payments
         sources:
           - repoURL: https://github.com/<GITHUB_ORG>/capstone-gitops.git
             targetRevision: main
             path: charts/fastapi-service
             helm:
               releaseName: fastapi-service
               valueFiles:
                 - '$values/{{ .path.path }}/values.yaml'
           - repoURL: https://github.com/<GITHUB_ORG>/capstone-gitops.git
             targetRevision: main
             ref: values
         destination:
           server: https://kubernetes.default.svc
           namespace: 'payments-{{ index .path.segments 2 }}'
         syncPolicy:
           syncOptions: [CreateNamespace=true, ServerSideApply=true]
           {{- if ne (index .path.segments 2) "prod" }}
           automated: { prune: true, selfHeal: true }
           {{- end }}
   ```

   `.path.segments` for `gitops/workloads/dev/fastapi-service` is `[gitops, workloads, dev, fastapi-service]`, so index 2 is the environment.

   ```bash
   git rm gitops/platform/apps/fastapi-service-dev.yaml
   git add . && git commit -m "lab5: applicationset per environment" && git push
   argocd app wait root --sync --timeout 120
   argocd app list -l team=payments
   ```

   ```text
   NAME                          ...  PROJECT   STATUS     HEALTH   SYNCPOLICY
   argocd/fastapi-service-dev         payments  Synced     Healthy  Auto-Prune
   argocd/fastapi-service-prod        payments  OutOfSync  Missing  Manual
   argocd/fastapi-service-stage       payments  Synced     Healthy  Auto-Prune
   ```

   The root app pruned the old Application (same name, now owned by the ApplicationSet) and the ApplicationSet recreated it.

3. Dry-run the generator before future changes.

   ```bash
   argocd appset generate gitops/platform/apps/fastapi-service.appset.yaml -o yaml | yq '.[].metadata.name'
   ```

   ```text
   fastapi-service-dev
   fastapi-service-prod
   fastapi-service-stage
   ```

4. Promote dev to stage by copying the tag, the way the capstone's promotion workflow does.

   ```bash
   tag=$(yq '.image.tag' gitops/workloads/dev/fastapi-service/values.yaml)
   yq -i ".image.tag = \"$tag\"" gitops/workloads/stage/fastapi-service/values.yaml
   git commit -am "promote fastapi-service $tag to stage" && git push
   ```

### Verify

Three Applications exist; prod requires `argocd app sync fastapi-service-prod` to deploy. Deleting a directory in Git deletes the corresponding Application (try it with stage, then restore).

### Cleanup

Keep for Lab 6 and 7.

### What you learned

Generator-driven fan-out, Go-template conditionals to vary policy per environment, and promotion as a Git commit.

---

## Lab 6: Notifications to GitHub Commit Status

**Objective.** Make every successful sync post a commit status to the GitOps repo so PRs show "deployed to dev".

**Prerequisites.** Lab 5; a GitHub App with `statuses: write` (or a fine-grained PAT for the lab only).

**Time.** 30 minutes.

### Steps

1. Store the credential in the notifications Secret (lab shortcut; the capstone uses ESO).

   ```bash
   kubectl create secret generic argocd-notifications-secret -n argocd \
     --from-literal=github-token='<FINE_GRAINED_PAT>' --dry-run=client -o yaml | kubectl apply -f -
   ```

2. Configure service, template and trigger via Helm values (or patch the ConfigMap).

   ```yaml
   # add to argocd-values-lab.yaml
   notifications:
     enabled: true
     notifiers:
       service.github: |
         token: $github-token
     templates:
       template.app-deployed: |
         message: Application {{.app.metadata.name}} synced to {{.app.status.sync.revision | trunc 7}}.
         github:
           repoURLPath: "{{ (index .app.spec.sources 1).repoURL }}"
           revisionPath: "{{.app.status.sync.revision}}"
           status:
             state: success
             label: "argocd/{{.app.metadata.name}}"
             targetURL: "{{.context.argocdUrl}}/applications/{{.app.metadata.name}}"
     triggers:
       trigger.on-deployed: |
         - when: app.status.operationState.phase in ['Succeeded'] and app.status.health.status == 'Healthy'
           oncePer: app.status.operationState.syncResult.revision
           send: [app-deployed]
   ```

   ```bash
   helm upgrade argocd argo/argo-cd -n argocd --version 7.7.0 -f argocd-values-lab.yaml --wait
   ```

3. Subscribe the ApplicationSet template and push a change.

   ```yaml
       metadata:
         annotations:
           notifications.argoproj.io/subscribe.on-deployed.github: ""
   ```

   ```bash
   git commit -am "lab6: subscribe to notifications" && git push
   ```

4. Observe.

   ```bash
   kubectl logs deployment/argocd-notifications-controller -n argocd --tail 20
   ```

   ```text
   level=info msg="Sending notification" app=fastapi-service-dev trigger=on-deployed service=github
   ```

   In GitHub, the commit shows a green check labelled `argocd/fastapi-service-dev`.

### Verify

`argocd admin notifications template notify app-deployed fastapi-service-dev -n argocd` renders the message without error.

### Cleanup

```bash
kubectl delete secret argocd-notifications-secret -n argocd
```

### What you learned

Triggers, templates and services; `oncePer` to avoid duplicate sends; feeding deployment state back to Git where developers already look.

---

## Lab 7: Second Cluster, Cluster Generator and Sync Windows

**Objective.** Register a second kind cluster, deploy a platform add-on to every cluster with a cluster generator, and block prod syncs over the weekend.

**Prerequisites.** Lab 5; Docker has memory for two kind clusters (about 4 GB).

**Time.** 45 minutes.

### Steps

1. Create and register the cluster on a shared Docker network.

   ```bash
   kind create cluster --name workload-prod --image kindest/node:v1.31.2
   kubectl config use-context kind-argocd-lab
   # kind clusters share the "kind" docker network, so the control-plane container name resolves
   argocd cluster add kind-workload-prod --name workload-prod --label tier=workload --label env=prod --yes
   kubectl -n argocd get secret -l argocd.argoproj.io/secret-type=cluster -o jsonpath='{.items[0].data.server}' | base64 -d; echo
   ```

   If the server shows `https://127.0.0.1:<port>`, patch it to `https://workload-prod-control-plane:6443`:

   ```bash
   kubectl -n argocd patch secret cluster-127.0.0.1-<PORT> --type merge \
     -p "{\"stringData\":{\"server\":\"https://workload-prod-control-plane:6443\"}}"
   argocd cluster list
   ```

   Both clusters should read `Successful`.

2. Deploy metrics-server to every cluster labelled `tier=workload` plus in-cluster.

   ```yaml
   # gitops/platform/addons/metrics-server.appset.yaml
   apiVersion: argoproj.io/v1alpha1
   kind: ApplicationSet
   metadata:
     name: metrics-server
     namespace: argocd
   spec:
     goTemplate: true
     generators:
       - clusters: {}                      # all registered clusters, including in-cluster
     template:
       metadata:
         name: 'metrics-server-{{ .name }}'
       spec:
         project: platform
         source:
           repoURL: https://kubernetes-sigs.github.io/metrics-server/
           chart: metrics-server
           targetRevision: 3.12.x
           helm:
             values: |
               args: ["--kubelet-insecure-tls"]   # kind only; never in production
         destination:
           server: '{{ .server }}'
           namespace: kube-system
         syncPolicy:
           automated: { prune: true, selfHeal: true }
   ```

   ```bash
   git add . && git commit -m "lab7: metrics-server everywhere" && git push
   argocd app wait root --sync && sleep 30 && argocd app list | grep metrics-server
   ```

   ```text
   argocd/metrics-server-in-cluster     https://kubernetes.default.svc           kube-system  platform  Synced  Healthy
   argocd/metrics-server-workload-prod  https://workload-prod-control-plane:6443 kube-system  platform  Synced  Healthy
   ```

3. Add a deny sync window to the payments project for `*-prod` apps and test it.

   ```bash
   argocd proj windows add payments --kind deny --schedule "* * * * *" --duration 1h --applications '*-prod' --manual-sync=false
   argocd app sync fastapi-service-prod
   ```

   ```text
   FATA[0000] rpc error: code = PermissionDenied desc = cannot sync: blocked by sync window
   ```

   Replace the always-on window with the real weekend schedule in `gitops/platform/projects/payments.yaml`:

   ```yaml
     syncWindows:
       - kind: deny
         schedule: "0 22 * * 5"
         duration: 58h
         applications: ["*-prod"]
         manualSync: true
         timeZone: Europe/London
   ```

   ```bash
   argocd proj windows delete payments 0
   git commit -am "lab7: weekend freeze" && git push
   ```

### Verify

`argocd proj windows list payments` shows the deny window; `argocd app get fastapi-service-prod | grep SyncWindow` prints `Sync Allowed` on a weekday.

### Cleanup

```bash
argocd cluster rm https://workload-prod-control-plane:6443
kind delete cluster --name workload-prod
git rm gitops/platform/addons/metrics-server.appset.yaml && git commit -m "lab7: cleanup" && git push
```

### What you learned

Hub-and-spoke multi-cluster, cluster generators for add-ons, and sync windows as a change-freeze primitive.

---

## Lab 8: Canary Release with Argo Rollouts and Prometheus Analysis

**Objective.** Convert the FastAPI Deployment to a Rollout, deploy a canary, and let an AnalysisTemplate querying Prometheus decide whether to promote.

**Prerequisites.** Lab 5; ingress-nginx installed; about 60 minutes.

**Time.** 60 minutes.

### Steps

1. Install Argo Rollouts and kube-prometheus-stack through Git (platform project).

   ```yaml
   # gitops/platform/addons/argo-rollouts.yaml
   apiVersion: argoproj.io/v1alpha1
   kind: Application
   metadata:
     name: argo-rollouts
     namespace: argocd
     annotations: { argocd.argoproj.io/sync-wave: "-5" }
   spec:
     project: platform
     source:
       repoURL: https://argoproj.github.io/argo-helm
       chart: argo-rollouts
       targetRevision: 2.37.x
       helm:
         values: |
           dashboard: { enabled: true }
     destination: { server: https://kubernetes.default.svc, namespace: argo-rollouts }
     syncPolicy:
       automated: { prune: true, selfHeal: true }
       syncOptions: [CreateNamespace=true, ServerSideApply=true]
   ---
   # gitops/platform/addons/kube-prometheus-stack.yaml
   apiVersion: argoproj.io/v1alpha1
   kind: Application
   metadata:
     name: kube-prometheus-stack
     namespace: argocd
     annotations: { argocd.argoproj.io/sync-wave: "-5" }
   spec:
     project: platform
     source:
       repoURL: https://prometheus-community.github.io/helm-charts
       chart: kube-prometheus-stack
       targetRevision: 65.x
       helm:
         values: |
           grafana: { enabled: false }
           alertmanager: { enabled: false }
           prometheus:
             prometheusSpec:
               serviceMonitorSelectorNilUsesHelmValues: false
               retention: 2h
     destination: { server: https://kubernetes.default.svc, namespace: monitoring }
     syncPolicy:
       automated: { prune: true, selfHeal: true }
       syncOptions: [CreateNamespace=true, ServerSideApply=true]
   ```

   ```bash
   git add . && git commit -m "lab8: rollouts + prometheus" && git push
   argocd app wait argo-rollouts kube-prometheus-stack --health --timeout 600
   ```

   `ServerSideApply=true` matters here: the Prometheus CRDs exceed the client-side annotation limit.

2. Switch the chart's Deployment to a Rollout behind a values toggle. In `charts/fastapi-service/templates/deployment.yaml` wrap the existing content:

   ```yaml
   {{- if not .Values.rollout.enabled }}
   ... existing Deployment ...
   {{- end }}
   ```

   Add `templates/rollout.yaml`:

   ```yaml
   {{- if .Values.rollout.enabled }}
   apiVersion: argoproj.io/v1alpha1
   kind: Rollout
   metadata:
     name: {{ include "fastapi-service.fullname" . }}
     labels: {{- include "fastapi-service.labels" . | nindent 4 }}
   spec:
     replicas: {{ .Values.replicaCount }}
     selector:
       matchLabels: {{- include "fastapi-service.selectorLabels" . | nindent 6 }}
     template:
       metadata:
         labels: {{- include "fastapi-service.selectorLabels" . | nindent 8 }}
       spec:
         securityContext: {{- toYaml .Values.podSecurityContext | nindent 8 }}
         containers:
           - name: api
             image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
             ports: [{ name: http, containerPort: 8080 }]
             securityContext: {{- toYaml .Values.securityContext | nindent 12 }}
             resources: {{- toYaml .Values.resources | nindent 12 }}
     strategy:
       canary:
         canaryService: {{ include "fastapi-service.fullname" . }}-canary
         stableService: {{ include "fastapi-service.fullname" . }}
         trafficRouting:
           nginx:
             stableIngress: {{ include "fastapi-service.fullname" . }}
         analysis:
           templates:
             - templateName: {{ include "fastapi-service.fullname" . }}-success-rate
           startingStep: 1
         steps:
           {{- toYaml .Values.rollout.steps | nindent 8 }}
   ---
   apiVersion: v1
   kind: Service
   metadata:
     name: {{ include "fastapi-service.fullname" . }}-canary
   spec:
     selector: {{- include "fastapi-service.selectorLabels" . | nindent 4 }}
     ports: [{ name: http, port: 80, targetPort: http }]
   ---
   apiVersion: argoproj.io/v1alpha1
   kind: AnalysisTemplate
   metadata:
     name: {{ include "fastapi-service.fullname" . }}-success-rate
   spec:
     metrics:
       - name: success-rate
         interval: 20s
         count: 6
         successCondition: result[0] >= 0.95
         failureLimit: 1
         provider:
           prometheus:
             address: http://kube-prometheus-stack-prometheus.monitoring.svc:9090
             query: |
               sum(rate(nginx_ingress_controller_requests{ingress="{{ include "fastapi-service.fullname" . }}",canary="true",status!~"5.."}[1m]))
               /
               clamp_min(sum(rate(nginx_ingress_controller_requests{ingress="{{ include "fastapi-service.fullname" . }}",canary="true"}[1m])), 0.001)
   {{- end }}
   ```

   Add to `values.yaml`:

   ```yaml
   rollout:
     enabled: false
     steps:
       - setWeight: 20
       - pause: { duration: 1m }
       - setWeight: 60
       - pause: { duration: 1m }
   ingress:
     enabled: true
     className: nginx
     host: fastapi-dev.localtest.me
   ```

   Enable it for dev only (`gitops/workloads/dev/fastapi-service/values.yaml`: `rollout: { enabled: true }`), ensure the chart's Ingress targets the stable Service, and ensure ingress-nginx exposes metrics (`controller.metrics.enabled=true` plus a ServiceMonitor) so the query returns data.

   ```bash
   git add . && git commit -m "lab8: rollout strategy for dev" && git push
   argocd app wait fastapi-service-dev --health --timeout 300
   kubectl argo rollouts get rollout fastapi-service -n payments-dev
   ```

   ```text
   Name:            fastapi-service
   Namespace:       payments-dev
   Status:          ✔ Healthy
   Strategy:        Canary
     Step:          4/4
     SetWeight:     100
     ActualWeight:  100
   Images:          ghcr.io/<GITHUB_ORG>/fastapi-service:1.0.1 (stable)
   ```

3. Generate steady traffic in a second terminal, then release a new tag.

   ```bash
   while true; do curl -s -o /dev/null http://fastapi-dev.localtest.me/healthz; sleep 0.2; done &
   yq -i '.image.tag = "1.0.2"' gitops/workloads/dev/fastapi-service/values.yaml
   git commit -am "dev: fastapi-service 1.0.2 canary" && git push
   kubectl argo rollouts get rollout fastapi-service -n payments-dev --watch
   ```

   ```text
   Status:          ◌ Progressing
   Strategy:        Canary
     Step:          1/4
     SetWeight:     20
     ActualWeight:  20
   ...
   NAME                                          KIND         STATUS     AGE  INFO
   ⟳ fastapi-service                             Rollout      ◌ Progressing
   ├──# revision:2
   │  ├──⧉ fastapi-service-7d9f8b6c5d            ReplicaSet   ✔ Healthy  40s  canary
   │  └──α fastapi-service-7d9f8b6c5d-2-1        AnalysisRun  ◌ Running  30s  ✔ 2
   └──# revision:1
      └──⧉ fastapi-service-5c8d7f9b4e            ReplicaSet   ✔ Healthy  12m  stable
   ```

   Argo CD shows the Rollout as `Progressing`, then `Healthy` once the final step completes. In the Argo CD UI the Rollout resource exposes `promote-full` and `abort` actions.

4. Force a failure. Point the image at a tag that returns 500s on `/healthz` (or temporarily change `successCondition` to `result[0] >= 1.01`), push, and watch the AnalysisRun fail and the Rollout abort back to stable with health `Degraded`.

   ```bash
   kubectl argo rollouts get analysisrun -n payments-dev
   ```

   ```text
   NAME                           STATUS   AGE
   fastapi-service-7d9f8b6c5d-3-1 Failed   2m
   ```

   Revert the commit; the Rollout retries with the stable image and returns to `Healthy`.

### Verify

`argocd app get fastapi-service-dev` reflects Rollout health; `kubectl argo rollouts status fastapi-service -n payments-dev` exits 0 after promotion and non-zero when Degraded.

### Cleanup

```bash
kill %1 2>/dev/null                             # stop the curl loop
argocd app delete root --yes                    # cascades to everything Git-managed
kubectl delete namespace payments-dev payments-stage payments-prod monitoring argo-rollouts --ignore-not-found
# Optional: tear down the lab cluster entirely
kind delete cluster --name argocd-lab
```

### What you learned

Progressive delivery as a controller loop one level above Argo CD: Argo CD syncs the Rollout spec, Argo Rollouts drives traffic and consults Prometheus, and the result rolls back up into Argo CD's health view.

---

## Where to Go Next

- Wire the GitHub Actions workflow from [../../06-cicd/github-actions/](../../06-cicd/github-actions/README.md) to open the promotion PR automatically.
- Replace the PAT in Lab 6 with an ExternalSecret backed by Vault, as described in [05-production-patterns.md](05-production-patterns.md).
- Surface these Applications in Backstage with the Argo CD plugin ([../backstage/README.md](../backstage/README.md)).
