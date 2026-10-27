# Helm: Hands-On Labs

| | |
|---|---|
| **Cluster** | 04 Containers & IDP |
| **Role** | Bridge (from Kubernetes) |
| **Prerequisites** | [01-installation-and-setup.md](01-installation-and-setup.md) complete: kind cluster `helm-lab`, Helm 3.16+, plugins `diff`/`secrets`/`unittest`, `ct`, `kubeconform`, `sops`, `age` |
| **Target version** | Helm 3.16+, Kubernetes 1.31+ |
| **Capstone phase** | Phase 3 (Months 9-12) |

> Six progressive labs. You start by installing and upgrading a third-party chart to learn release mechanics, then build the capstone's production chart for the FastAPI `api` service from scratch (HPA, PDB, NetworkPolicy, ServiceMonitor toggles, migration hook, test hook, JSON schema), unit-test and lint it the way CI does, encrypt environment secrets with SOPS, and finally publish the chart to an OCI registry and sign it. Every lab ends with cleanup.

Lab layout used throughout (create it once):

```bash
mkdir -p ~/helm-labs/charts ~/helm-labs/envs/{dev,stage,prod} && cd ~/helm-labs
kubectl config use-context kind-helm-lab
```

---

## Lab 1: Releases, Revisions and Rollback with a Third-Party Chart

**Objective.** Understand what a release physically is by installing, upgrading, breaking and rolling back a Redis chart.

**Prerequisites.** kind cluster up; internet access to Docker Hub.

**Time.** 30 minutes.

### Steps

1. Install Bitnami Redis from its OCI registry with a secure, minimal configuration.

   ```bash
   kubectl create namespace cache
   helm install cache oci://registry-1.docker.io/bitnamicharts/redis --version 20.2.1 \
     --namespace cache \
     --set architecture=standalone \
     --set auth.enabled=true \
     --set auth.password=lab-only-not-a-real-password \
     --set master.persistence.enabled=false \
     --set master.resources.limits.memory=256Mi \
     --wait --timeout 3m
   ```

   Expected output (abridged):

   ```text
   Pulled: registry-1.docker.io/bitnamicharts/redis:20.2.1
   NAME: cache
   LAST DEPLOYED: ...
   NAMESPACE: cache
   STATUS: deployed
   REVISION: 1
   ```

   Passing a password with `--set` is an insecure shortcut for this lab; it lands in shell history and the release Secret. Lab 5 shows the right way.

2. Look at the release storage.

   ```bash
   helm list --namespace cache
   kubectl get secrets --namespace cache -l owner=helm
   helm get manifest cache --namespace cache | grep -c '^kind:'
   ```

   The third command counts the Kubernetes objects Helm manages for this release.

3. Upgrade with a changed value and inspect the diff first.

   ```bash
   helm diff upgrade cache oci://registry-1.docker.io/bitnamicharts/redis --version 20.2.1 \
     --namespace cache --reuse-values \
     --set master.resources.limits.memory=512Mi --context 2
   ```

   Expected output (abridged):

   ```text
   cache, cache-master, StatefulSet (apps) has changed:
   ...
   -             memory: 256Mi
   +             memory: 512Mi
   ```

   Now apply:

   ```bash
   helm upgrade cache oci://registry-1.docker.io/bitnamicharts/redis --version 20.2.1 \
     --namespace cache --reuse-values \
     --set master.resources.limits.memory=512Mi --wait
   helm history cache --namespace cache
   ```

   Expected output:

   ```text
   REVISION  UPDATED  STATUS      CHART         APP VERSION  DESCRIPTION
   1         ...      superseded  redis-20.2.1  7.4.1        Install complete
   2         ...      deployed    redis-20.2.1  7.4.1        Upgrade complete
   ```

4. Break it on purpose: upgrade to a non-existent image tag with `--atomic`.

   ```bash
   helm upgrade cache oci://registry-1.docker.io/bitnamicharts/redis --version 20.2.1 \
     --namespace cache --reuse-values \
     --set image.tag=does-not-exist --atomic --wait --timeout 90s
   ```

   Expected output:

   ```text
   Error: UPGRADE FAILED: release cache failed, and has been rolled back due to atomic being set: context deadline exceeded
   ```

   ```bash
   helm history cache --namespace cache
   ```

   Revision 3 is `failed`, revision 4 is `deployed` with description `Rollback to 2`. `--atomic` did the rollback for you.

5. Roll back by hand to revision 1 and confirm the memory limit returns to 256Mi.

   ```bash
   helm rollback cache 1 --namespace cache --wait
   kubectl get statefulset cache-master --namespace cache -o jsonpath='{.spec.template.spec.containers[0].resources.limits.memory}{"\n"}'
   ```

   Expected output: `256Mi`.

### Verify

```bash
helm status cache --namespace cache --show-resources
helm get values cache --namespace cache
```

The values shown are those of revision 1 (rollback restores values too).

### Cleanup

```bash
helm uninstall cache --namespace cache
kubectl delete namespace cache
```

### What you learned

A release is a chain of Secrets; `upgrade` appends, `rollback` appends a copy of an older one; `--atomic` is your safety net; `helm diff` is how you look before you leap.

---

## Lab 2: Build the Production `api` Chart

**Objective.** Author the capstone FastAPI chart from scratch with secure defaults and feature toggles for HPA, PDB, NetworkPolicy and ServiceMonitor.

**Prerequisites.** Lab 1 concepts. No cluster needed until the end.

**Time.** 90 minutes.

### Steps

1. Scaffold and strip the starter.

   ```bash
   cd ~/helm-labs/charts
   helm create api
   rm -rf api/templates/* api/charts
   ```

2. Write `Chart.yaml`.

   ```yaml
   # charts/api/Chart.yaml
   apiVersion: v2
   name: api
   description: Capstone FastAPI service
   type: application
   version: 0.1.0
   appVersion: "1.0.0"
   kubeVersion: ">=1.29.0-0"
   maintainers:
     - name: Platform Team
       email: platform@example.com
   ```

