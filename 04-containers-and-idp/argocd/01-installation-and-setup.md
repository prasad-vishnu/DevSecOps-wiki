# Argo CD: Installation and Setup

| | |
|---|---|
| **Cluster** | 04 Containers & IDP |
| **Role** | Bridge |
| **Prerequisites** | A Kubernetes 1.31+ cluster (kind for labs), `kubectl`, `helm` 3.15+, a GitHub account for repository access |
| **Target version** | Argo CD 2.12+ |
| **Capstone phase** | Phase 3 (Months 9-12) |

> After this file you will have Argo CD running on a local kind cluster, the `argocd` CLI installed and logged in, the UI reachable, a private Git repository registered using a deploy key, and your first Application synced and healthy. You will also know the two production installation routes (Helm chart and Kustomize) and how to connect Argo CD to a remote cluster.

---

## 1. Installation Options

| Method | When to use | Pros | Cons |
|---|---|---|---|
| `kubectl apply` of upstream `install.yaml` | Quick labs, evaluation | One command, exactly what upstream tests | Hard to customise, HA variant is a separate file |
| Helm chart `argo/argo-cd` | Production, Terraform-driven installs | Values-driven, HA toggles, well-maintained by the community | Chart versions lag upstream by days; values schema is large |
| Kustomize overlay on upstream manifests | Teams already standardised on Kustomize | Patch any field, pin to upstream tag | You maintain the patches |
| Managed (Akuity, Codefresh, Red Hat OpenShift GitOps) | Enterprises wanting SLAs | Control plane hosted and upgraded for you | Cost, vendor lock-in for agent architecture |

The labs in this folder use the Helm chart because that is how Terraform installs it in the capstone.

---

## 2. Local Cluster with kind

Create a kind cluster with a port mapping so an Ingress can be reached on `localhost`.

```yaml
# kind-argocd.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: argocd-lab
nodes:
  - role: control-plane
    kubeadmConfigPatches:
      - |
        kind: InitConfiguration
        nodeRegistration:
          kubeletExtraArgs:
            node-labels: "ingress-ready=true"
    extraPortMappings:
      - containerPort: 80
        hostPort: 80
        protocol: TCP
      - containerPort: 443
        hostPort: 443
        protocol: TCP
  - role: worker
  - role: worker
```

```bash
kind create cluster --config kind-argocd.yaml --image kindest/node:v1.31.2
kubectl cluster-info --context kind-argocd-lab
```

Expected output:

```text
Kubernetes control plane is running at https://127.0.0.1:XXXXX
CoreDNS is running at https://127.0.0.1:XXXXX/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
```

Install ingress-nginx so the Argo CD UI is reachable without port-forwarding (optional; port-forward works too).

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
kubectl wait --namespace ingress-nginx --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller --timeout=180s
```

---

## 3. Install Argo CD with the Helm Chart

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
```

A minimal values file suitable for a lab. Note the explicit non-HA choices; section 8 shows the production values.

```yaml
# argocd-values-lab.yaml
global:
  domain: argocd.localtest.me        # resolves to 127.0.0.1 publicly

configs:
  params:
    server.insecure: true            # TLS terminated at ingress in the lab; see section 8 for production
  cm:
    timeout.reconciliation: 180s
    application.resourceTrackingMethod: annotation
    kustomize.buildOptions: "--enable-helm"
  rbac:
    policy.default: role:readonly

server:
  ingress:
    enabled: true
    ingressClassName: nginx
    annotations:
      nginx.ingress.kubernetes.io/backend-protocol: "HTTP"

redis:
  enabled: true

dex:
  enabled: false                     # enabled later when SSO is configured

notifications:
  enabled: true

applicationSet:
  enabled: true
```

```bash
kubectl create namespace argocd
helm install argocd argo/argo-cd \
  --namespace argocd \
  --version 7.7.0 \
  --values argocd-values-lab.yaml \
  --wait
```

Expected output (abridged):

```text
NAME: argocd
NAMESPACE: argocd
STATUS: deployed
REVISION: 1
NOTES:
In order to access the server UI you have the following options: ...
```

Verify every component is running.

```bash
kubectl get pods --namespace argocd
```

```text
NAME                                                READY   STATUS    RESTARTS   AGE
argocd-application-controller-0                     1/1     Running   0          2m
argocd-applicationset-controller-5d6f8c6b7c-h2k9x   1/1     Running   0          2m
argocd-notifications-controller-7c8d9f5b4d-tq2ms    1/1     Running   0          2m
argocd-redis-6c7b5d8f9d-xw8rb                       1/1     Running   0          2m
argocd-repo-server-84f6b7c9d8-l4mzp                 1/1     Running   0          2m
argocd-server-6d9f7b8c5f-p7vqn                      1/1     Running   0          2m
```

### Alternative: upstream manifests

```bash
kubectl create namespace argocd
kubectl apply --namespace argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/v2.12.6/manifests/install.yaml
# HA variant:
# -f https://raw.githubusercontent.com/argoproj/argo-cd/v2.12.6/manifests/ha/install.yaml
```

