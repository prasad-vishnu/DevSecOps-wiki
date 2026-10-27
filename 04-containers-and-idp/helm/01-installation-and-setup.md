# Helm: Installation and Setup

| | |
|---|---|
| **Cluster** | 04 Containers & IDP |
| **Role** | Bridge (from Kubernetes) |
| **Prerequisites** | A working `kubectl` and a cluster (kind recommended), Docker Desktop or Podman for kind |
| **Target version** | Helm 3.16+, Kubernetes 1.31+, kind 0.24+ |
| **Capstone phase** | Phase 3 (Months 9-12) |

> After this file you will have Helm 3 installed on Linux, macOS or Windows, the `diff`, `secrets` and `unittest` plugins loaded, the `ct` chart-testing tool and `kubeconform` available, a kind cluster to practise on, and a verified first release.

## 1. Install the Helm CLI

Helm is a single static Go binary. There is no server component to install in Helm 3.

### Linux (script, any distro)

```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```

The script detects architecture, downloads the latest release tarball, verifies its SHA-256 checksum and installs to `/usr/local/bin/helm`. Pin a version with `./get_helm.sh --version v3.16.3`.

### Linux (package managers)

```bash
# Debian / Ubuntu
curl -fsSL https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" \
  | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update && sudo apt-get install --yes helm

# Fedora
sudo dnf install helm

# Snap
sudo snap install helm --classic
```

### macOS

```bash
brew install helm
```

### Windows

```powershell
winget install Helm.Helm
# or
choco install kubernetes-helm
# or
scoop install helm
```

### Containers and CI runners

```dockerfile
# Pinned, checksum-verified install for a CI image
FROM alpine:3.20
ARG HELM_VERSION=3.16.3
ARG HELM_SHA256=<CHECKSUM_FROM_RELEASE_PAGE>
RUN apk add --no-cache curl ca-certificates \
 && curl -fsSLo helm.tgz "https://get.helm.sh/helm-v${HELM_VERSION}-linux-amd64.tar.gz" \
 && echo "${HELM_SHA256}  helm.tgz" | sha256sum -c - \
 && tar -xzf helm.tgz --strip-components=1 -C /usr/local/bin linux-amd64/helm \
 && rm helm.tgz
USER 65532:65532
```

GitHub Actions: `azure/setup-helm@v4` with `version: v3.16.3`. The `alpine/helm` and `alpine/k8s` images are convenient but pin a digest, not a floating tag.

### Verify

```bash
helm version
```

Expected output:

```text
version.BuildInfo{Version:"v3.16.3", GitCommit:"cfd07493f46efc9debd9cc1b02a0961186df7fdf", GitTreeState:"clean", GoVersion:"go1.22.7"}
```

## 2. Shell completion

```bash
# bash
helm completion bash | sudo tee /etc/bash_completion.d/helm > /dev/null
# zsh
helm completion zsh > "${fpath[1]}/_helm"
# fish
helm completion fish > ~/.config/fish/completions/helm.fish
# PowerShell
helm completion powershell | Out-String | Invoke-Expression
```

## 3. Where Helm keeps its files

Helm 3 follows the XDG base directory spec. Knowing these paths saves you time when something is cached or stale.

| Purpose | Linux / macOS default | Env override |
|---|---|---|
| Repo index cache, OCI blob cache | `~/.cache/helm` | `HELM_CACHE_HOME` |
| `repositories.yaml`, plugins, registry login | `~/.config/helm` | `HELM_CONFIG_HOME` |
| Plugin binaries | `~/.local/share/helm/plugins` | `HELM_DATA_HOME` / `HELM_PLUGINS` |
| Registry credentials | `~/.config/helm/registry/config.json` | `HELM_REGISTRY_CONFIG` |
| Kubeconfig | `~/.kube/config` | `KUBECONFIG` |
| Default namespace | current kubeconfig context | `HELM_NAMESPACE` or `--namespace` |
| Release storage driver | `secret` | `HELM_DRIVER` (`secret`, `configmap`, `sql`, `memory`) |