3. Write `values.yaml`. This is the chart's public API; every key is commented for helm-docs.

   ```yaml
   # charts/api/values.yaml
   # -- Replica count used only when autoscaling.enabled is false
   replicaCount: 2

   image:
     # -- Container image repository
     repository: ghcr.io/example/api
     # -- Image tag; empty means .Chart.AppVersion
     tag: ""
     # -- Image digest (sha256:...). When set, overrides tag for immutability
     digest: ""
     pullPolicy: IfNotPresent

   # -- Image pull secrets, e.g. [{name: ghcr-pull}]
   imagePullSecrets: []
   nameOverride: ""
   fullnameOverride: ""

   serviceAccount:
     create: true
     # -- Annotations, e.g. eks.amazonaws.com/role-arn for IRSA
     annotations: {}
     name: ""
     automountServiceAccountToken: false

   podAnnotations: {}
   podLabels: {}

   podSecurityContext:
     runAsNonRoot: true
     runAsUser: 65532
     runAsGroup: 65532
     fsGroup: 65532
     seccompProfile:
       type: RuntimeDefault

   securityContext:
     allowPrivilegeEscalation: false
     readOnlyRootFilesystem: true
     capabilities:
       drop: [ALL]

   containerPort: 8000

   service:
     type: ClusterIP
     port: 80

   ingress:
     enabled: false
     className: nginx
     annotations: {}
     hosts:
       - host: api.example.com
         paths:
           - path: /
             pathType: Prefix
     tls: []

   resources:
     requests:
       cpu: 100m
       memory: 128Mi
     limits:
       memory: 256Mi

   # -- Non-secret environment variables (key: value)
   env:
     LOG_LEVEL: info
     APP_ENV: dev

   # -- Secret environment variables sourced from existing Secrets
   envFromSecrets: []
   #  - name: api-db

   # -- Application config rendered into a ConfigMap and mounted at /etc/api/config.toml
   config: |
     [server]
     workers = 2

   livenessProbe:
     httpGet:
       path: /healthz
       port: http
     initialDelaySeconds: 5
     periodSeconds: 10
   readinessProbe:
     httpGet:
       path: /readyz
       port: http
     periodSeconds: 5
   startupProbe:
     httpGet:
       path: /healthz
       port: http
     failureThreshold: 30
     periodSeconds: 2

   autoscaling:
     enabled: false
     minReplicas: 2
     maxReplicas: 10
     targetCPUUtilizationPercentage: 70
     targetMemoryUtilizationPercentage: null
     behavior: {}

   podDisruptionBudget:
     enabled: true
     # -- Exactly one of minAvailable / maxUnavailable
     minAvailable: 1
     maxUnavailable: null

   networkPolicy:
     enabled: true
     # -- Namespaces allowed to reach the API (by kubernetes.io/metadata.name label)
     ingressFromNamespaces: [ingress-nginx]
     # -- Extra ingress rules appended verbatim
     extraIngress: []
     # -- Egress allow-list; DNS to kube-system is always allowed
     egress:
       - to:
           - ipBlock:
               cidr: 0.0.0.0/0
               except: [169.254.169.254/32]
         ports:
           - port: 443
             protocol: TCP

   serviceMonitor:
     enabled: false
     interval: 30s
     path: /metrics
     labels: {}

   migrations:
     # -- Run `alembic upgrade head` as a pre-install/pre-upgrade hook
     enabled: true
     command: ["alembic", "upgrade", "head"]

   topologySpreadConstraints:
     - maxSkew: 1
       topologyKey: topology.kubernetes.io/zone
       whenUnsatisfiable: ScheduleAnyway
       labelSelector: {}

   nodeSelector: {}
   tolerations: []
   affinity: {}
   priorityClassName: ""
   terminationGracePeriodSeconds: 30
   ```

4. Write `templates/_helpers.tpl`.

   ```yaml
   {{/* charts/api/templates/_helpers.tpl */}}
   {{- define "api.name" -}}
   {{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
   {{- end }}

   {{- define "api.fullname" -}}
   {{- if .Values.fullnameOverride }}
   {{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
   {{- else }}
   {{- $name := default .Chart.Name .Values.nameOverride }}
   {{- if contains $name .Release.Name }}
   {{- .Release.Name | trunc 63 | trimSuffix "-" }}
   {{- else }}
   {{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" }}
   {{- end }}
   {{- end }}
   {{- end }}

   {{- define "api.chart" -}}
   {{- printf "%s-%s" .Chart.Name .Chart.Version | replace "+" "_" | trunc 63 | trimSuffix "-" }}
   {{- end }}

   {{- define "api.selectorLabels" -}}
   app.kubernetes.io/name: {{ include "api.name" . }}
   app.kubernetes.io/instance: {{ .Release.Name }}
   {{- end }}

   {{- define "api.labels" -}}
   helm.sh/chart: {{ include "api.chart" . }}
   {{ include "api.selectorLabels" . }}
   app.kubernetes.io/version: {{ .Values.image.tag | default .Chart.AppVersion | quote }}
   app.kubernetes.io/managed-by: {{ .Release.Service }}
   app.kubernetes.io/part-of: capstone
   {{- end }}

   {{- define "api.serviceAccountName" -}}
   {{- if .Values.serviceAccount.create }}
   {{- default (include "api.fullname" .) .Values.serviceAccount.name }}
   {{- else }}
   {{- default "default" .Values.serviceAccount.name }}
   {{- end }}
   {{- end }}

   {{/* Image reference: digest wins over tag */}}
   {{- define "api.image" -}}
   {{- $repo := required "image.repository is required" .Values.image.repository -}}
   {{- if .Values.image.digest -}}
   {{ printf "%s@%s" $repo .Values.image.digest }}
   {{- else -}}
   {{ printf "%s:%s" $repo (.Values.image.tag | default .Chart.AppVersion) }}
   {{- end -}}
   {{- end }}

   {{/* Validation that runs on every render */}}
   {{- define "api.validate" -}}
   {{- if and .Values.podDisruptionBudget.enabled .Values.podDisruptionBudget.minAvailable .Values.podDisruptionBudget.maxUnavailable }}
   {{- fail "podDisruptionBudget: set only one of minAvailable or maxUnavailable" }}
   {{- end }}
   {{- if and .Values.autoscaling.enabled (gt (int .Values.autoscaling.minReplicas) (int .Values.autoscaling.maxReplicas)) }}
   {{- fail "autoscaling.minReplicas must be <= autoscaling.maxReplicas" }}
   {{- end }}
   {{- end }}
   ```