---

## 4. Install the `argocd` CLI

| Platform | Command |
|---|---|
| Linux (amd64) | `curl -sSL -o argocd https://github.com/argoproj/argo-cd/releases/download/v2.12.6/argocd-linux-amd64 && sudo install -m 555 argocd /usr/local/bin/argocd && rm argocd` |
| macOS | `brew install argocd` |
| Windows | `winget install ArgoProj.ArgoCD` or `choco install argocd-cli` |
| Container | `docker run --rm quay.io/argoproj/argocd:v2.12.6 argocd version --client` |

Verify the binary and the version hashes of the download if your organisation requires it (release notes publish `cli_checksums.txt`).

```bash
argocd version --client
```

```text
argocd: v2.12.6+4dab5bd
  BuildDate: 2024-11-12T16:49:13Z
  GitCommit: 4dab5bd6a60adea12d8c6a2d8f8a3c3a1b2c3d4e
  GitTreeState: clean
  GoVersion: go1.22.9
  Compiler: gc
  Platform: linux/amd64
```

---

## 5. First Login

The chart generates a random password for the built-in `admin` user and stores it in a Secret.

```bash
argocd admin initial-password --namespace argocd
```

```text
Xy7bQ2mK9pLw3RtV

 This password must be only used for first time login. We strongly recommend you update the password using `argocd account update-password`.
```

Log in through the ingress (or `kubectl port-forward svc/argocd-server -n argocd 8080:443` and use `localhost:8080`).

```bash
argocd login argocd.localtest.me --username admin --password "$(argocd admin initial-password -n argocd | head -1)" --insecure --grpc-web
argocd account update-password
```

The `--grpc-web` flag is needed when an ingress controller does not pass HTTP/2 through. Change the password immediately, then delete the initial Secret so it is not left around.

```bash
kubectl delete secret argocd-initial-admin-secret --namespace argocd
```

Open the UI at `http://argocd.localtest.me`. In production the `admin` account is disabled entirely (`admin.enabled: "false"` in `argocd-cm`) once SSO works.

---

## 6. Register a Git Repository

### 6.1 Public HTTPS repo (no credentials)

```bash
argocd repo add https://github.com/argoproj/argocd-example-apps.git
```

### 6.2 Private repo via SSH deploy key (recommended for labs)

Generate a key *dedicated to Argo CD*, add the public half as a read-only deploy key on the GitHub repository, and register the private half.

```bash
ssh-keygen -t ed25519 -C "argocd-deploy-key" -f ./argocd-deploy-key -N ""
cat argocd-deploy-key.pub       # paste into GitHub: Settings > Deploy keys (read-only)

argocd repo add git@github.com:<GITHUB_ORG>/<REPO>.git \
  --ssh-private-key-path ./argocd-deploy-key \
  --name capstone-gitops
rm argocd-deploy-key          # Argo CD now holds it in a Secret; do not keep it on disk
```

The declarative equivalent, which is what the capstone commits (the private key is injected by External Secrets Operator, never committed):

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: repo-capstone-gitops
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository
stringData:
  type: git
  name: capstone-gitops
  url: git@github.com:<GITHUB_ORG>/<REPO>.git
  sshPrivateKey: |
    <INJECTED_BY_EXTERNAL_SECRETS_OPERATOR>
```

### 6.3 GitHub App (recommended for production)

A GitHub App gives short-lived installation tokens, fine-grained repo scope, and one credential for a whole organisation.

```bash
argocd repocreds add https://github.com/<GITHUB_ORG>/ \
  --github-app-id <APP_ID> \
  --github-app-installation-id <INSTALLATION_ID> \
  --github-app-private-key-path ./github-app.pem
```

Any repository URL beginning with the prefix now authenticates automatically.

Verify connectivity.

```bash
argocd repo list
```

```text
TYPE  NAME             REPO                                              INSECURE  OCI    LFS    CREDS  STATUS      MESSAGE  PROJECT
git   capstone-gitops  git@github.com:<GITHUB_ORG>/<REPO>.git            false     false  false  true   Successful
```

---

## 7. Your First Application

```yaml
# guestbook-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: guestbook
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io   # delete cluster resources when the Application is deleted
spec:
  project: default
  source:
    repoURL: https://github.com/argoproj/argocd-example-apps.git
    targetRevision: HEAD
    path: guestbook
  destination:
    server: https://kubernetes.default.svc
    namespace: guestbook
  syncPolicy:
    syncOptions:
      - CreateNamespace=true
```

```bash
kubectl apply -f guestbook-app.yaml
argocd app get guestbook
```

```text
Name:               argocd/guestbook
Project:            default
Server:             https://kubernetes.default.svc
Namespace:          guestbook
URL:                http://argocd.localtest.me/applications/guestbook
Source:
- Repo:             https://github.com/argoproj/argocd-example-apps.git
  Target:           HEAD
  Path:             guestbook
SyncWindow:         Sync Allowed
Sync Policy:        Manual
Sync Status:        OutOfSync from HEAD (d7927a2)
Health Status:      Missing