```bash
helm env
```

prints every effective path and variable.

## 4. Create a practice cluster with kind

```bash
# Install kind
# Linux
curl -Lo ./kind "https://kind.sigs.k8s.io/dl/v0.24.0/kind-linux-amd64" && chmod +x ./kind && sudo mv ./kind /usr/local/bin/kind
# macOS
brew install kind
# Windows
winget install Kubernetes.kind
```

Cluster config with an ingress-ready port mapping and a pinned node image so labs are reproducible:

```yaml
# kind-helm.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: helm-lab
nodes:
  - role: control-plane
    image: kindest/node:v1.31.2
    kubeadmConfigPatches:
      - |
        kind: InitConfiguration
        nodeRegistration:
          kubeletExtraArgs:
            node-labels: "ingress-ready=true"
    extraPortMappings:
      - containerPort: 80
        hostPort: 8080
        protocol: TCP
      - containerPort: 443
        hostPort: 8443
        protocol: TCP
  - role: worker
    image: kindest/node:v1.31.2
```

```bash
kind create cluster --config kind-helm.yaml
kubectl cluster-info --context kind-helm-lab
kubectl get nodes
```

Expected output:

```text
NAME                     STATUS   ROLES           AGE   VERSION
helm-lab-control-plane   Ready    control-plane   60s   v1.31.2
helm-lab-worker          Ready    <none>          40s   v1.31.2
```

## 5. Add a repository and install a first release

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm search repo ingress-nginx --versions | head -5
```

Install ingress-nginx configured for kind (hostNetwork on the labelled node):

```bash
helm upgrade --install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx --create-namespace \
  --version 4.11.3 \
  --set controller.hostNetwork=true \
  --set controller.hostPort.enabled=true \
  --set controller.service.type=ClusterIP \
  --set controller.nodeSelector."ingress-ready"=true \
  --set controller.tolerations[0].key=node-role.kubernetes.io/control-plane \
  --set controller.tolerations[0].operator=Exists \
  --set controller.tolerations[0].effect=NoSchedule \
  --wait --timeout 3m
```

Expected output (abridged):

```text
Release "ingress-nginx" does not exist. Installing it now.
NAME: ingress-nginx
LAST DEPLOYED: Mon Oct 27 10:12:44 2025
NAMESPACE: ingress-nginx
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
The ingress-nginx controller has been installed.
...
```

```bash
helm list --all-namespaces
kubectl get secret --namespace ingress-nginx -l owner=helm
```

The second command shows the release Secret `sh.helm.release.v1.ingress-nginx.v1`. That Secret *is* the release; delete it and Helm forgets the release exists (the objects stay).

## 6. Install the plugins used in this folder

Plugins are Git repositories cloned into `HELM_PLUGINS`. Pin versions in CI.

```bash
# helm-diff: show what an upgrade would change
helm plugin install https://github.com/databus23/helm-diff --version v3.9.11

# helm-secrets: transparently decrypt SOPS-encrypted values files
helm plugin install https://github.com/jkroepke/helm-secrets --version v4.6.2

# helm-unittest: unit-test rendered templates with YAML assertions
helm plugin install https://github.com/helm-unittest/helm-unittest --version v0.6.3

helm plugin list
```

Expected output:

```text
NAME        VERSION  DESCRIPTION
diff        3.9.11   Preview helm upgrade changes as a diff
secrets     4.6.2    This plugin provides secrets values encryption for Helm charts secure storing
unittest    0.6.3    Unit test for helm chart in YAML with ease to keep your chart functional and robust!
```

helm-secrets needs a backend. Install SOPS and age:

```bash
# SOPS
# Linux
curl -fsSLo sops https://github.com/getsops/sops/releases/download/v3.9.1/sops-v3.9.1.linux.amd64 && chmod +x sops && sudo mv sops /usr/local/bin/
# macOS
brew install sops age