5. Write the workload templates.

   ```yaml
   # charts/api/templates/serviceaccount.yaml
   {{- if .Values.serviceAccount.create -}}
   apiVersion: v1
   kind: ServiceAccount
   metadata:
     name: {{ include "api.serviceAccountName" . }}
     labels: {{- include "api.labels" . | nindent 4 }}
     {{- with .Values.serviceAccount.annotations }}
     annotations: {{- toYaml . | nindent 4 }}
     {{- end }}
   automountServiceAccountToken: {{ .Values.serviceAccount.automountServiceAccountToken }}
   {{- end }}
   ```

   ```yaml
   # charts/api/templates/configmap.yaml
   apiVersion: v1
   kind: ConfigMap
   metadata:
     name: {{ include "api.fullname" . }}
     labels: {{- include "api.labels" . | nindent 4 }}
   data:
     config.toml: |
       {{- tpl .Values.config . | nindent 4 }}
   ```

   ```yaml
   # charts/api/templates/deployment.yaml
   {{- include "api.validate" . -}}
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: {{ include "api.fullname" . }}
     labels: {{- include "api.labels" . | nindent 4 }}
   spec:
     {{- if not .Values.autoscaling.enabled }}
     replicas: {{ .Values.replicaCount }}
     {{- end }}
     revisionHistoryLimit: 5
     strategy:
       type: RollingUpdate
       rollingUpdate:
         maxSurge: 25%
         maxUnavailable: 0
     selector:
       matchLabels: {{- include "api.selectorLabels" . | nindent 6 }}
     template:
       metadata:
         annotations:
           checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
           {{- with .Values.podAnnotations }}
           {{- toYaml . | nindent 8 }}
           {{- end }}
         labels:
           {{- include "api.selectorLabels" . | nindent 8 }}
           {{- with .Values.podLabels }}
           {{- toYaml . | nindent 8 }}
           {{- end }}
       spec:
         {{- with .Values.imagePullSecrets }}
         imagePullSecrets: {{- toYaml . | nindent 8 }}
         {{- end }}
         serviceAccountName: {{ include "api.serviceAccountName" . }}
         automountServiceAccountToken: {{ .Values.serviceAccount.automountServiceAccountToken }}
         securityContext: {{- toYaml .Values.podSecurityContext | nindent 8 }}
         terminationGracePeriodSeconds: {{ .Values.terminationGracePeriodSeconds }}
         {{- with .Values.priorityClassName }}
         priorityClassName: {{ . }}
         {{- end }}
         containers:
           - name: api
             image: {{ include "api.image" . | quote }}
             imagePullPolicy: {{ .Values.image.pullPolicy }}
             securityContext: {{- toYaml .Values.securityContext | nindent 12 }}
             ports:
               - name: http
                 containerPort: {{ .Values.containerPort }}
                 protocol: TCP
             env:
               {{- range $k, $v := .Values.env }}
               - name: {{ $k }}
                 value: {{ $v | quote }}
               {{- end }}
               - name: PORT
                 value: {{ .Values.containerPort | quote }}
             {{- with .Values.envFromSecrets }}
             envFrom:
               {{- range . }}
               - secretRef:
                   name: {{ .name }}
               {{- end }}
             {{- end }}
             livenessProbe: {{- toYaml .Values.livenessProbe | nindent 12 }}
             readinessProbe: {{- toYaml .Values.readinessProbe | nindent 12 }}
             startupProbe: {{- toYaml .Values.startupProbe | nindent 12 }}
             resources: {{- toYaml .Values.resources | nindent 12 }}
             volumeMounts:
               - name: config
                 mountPath: /etc/api
                 readOnly: true
               - name: tmp
                 mountPath: /tmp
         volumes:
           - name: config
             configMap:
               name: {{ include "api.fullname" . }}
           - name: tmp
             emptyDir:
               sizeLimit: 64Mi
         {{- with .Values.topologySpreadConstraints }}
         topologySpreadConstraints:
           {{- range . }}
           - maxSkew: {{ .maxSkew }}
             topologyKey: {{ .topologyKey }}
             whenUnsatisfiable: {{ .whenUnsatisfiable }}
             labelSelector:
               matchLabels: {{- include "api.selectorLabels" $ | nindent 16 }}
           {{- end }}
         {{- end }}
         {{- with .Values.nodeSelector }}
         nodeSelector: {{- toYaml . | nindent 8 }}
         {{- end }}
         {{- with .Values.affinity }}
         affinity: {{- toYaml . | nindent 8 }}
         {{- end }}
         {{- with .Values.tolerations }}
         tolerations: {{- toYaml . | nindent 8 }}
         {{- end }}
   ```

   ```yaml
   # charts/api/templates/service.yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: {{ include "api.fullname" . }}
     labels: {{- include "api.labels" . | nindent 4 }}
   spec:
     type: {{ .Values.service.type }}
     ports:
       - port: {{ .Values.service.port }}
         targetPort: http
         protocol: TCP
         name: http
     selector: {{- include "api.selectorLabels" . | nindent 4 }}
   ```