GROUP  KIND        NAMESPACE  NAME          STATUS     HEALTH   HOOK  MESSAGE
       Service     guestbook  guestbook-ui  OutOfSync  Missing
apps   Deployment  guestbook  guestbook-ui  OutOfSync  Missing
```

The Application is `OutOfSync` and `Missing` because sync policy is manual. Sync it.

```bash
argocd app sync guestbook
argocd app wait guestbook --health --timeout 120
```

```text
...
Sync Status:        Synced to HEAD (d7927a2)
Health Status:      Healthy
```

---

## 8. Production Values (Reference)

The differences from the lab values that matter most.

```yaml
# argocd-values-prod.yaml (excerpt)
global:
  domain: argocd.example.com

redis-ha:
  enabled: true                       # 3 Redis + HAProxy
redis:
  enabled: false

controller:
  replicas: 2                         # sharded by cluster
  env:
    - name: ARGOCD_CONTROLLER_SHARDING_ALGORITHM
      value: round-robin
  resources:
    requests: { cpu: "1", memory: 2Gi }
    limits:   { memory: 4Gi }

repoServer:
  replicas: 3
  autoscaling:
    enabled: true
    minReplicas: 3
    maxReplicas: 8
  resources:
    requests: { cpu: 500m, memory: 1Gi }
    limits:   { memory: 3Gi }

server:
  replicas: 3
  autoscaling:
    enabled: true
  ingress:
    enabled: true
    ingressClassName: nginx
    annotations:
      cert-manager.io/cluster-issuer: letsencrypt-prod
      nginx.ingress.kubernetes.io/backend-protocol: "GRPC"   # pass through to argocd-server TLS
    tls: true

configs:
  params:
    server.insecure: false
  cm:
    admin.enabled: "false"
    url: https://argocd.example.com
    oidc.config: |
      name: Entra ID
      issuer: https://login.microsoftonline.com/<TENANT_ID>/v2.0
      clientID: <CLIENT_ID>
      clientSecret: $oidc.entra.clientSecret
      requestedScopes: ["openid", "profile", "email", "groups"]
  rbac:
    policy.default: ""
    policy.csv: |
      g, <PLATFORM_TEAM_GROUP_ID>, role:admin
```

`$oidc.entra.clientSecret` references a key in the `argocd-secret` Secret, which ESO populates from Vault or a cloud secret manager.

---

## 9. Connect a Remote Cluster

Argo CD manages other clusters by storing a kubeconfig-like Secret labelled `argocd.argoproj.io/secret-type: cluster`. The CLI shortcut creates a ServiceAccount with cluster-admin in the target cluster and copies its token.

```bash
kind create cluster --name workload-dev --image kindest/node:v1.31.2
kubectl config use-context kind-argocd-lab
argocd cluster add kind-workload-dev --name workload-dev --yes
```

```text
INFO[0000] ServiceAccount "argocd-manager" created in namespace "kube-system"
INFO[0000] ClusterRole "argocd-manager-role" created
INFO[0000] ClusterRoleBinding "argocd-manager-role-binding" created
Cluster 'https://workload-dev-control-plane:6443' added
```

Note for kind: the API endpoint stored in the kubeconfig is `127.0.0.1:<port>`, unreachable from inside the Argo CD cluster. Replace it with the Docker network address (`https://workload-dev-control-plane:6443`) and connect both kind clusters to the same Docker network, or use `--kubeconfig` with a corrected server URL. In the cloud, prefer workload identity (IRSA/EKS Pod Identity, Workload Identity Federation, Managed Identity) over copied tokens; see [05-production-patterns.md](05-production-patterns.md).

```bash
argocd cluster list
```

```text
SERVER                                   NAME          VERSION  STATUS      MESSAGE  PROJECT
https://workload-dev-control-plane:6443  workload-dev  1.31     Successful
https://kubernetes.default.svc           in-cluster               Unknown     Cluster has no applications and is not being monitored.
```

---

## 10. Verification Checklist

```bash
argocd version                         # server and client versions match major.minor
argocd account get-user-info           # shows logged-in user and groups
argocd repo list                       # STATUS Successful
argocd cluster list                    # STATUS Successful for every cluster with apps
argocd app list                        # guestbook Synced / Healthy
kubectl get crd | grep argoproj        # applications, applicationsets, appprojects
```

Enable shell completion so the rest of this folder is faster to type.

```bash
source <(argocd completion bash)       # or zsh / fish / powershell
```

---

## 11. Uninstall / Cleanup

```bash
argocd app delete guestbook --yes
helm uninstall argocd --namespace argocd
kubectl delete namespace argocd
kubectl delete crd applications.argoproj.io applicationsets.argoproj.io appprojects.argoproj.io
kind delete cluster --name argocd-lab
kind delete cluster --name workload-dev
```

Deleting the CRDs removes every Application object. If Applications have the `resources-finalizer`, delete the Applications *first* (or remove the finalizers) so cascading deletion does not wipe the workloads you still want.

Next: [02-core-concepts.md](02-core-concepts.md).