# age key for labs (never commit the private key)
age-keygen -o ~/.config/sops/age/keys.txt
export SOPS_AGE_RECIPIENTS=$(grep -o 'age1[0-9a-z]*' ~/.config/sops/age/keys.txt)
```

## 7. Install chart-testing (`ct`) and kubeconform

```bash
# ct
# macOS
brew install chart-testing
# Linux
curl -fsSLo ct.tgz https://github.com/helm/chart-testing/releases/download/v3.11.0/chart-testing_3.11.0_linux_amd64.tar.gz
tar -xzf ct.tgz ct && sudo mv ct /usr/local/bin/ && rm ct.tgz
ct version

# ct depends on yamllint and yamale
pip install --user yamllint yamale

# kubeconform: validate rendered manifests against Kubernetes OpenAPI schemas
# macOS
brew install kubeconform
# Linux
curl -fsSLo kubeconform.tgz https://github.com/yannh/kubeconform/releases/download/v0.6.7/kubeconform-linux-amd64.tar.gz
tar -xzf kubeconform.tgz kubeconform && sudo mv kubeconform /usr/local/bin/ && rm kubeconform.tgz
```

Minimal `ct.yaml` you will reuse in labs and CI:

```yaml
# ct.yaml
remote: origin
target-branch: main
chart-dirs:
  - charts
chart-repos:
  - bitnami=https://charts.bitnami.com/bitnami
helm-extra-args: --timeout 300s
validate-maintainers: false
check-version-increment: true
```

## 8. Log in to an OCI registry

Helm 3.8+ treats OCI registries as first-class chart sources. Use a short-lived token, never a long-lived password in your shell history.

```bash
# GitHub Container Registry with a fine-grained token in an env var
echo "$GHCR_TOKEN" | helm registry login ghcr.io --username <GITHUB_USERNAME> --password-stdin

# Amazon ECR (token valid 12 hours)
aws ecr get-login-password --region <REGION> \
  | helm registry login --username AWS --password-stdin <ACCOUNT_ID>.dkr.ecr.<REGION>.amazonaws.com

# Azure Container Registry via Entra ID
az acr login --name <ACR_NAME> --expose-token --output tsv --query accessToken \
  | helm registry login <ACR_NAME>.azurecr.io --username 00000000-0000-0000-0000-000000000000 --password-stdin

# Google Artifact Registry
gcloud auth print-access-token \
  | helm registry login --username oauth2accesstoken --password-stdin <REGION>-docker.pkg.dev
```

Credentials are stored in `~/.config/helm/registry/config.json`. Helm also reads Docker's credential helpers, so if `docker login` works, `helm registry` usually works too.

Verify by pulling a public OCI chart:

```bash
helm pull oci://registry-1.docker.io/bitnamicharts/redis --version 20.2.1
ls redis-20.2.1.tgz
```

## 9. Editor and local tooling

- VS Code: the "Kubernetes" extension (ms-kubernetes-tools) adds Helm template syntax highlighting and `helm template` preview. Add `"files.associations": {"**/templates/*.yaml": "helm"}` to settings to stop YAML linters complaining about `{{ }}`.
- `yq` (mikefarah) for inspecting rendered output: `helm template . | yq 'select(.kind == "Deployment") | .spec.template.spec.containers[0].image'`.
- `pre-commit` hook example:

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/gruntwork-io/pre-commit
    rev: v0.1.24
    hooks:
      - id: helmlint
  - repo: https://github.com/norwoodj/helm-docs
    rev: v1.14.2
    hooks:
      - id: helm-docs
```

## 10. Verification checklist

```bash
helm version --short                   # v3.16.x
helm plugin list | grep -E 'diff|secrets|unittest'
ct version
kubeconform -v
kubectl config current-context         # kind-helm-lab
helm list --all-namespaces             # ingress-nginx deployed
helm env | grep HELM_PLUGINS
```

If every line prints what you expect, proceed to [02-core-concepts.md](02-core-concepts.md).

## Cleanup (if you are not continuing to the labs immediately)

```bash
helm uninstall ingress-nginx --namespace ingress-nginx
kind delete cluster --name helm-lab
```