6. Write the toggled templates.

   ```yaml
   # charts/api/templates/hpa.yaml
   {{- if .Values.autoscaling.enabled }}
   apiVersion: autoscaling/v2
   kind: HorizontalPodAutoscaler
   metadata:
     name: {{ include "api.fullname" . }}
     labels: {{- include "api.labels" . | nindent 4 }}
   spec:
     scaleTargetRef:
       apiVersion: apps/v1
       kind: Deployment
       name: {{ include "api.fullname" . }}
     minReplicas: {{ .Values.autoscaling.minReplicas }}
     maxReplicas: {{ .Values.autoscaling.maxReplicas }}
     metrics:
       {{- with .Values.autoscaling.targetCPUUtilizationPercentage }}
       - type: Resource
         resource:
           name: cpu
           target:
             type: Utilization
             averageUtilization: {{ . }}
       {{- end }}
       {{- with .Values.autoscaling.targetMemoryUtilizationPercentage }}
       - type: Resource
         resource:
           name: memory
           target:
             type: Utilization
             averageUtilization: {{ . }}
       {{- end }}
     {{- with .Values.autoscaling.behavior }}
     behavior: {{- toYaml . | nindent 4 }}
     {{- end }}
   {{- end }}
   ```

   ```yaml
   # charts/api/templates/pdb.yaml
   {{- if .Values.podDisruptionBudget.enabled }}
   apiVersion: policy/v1
   kind: PodDisruptionBudget
   metadata:
     name: {{ include "api.fullname" . }}
     labels: {{- include "api.labels" . | nindent 4 }}
   spec:
     {{- if .Values.podDisruptionBudget.minAvailable }}
     minAvailable: {{ .Values.podDisruptionBudget.minAvailable }}
     {{- else }}
     maxUnavailable: {{ .Values.podDisruptionBudget.maxUnavailable | default 1 }}
     {{- end }}
     selector:
       matchLabels: {{- include "api.selectorLabels" . | nindent 6 }}
   {{- end }}
   ```

   ```yaml
   # charts/api/templates/networkpolicy.yaml
   {{- if .Values.networkPolicy.enabled }}
   apiVersion: networking.k8s.io/v1
   kind: NetworkPolicy
   metadata:
     name: {{ include "api.fullname" . }}
     labels: {{- include "api.labels" . | nindent 4 }}
   spec:
     podSelector:
       matchLabels: {{- include "api.selectorLabels" . | nindent 6 }}
     policyTypes: [Ingress, Egress]
     ingress:
       - from:
           {{- range .Values.networkPolicy.ingressFromNamespaces }}
           - namespaceSelector:
               matchLabels:
                 kubernetes.io/metadata.name: {{ . }}
           {{- end }}
           - podSelector: {}
         ports:
           - port: {{ .Values.containerPort }}
             protocol: TCP
       {{- with .Values.networkPolicy.extraIngress }}
       {{- toYaml . | nindent 4 }}
       {{- end }}
     egress:
       - to:
           - namespaceSelector:
               matchLabels:
                 kubernetes.io/metadata.name: kube-system
             podSelector:
               matchLabels:
                 k8s-app: kube-dns
         ports:
           - port: 53
             protocol: UDP
           - port: 53
             protocol: TCP
       {{- with .Values.networkPolicy.egress }}
       {{- toYaml . | nindent 4 }}
       {{- end }}
   {{- end }}
   ```

   ```yaml
   # charts/api/templates/servicemonitor.yaml
   {{- if and .Values.serviceMonitor.enabled (.Capabilities.APIVersions.Has "monitoring.coreos.com/v1") }}
   apiVersion: monitoring.coreos.com/v1
   kind: ServiceMonitor
   metadata:
     name: {{ include "api.fullname" . }}
     labels:
       {{- include "api.labels" . | nindent 4 }}
       {{- with .Values.serviceMonitor.labels }}
       {{- toYaml . | nindent 4 }}
       {{- end }}
   spec:
     selector:
       matchLabels: {{- include "api.selectorLabels" . | nindent 6 }}
     namespaceSelector:
       matchNames: [{{ .Release.Namespace }}]
     endpoints:
       - port: http
         path: {{ .Values.serviceMonitor.path }}
         interval: {{ .Values.serviceMonitor.interval }}
   {{- end }}
   ```

   ```yaml
   # charts/api/templates/ingress.yaml
   {{- if .Values.ingress.enabled }}
   {{- $fullName := include "api.fullname" . }}
   {{- $svcPort := .Values.service.port }}
   apiVersion: networking.k8s.io/v1
   kind: Ingress
   metadata:
     name: {{ $fullName }}
     labels: {{- include "api.labels" . | nindent 4 }}
     {{- with .Values.ingress.annotations }}
     annotations: {{- toYaml . | nindent 4 }}
     {{- end }}
   spec:
     ingressClassName: {{ .Values.ingress.className }}
     {{- with .Values.ingress.tls }}
     tls: {{- toYaml . | nindent 4 }}
     {{- end }}
     rules:
       {{- range .Values.ingress.hosts }}
       - host: {{ .host | quote }}
         http:
           paths:
             {{- range .paths }}
             - path: {{ .path }}
               pathType: {{ .pathType }}
               backend:
                 service:
                   name: {{ $fullName }}
                   port:
                     number: {{ $svcPort }}
             {{- end }}
       {{- end }}
   {{- end }}
   ```

7. Write the hooks and NOTES.

   ```yaml
   # charts/api/templates/migration-job.yaml
   {{- if .Values.migrations.enabled }}
   apiVersion: batch/v1
   kind: Job
   metadata:
     name: {{ include "api.fullname" . }}-migrate
     labels: {{- include "api.labels" . | nindent 4 }}
     annotations:
       "helm.sh/hook": pre-install,pre-upgrade
       "helm.sh/hook-weight": "-5"
       "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
   spec:
     backoffLimit: 2
     ttlSecondsAfterFinished: 600
     template:
       metadata:
         labels: {{- include "api.selectorLabels" . | nindent 10 }}
       spec:
         restartPolicy: Never
         serviceAccountName: {{ include "api.serviceAccountName" . }}
         securityContext: {{- toYaml .Values.podSecurityContext | nindent 10 }}
         containers:
           - name: migrate
             image: {{ include "api.image" . | quote }}
             command: {{- toYaml .Values.migrations.command | nindent 14 }}
             securityContext: {{- toYaml .Values.securityContext | nindent 14 }}
             env:
               {{- range $k, $v := .Values.env }}
               - name: {{ $k }}
                 value: {{ $v | quote }}
               {{- end }}
             {{- with .Values.envFromSecrets }}
             envFrom:
               {{- range . }}
               - secretRef:
                   name: {{ .name }}
               {{- end }}
             {{- end }}
             resources: {{- toYaml .Values.resources | nindent 14 }}
   {{- end }}
   ```

   ```yaml
   # charts/api/templates/tests/test-connection.yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: {{ include "api.fullname" . }}-test-connection
     labels: {{- include "api.labels" . | nindent 4 }}
     annotations:
       "helm.sh/hook": test
       "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
   spec:
     restartPolicy: Never
     securityContext: {{- toYaml .Values.podSecurityContext | nindent 4 }}
     containers:
       - name: curl
         image: curlimages/curl:8.10.1
         command: ["curl", "--fail", "--silent", "--max-time", "10",
                   "http://{{ include "api.fullname" . }}:{{ .Values.service.port }}/healthz"]
         securityContext: {{- toYaml .Values.securityContext | nindent 8 }}
   ```

   ```text
   {{/* charts/api/templates/NOTES.txt */}}
   {{ .Chart.Name }} {{ .Chart.Version }} (app {{ .Values.image.tag | default .Chart.AppVersion }}) deployed as release {{ .Release.Name }} in {{ .Release.Namespace }}.

   Image: {{ include "api.image" . }}
   Autoscaling: {{ ternary "enabled" "disabled" .Values.autoscaling.enabled }}
   NetworkPolicy: {{ ternary "enabled" "disabled" .Values.networkPolicy.enabled }}

   Port-forward for a local check:
     kubectl --namespace {{ .Release.Namespace }} port-forward svc/{{ include "api.fullname" . }} 8080:{{ .Values.service.port }}
     curl http://localhost:8080/healthz
   ```

8. Add `values.schema.json` (copy the one in [02-core-concepts.md](02-core-concepts.md#12-schema-validation) and extend it with `podDisruptionBudget`, `networkPolicy.enabled`, `serviceMonitor.enabled` as booleans).

9. Render and validate without a cluster.

   ```bash
   cd ~/helm-labs
   helm lint charts/api --strict
   helm template api charts/api --namespace api | kubeconform -strict -summary
   helm template api charts/api --namespace api \
     --set autoscaling.enabled=true --set serviceMonitor.enabled=true \
     --api-versions monitoring.coreos.com/v1 \
     | grep -E '^kind:' | sort | uniq -c
   ```

   Expected output of the last command:

   ```text
      1 kind: ConfigMap
      1 kind: Deployment
      1 kind: HorizontalPodAutoscaler
      1 kind: Job
      1 kind: NetworkPolicy
      1 kind: Pod
      1 kind: PodDisruptionBudget
      1 kind: Service
      1 kind: ServiceAccount
      1 kind: ServiceMonitor
   ```

   Confirm the HPA toggle removes `replicas`:

   ```bash
   helm template api charts/api --set autoscaling.enabled=true --show-only templates/deployment.yaml | grep -c 'replicas:'
   ```

   Expected: `0`.

10. Trigger the validation helper.

    ```bash
    helm template api charts/api --set autoscaling.enabled=true --set autoscaling.minReplicas=5 --set autoscaling.maxReplicas=2
    ```

    Expected output:

    ```text
    Error: execution error at (api/templates/deployment.yaml:1:4): autoscaling.minReplicas must be <= autoscaling.maxReplicas
    ```

### Verify

`helm lint --strict` reports `0 chart(s) failed`; `kubeconform` reports no invalid resources.

### Cleanup

Nothing was installed. Keep the chart for the next labs.

### What you learned

A production chart is mostly conditionals around a hardened Deployment; secure defaults live in `values.yaml`; validation belongs in `_helpers.tpl` and `values.schema.json`; the HPA/replicas interaction is the canonical three-way-merge trap.

---

## Lab 3: Install, Upgrade and Test on kind with Per-Environment Values

**Objective.** Deploy the chart to the kind cluster with a dev values file, exercise hooks, `helm test`, config-checksum rollouts and the HPA toggle.

**Prerequisites.** Lab 2; ingress-nginx and metrics-server in the cluster.

**Time.** 45 minutes.

### Steps

1. Install metrics-server (needed for HPA) tuned for kind.

   ```bash
   helm repo add metrics-server https://kubernetes-sigs.github.io/metrics-server/
   helm upgrade --install metrics-server metrics-server/metrics-server \
     --namespace kube-system --version 3.12.2 \
     --set 'args={--kubelet-insecure-tls}' --wait
   ```

   `--kubelet-insecure-tls` is a kind-only shortcut; never set it on a real cluster.

2. Since the real FastAPI image is built in [../docker/04-hands-on-labs.md](../docker/04-hands-on-labs.md), use a stand-in image that serves `/healthz` for this lab and disable migrations.

   ```yaml
   # envs/dev/values.yaml
   image:
     repository: ghcr.io/example/api
     tag: "1.0.0"
   replicaCount: 1
   env:
     APP_ENV: dev
     LOG_LEVEL: debug
   migrations:
     enabled: false
   ingress:
     enabled: true
     hosts:
       - host: api.127.0.0.1.nip.io
         paths:
           - path: /
             pathType: Prefix
   networkPolicy:
     ingressFromNamespaces: [ingress-nginx]
   ```

   For the stand-in, build and load a tiny image into kind:

   ```bash
   mkdir -p /tmp/stub && cat > /tmp/stub/Dockerfile <<'EOF'
   FROM python:3.12-slim
   RUN pip install --no-cache-dir fastapi==0.115.4 uvicorn==0.32.0 && useradd -u 65532 app
   COPY app.py /app/app.py
   USER 65532
   CMD ["uvicorn", "--app-dir", "/app", "app:app", "--host", "0.0.0.0", "--port", "8000"]
   EOF
   cat > /tmp/stub/app.py <<'EOF'
   from fastapi import FastAPI
   app = FastAPI()
   @app.get("/healthz")
   def healthz(): return {"status": "ok"}
   @app.get("/readyz")
   def readyz(): return {"ready": True}
   @app.get("/")
   def root(): return {"service": "api"}
   EOF
   docker build -t ghcr.io/example/api:1.0.0 /tmp/stub
   kind load docker-image ghcr.io/example/api:1.0.0 --name helm-lab
   ```

3. Install.

   ```bash
   helm upgrade --install api charts/api --namespace api --create-namespace \
     -f envs/dev/values.yaml --wait --timeout 2m
   helm test api --namespace api --logs
   ```

   Expected output (abridged):

   ```text
   NAME: api
   ...
   TEST SUITE:     api-test-connection
   Last Started:   ...
   Phase:          Succeeded
   POD LOGS: api-test-connection
   {"status":"ok"}
   ```

4. Hit it through the ingress.

   ```bash
   curl -s http://api.127.0.0.1.nip.io:8080/
   ```

   Expected: `{"service":"api"}`.

5. Change only config and prove the checksum annotation triggers a rollout.

   ```bash
   kubectl get pods --namespace api -o name
   helm upgrade api charts/api --namespace api -f envs/dev/values.yaml \
     --set-string 'config=[server]\nworkers = 4' --wait
   kubectl get pods --namespace api -o name
   ```

   The pod name changed although the Deployment image did not.

6. Turn on autoscaling and watch `replicas` disappear from the Deployment spec while the HPA appears.

   ```bash
   helm upgrade api charts/api --namespace api -f envs/dev/values.yaml \
     --set autoscaling.enabled=true --set autoscaling.minReplicas=2 --wait
   kubectl get hpa --namespace api
   kubectl get deployment api --namespace api -o jsonpath='{.spec.replicas}{"\n"}'
   ```

   `spec.replicas` prints `2`, set by the HPA, not by Helm. Run `helm get manifest api -n api | grep replicas` to confirm Helm no longer manages the field.

7. Enable migrations with a harmless command to see hook ordering.

   ```bash
   helm upgrade api charts/api --namespace api -f envs/dev/values.yaml \
     --set migrations.enabled=true \
     --set 'migrations.command={python,-c,print("migrated")}' --wait
   kubectl get jobs --namespace api
   helm get hooks api --namespace api | grep -A2 'helm.sh/hook'
   ```

   The Job completed and was deleted (`hook-succeeded`); the hook manifest is still visible in `helm get hooks`.

### Verify

```bash
helm history api --namespace api
kubectl get networkpolicy,pdb,hpa --namespace api
```

### Cleanup

Keep the release for Lab 4. If stopping here: `helm uninstall api -n api && kubectl delete ns api`.

### What you learned

Values files per environment are the unit of promotion; `helm test` turns smoke tests into part of the release; checksum annotations and HPA-aware templates are what separate a demo chart from a production one.

---

## Lab 4: Unit Tests, `ct` and a CI Workflow

**Objective.** Make the chart self-verifying: helm-unittest assertions, `ct lint`/`ct install`, and a GitHub Actions workflow that runs them in a kind cluster.

**Prerequisites.** Lab 3; `ct`, `helm unittest` plugin.

**Time.** 45 minutes.

### Steps

1. Write unit tests.

   ```yaml
   # charts/api/tests/deployment_test.yaml
   suite: deployment
   templates:
     - deployment.yaml
     - configmap.yaml
   tests:
     - it: runs as non-root with a read-only root filesystem
       template: deployment.yaml
       asserts:
         - equal:
             path: spec.template.spec.securityContext.runAsNonRoot
             value: true
         - equal:
             path: spec.template.spec.containers[0].securityContext.readOnlyRootFilesystem
             value: true
         - contains:
             path: spec.template.spec.containers[0].securityContext.capabilities.drop
             content: ALL
     - it: omits replicas when autoscaling is on
       template: deployment.yaml
       set:
         autoscaling.enabled: true
       asserts:
         - notExists:
             path: spec.replicas
     - it: uses the digest when provided
       template: deployment.yaml
       set:
         image.digest: sha256:0000000000000000000000000000000000000000000000000000000000000000
       asserts:
         - matchRegex:
             path: spec.template.spec.containers[0].image
             pattern: ^ghcr.io/example/api@sha256:0{64}$
     - it: fails when PDB has both minAvailable and maxUnavailable
       template: deployment.yaml
       set:
         podDisruptionBudget.minAvailable: 1
         podDisruptionBudget.maxUnavailable: 1
       asserts:
         - failedTemplate:
             errorPattern: set only one of minAvailable or maxUnavailable
   ```

   ```yaml
   # charts/api/tests/toggles_test.yaml
   suite: toggles
   templates:
     - hpa.yaml
     - pdb.yaml
     - networkpolicy.yaml
     - servicemonitor.yaml
   tests:
     - it: renders nothing optional by default except PDB and NetworkPolicy
       asserts:
         - hasDocuments: { count: 0 }
           template: hpa.yaml
         - hasDocuments: { count: 1 }
           template: pdb.yaml
         - hasDocuments: { count: 1 }
           template: networkpolicy.yaml
         - hasDocuments: { count: 0 }
           template: servicemonitor.yaml
     - it: renders a ServiceMonitor only when the CRD exists
       template: servicemonitor.yaml
       set:
         serviceMonitor.enabled: true
       capabilities:
         apiVersions: [monitoring.coreos.com/v1]
       asserts:
         - isKind: { of: ServiceMonitor }
   ```

   ```bash
   helm unittest charts/api
   ```

   Expected output (abridged):

   ```text
   ### Chart [ api ] charts/api

    PASS  deployment   charts/api/tests/deployment_test.yaml
    PASS  toggles      charts/api/tests/toggles_test.yaml

   Charts:      1 passed, 1 total
   Test Suites: 2 passed, 2 total
   Tests:       6 passed, 6 total
   ```

2. Add `ct` value files. `ct install` installs the chart once per file in `ci/`.

   ```bash
   mkdir -p charts/api/ci
   cat > charts/api/ci/default-values.yaml <<'EOF'
   migrations:
     enabled: false
   image:
     tag: "1.0.0"
   EOF
   cat > charts/api/ci/hpa-values.yaml <<'EOF'
   migrations:
     enabled: false
   image:
     tag: "1.0.0"
   autoscaling:
     enabled: true
   EOF
   ```

3. Initialise Git (ct compares against a target branch) and run `ct`.

   ```bash
   git init -q && git add -A && git commit -qm "api chart" && git branch -M main
   git checkout -qb feature/ct
   sed -i 's/^version: 0.1.0/version: 0.1.1/' charts/api/Chart.yaml
   git commit -qam "bump chart"
   cat > ct.yaml <<'EOF'
   target-branch: main
   chart-dirs: [charts]
   check-version-increment: true
   validate-maintainers: false
   helm-extra-args: --timeout 120s
   EOF
   ct lint --config ct.yaml
   ct install --config ct.yaml --namespace ct-test
   ```

   Expected output tail:

   ```text
   ------------------------------------------------------------------------------------------------------------------------
    ✔︎ api => (version: "0.1.1", path: "charts/api")
   ------------------------------------------------------------------------------------------------------------------------
   All charts installed successfully
   ```

   Revert the version bump and rerun `ct lint` to see `check-version-increment` fail: `Chart version not ok. Needs a version bump!`.

4. CI workflow.

   ```yaml
   # .github/workflows/chart-ci.yaml
   name: chart-ci
   on:
     pull_request:
       paths: ["charts/**"]
   permissions:
     contents: read
   jobs:
     test:
       runs-on: ubuntu-24.04
       steps:
         - uses: actions/checkout@v4
           with: { fetch-depth: 0 }
         - uses: azure/setup-helm@v4
           with: { version: v3.16.3 }
         - uses: actions/setup-python@v5
           with: { python-version: "3.12" }
         - uses: helm/chart-testing-action@v2
         - name: Lint
           run: ct lint --config ct.yaml
         - name: Unit tests
           run: |
             helm plugin install https://github.com/helm-unittest/helm-unittest --version v0.6.3
             helm unittest charts/api --output-type JUnit --output-file junit.xml
         - name: Schema validation
           run: |
             curl -fsSLo kc.tgz https://github.com/yannh/kubeconform/releases/download/v0.6.7/kubeconform-linux-amd64.tar.gz
             tar -xzf kc.tgz kubeconform
             for env in dev stage prod; do
               helm template api charts/api -f envs/$env/values.yaml --api-versions monitoring.coreos.com/v1 | ./kubeconform -strict -summary
             done
         - uses: helm/kind-action@v1
           with: { node_image: kindest/node:v1.31.2 }
         - name: Install tests
           run: ct install --config ct.yaml
   ```

### Verify

`helm unittest`, `ct lint` and `ct install` all pass locally; the workflow file passes `actionlint` if you have it.

### Cleanup

```bash
kubectl delete namespace ct-test --ignore-not-found
git checkout -q main
```

### What you learned

Unit tests pin security invariants so a refactor cannot silently drop `runAsNonRoot`; `ct` enforces version bumps and real installs; the same commands run on a laptop and in CI.

---

## Lab 5: Secrets with SOPS and the External Secrets Operator

**Objective.** Keep secrets out of plain values files two ways: encrypted values via helm-secrets/SOPS for cluster-level config, and an `ExternalSecret` rendered by the chart for application secrets.

**Prerequisites.** Lab 3 release running; `sops`, `age`, helm-secrets plugin.

**Time.** 40 minutes.

### Steps

1. Create a SOPS config and an encrypted values file.

   ```bash
   cat > .sops.yaml <<EOF
   creation_rules:
     - path_regex: envs/.*/secrets\.yaml$
       age: ${SOPS_AGE_RECIPIENTS}
   EOF
   cat > envs/dev/secrets.yaml <<'EOF'
   env:
     FEATURE_FLAGS_TOKEN: lab-placeholder-value
   EOF
   sops --encrypt --in-place envs/dev/secrets.yaml
   head -5 envs/dev/secrets.yaml
   ```

   The values are now `ENC[AES256_GCM,...]` strings; the file is safe to commit. In production, replace age with an AWS KMS key ARN (`kms:` rule) so CI decrypts via OIDC-assumed IAM, not a key file.

2. Deploy with transparent decryption.

   ```bash
   helm secrets upgrade --install api charts/api --namespace api \
     -f envs/dev/values.yaml -f secrets://envs/dev/secrets.yaml --wait
   kubectl exec --namespace api deploy/api -- env | grep FEATURE_FLAGS_TOKEN
   ```

   Caveat: the decrypted value is now in the release Secret and in the Pod env. SOPS protects the Git repo, not the cluster. Anyone with `get secrets` in the namespace can read it. That is acceptable for low-sensitivity config and wrong for database credentials, which is why step 3 exists.

3. Add an `ExternalSecret` template to the chart.

   ```yaml
   # charts/api/values.yaml (append)
   externalSecret:
     enabled: false
     # -- ClusterSecretStore name (e.g. aws-secrets-manager)
     storeRef: aws-secrets-manager
     refreshInterval: 1h
     # -- Remote key -> env var name
     data:
       DATABASE_URL: capstone/dev/api/database_url
   ```

   ```yaml
   # charts/api/templates/externalsecret.yaml
   {{- if .Values.externalSecret.enabled }}
   apiVersion: external-secrets.io/v1beta1
   kind: ExternalSecret
   metadata:
     name: {{ include "api.fullname" . }}
     labels: {{- include "api.labels" . | nindent 4 }}
   spec:
     refreshInterval: {{ .Values.externalSecret.refreshInterval }}
     secretStoreRef:
       kind: ClusterSecretStore
       name: {{ .Values.externalSecret.storeRef }}
     target:
       name: {{ include "api.fullname" . }}-secrets
       creationPolicy: Owner
     data:
       {{- range $env, $remote := .Values.externalSecret.data }}
       - secretKey: {{ $env }}
         remoteRef:
           key: {{ $remote }}
       {{- end }}
   {{- end }}
   ```

   And wire it in `envs/prod/values.yaml`:

   ```yaml
   externalSecret:
     enabled: true
   envFromSecrets:
     - name: api-secrets
   ```

4. Install ESO into kind with a fake provider to see the flow end to end without a cloud account.

   ```bash
   helm repo add external-secrets https://charts.external-secrets.io
   helm upgrade --install external-secrets external-secrets/external-secrets \
     --namespace external-secrets --create-namespace --version 0.10.5 --wait
   kubectl apply -f - <<'EOF'
   apiVersion: external-secrets.io/v1beta1
   kind: ClusterSecretStore
   metadata:
     name: fake
   spec:
     provider:
       fake:
         data:
           - key: capstone/dev/api/database_url
             value: postgresql://app:lab-placeholder@db.example.internal:5432/app
   EOF
   helm secrets upgrade --install api charts/api --namespace api \
     -f envs/dev/values.yaml -f secrets://envs/dev/secrets.yaml \
     --set externalSecret.enabled=true --set externalSecret.storeRef=fake \
     --set 'envFromSecrets[0].name=api-secrets' --wait
   kubectl get externalsecret,secret --namespace api
   ```

   Expected: `externalsecret.external-secrets.io/api` with `STATUS SecretSynced`, and `secret/api-secrets` created by ESO.

### Verify

```bash
kubectl exec --namespace api deploy/api -- sh -c 'echo $DATABASE_URL'
```

### Cleanup

```bash
kubectl delete clustersecretstore fake
helm uninstall external-secrets --namespace external-secrets
kubectl delete namespace external-secrets
```

### What you learned

SOPS encrypts values at rest in Git and is right for configuration; ESO keeps credentials in a managed secret store and the chart only references them; the chart never contains a secret literal in either path.

---

## Lab 6: Package, Push to an OCI Registry, Sign, and Install from It

**Objective.** Publish the chart as an OCI artifact to a local registry (standing in for GHCR/ECR), sign it with Cosign keyless-style using a local key for the lab, verify, and install from the registry.

**Prerequisites.** Lab 2 chart; Docker; `cosign` installed (`brew install cosign` or the release binary).

**Time.** 30 minutes.

### Steps

1. Run a local OCI registry reachable from your shell.

   ```bash
   docker run -d --restart=always -p 5001:5000 --name helm-registry registry:2
   ```

2. Package and push.

   ```bash
   helm package charts/api --destination dist
   helm push dist/api-0.1.1.tgz oci://localhost:5001/charts --plain-http
   ```

   Expected output:

   ```text
   Pushed: localhost:5001/charts/api:0.1.1
   Digest: sha256:7a1b...
   ```

3. Inspect it from the registry.

   ```bash
   helm show chart oci://localhost:5001/charts/api --version 0.1.1 --plain-http
   curl -s http://localhost:5001/v2/charts/api/tags/list
   ```

   Expected: `{"name":"charts/api","tags":["0.1.1"]}`.

4. Sign the chart artifact with Cosign. In CI you would use keyless signing with the GitHub Actions OIDC identity (`cosign sign --yes <ref>@<digest>`), which is what the capstone does; here use a throwaway key.

   ```bash
   COSIGN_PASSWORD="" cosign generate-key-pair
   DIGEST=$(crane digest localhost:5001/charts/api:0.1.1 2>/dev/null || \
     curl -sI -H 'Accept: application/vnd.oci.image.manifest.v1+json' http://localhost:5001/v2/charts/api/manifests/0.1.1 | awk -F': ' 'tolower($1)=="docker-content-digest"{print $2}' | tr -d '\r')
   COSIGN_PASSWORD="" cosign sign --key cosign.key --allow-insecure-registry --yes "localhost:5001/charts/api@${DIGEST}"
   cosign verify --key cosign.pub --allow-insecure-registry "localhost:5001/charts/api@${DIGEST}" | head -3
   ```

   Expected output starts with `Verification for localhost:5001/charts/api@sha256:... --` and `The following checks were performed on each of these signatures`.

5. Install from the registry.

   ```bash
   helm uninstall api --namespace api
   helm install api oci://localhost:5001/charts/api --version 0.1.1 --plain-http \
     --namespace api -f envs/dev/values.yaml --wait
   helm list --namespace api
   ```

   The `CHART` column reads `api-0.1.1`, and `helm get metadata api -n api` shows the chart came from the OCI source.

### Verify

```bash
helm history api --namespace api
curl -s http://api.127.0.0.1.nip.io:8080/healthz
```

### Cleanup (entire lab series)

```bash
helm uninstall api --namespace api
kubectl delete namespace api
helm uninstall metrics-server --namespace kube-system
helm uninstall ingress-nginx --namespace ingress-nginx
kubectl delete namespace ingress-nginx
docker rm -f helm-registry
rm -f cosign.key cosign.pub
kind delete cluster --name helm-lab
```

### What you learned

OCI is the distribution path: chart version becomes the tag, the artifact has a digest, and Cosign signs the digest exactly as for container images. From here the chart reference `oci://ghcr.io/<ORG>/charts/api` plus a values file is all Argo CD needs; continue in [../argocd/04-hands-on-labs.md](../argocd/04-hands-on-labs.md).
