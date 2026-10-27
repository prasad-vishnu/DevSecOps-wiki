# OWASP ZAP: Installation and Setup

| | |
|---|---|
| **Cluster** | 08 DevSecOps |
| **Role** | Anchor (Gate 6 DAST) |
| **Prerequisites** | Docker 24+, a running copy of the capstone `inventory-api` on `http://localhost:8000` (or any app you own) |
| **Target version** | ZAP 2.15+, image `ghcr.io/zaproxy/zaproxy:stable` |
| **Capstone phase** | Phase 5 (Months 17-20) |

> After this file you will have ZAP running three ways (Docker packaged scan, desktop UI, headless daemon with API key), understand which image tag to use where, and have produced your first baseline report against your own service on localhost.

---

## 1. Choose How to Run ZAP

| Mode | Use when | Command shape |
|---|---|---|
| Docker packaged scan | CI, quick local checks | `docker run ghcr.io/zaproxy/zaproxy:stable zap-baseline.py -t <URL>` |
| Docker Automation Framework | CI with auth, spec import, custom policy | `docker run ... zap.sh -cmd -autorun /zap/wrk/zap.yaml` |
| Docker daemon + API | Driving ZAP from Python/pytest or another container | `docker run ... zap.sh -daemon -port 8080 -config api.key=<API_KEY>` |
| Desktop UI | Learning, building contexts/auth interactively, exporting to AF YAML | Installer or `zap.sh` |
| Desktop via Docker | Linux with X forwarding, when you want the UI but not an install | `zap-webswing.sh` (browser-based UI on :8080) |

The images:

| Tag | Contents | Use |
|---|---|---|
| `ghcr.io/zaproxy/zaproxy:stable` | Latest release plus release-maturity add-ons, Python packaged scan scripts, Firefox for the AJAX spider | Default for CI |
| `ghcr.io/zaproxy/zaproxy:weekly` | Weekly build with beta/alpha add-ons | When you need a new rule before release |
| `ghcr.io/zaproxy/zaproxy:bare` | Minimal, no scripts, no browser | Building your own image on top |
| `ghcr.io/zaproxy/zaproxy:nightly` | Development build | Testing fixes only |

Pin the stable tag by digest in reproducible pipelines; the tag itself moves.

---

## 2. Docker Install (All Platforms)

```bash
docker pull ghcr.io/zaproxy/zaproxy:stable
docker run --rm ghcr.io/zaproxy/zaproxy:stable zap.sh -version
```

Expected output (version will vary):

```text
ZAP 2.15.0
```

Confirm the packaged scripts are present:

```bash
docker run --rm ghcr.io/zaproxy/zaproxy:stable ls /zap | grep -E 'zap-(baseline|full-scan|api-scan)\.py'
```

```text
zap-api-scan.py
zap-baseline.py
zap-full-scan.py
```

### Mounting a working directory

Reports are written inside the container to `/zap/wrk/`. Mount a host directory there and make it writable by the `zap` user (uid 1000):

```bash
mkdir -p ./zap-out && chmod 777 ./zap-out    # lab shortcut; see note
docker run --rm -v "$(pwd)/zap-out:/zap/wrk/:rw" ghcr.io/zaproxy/zaproxy:stable \
  zap-baseline.py -t http://host.docker.internal:8000 -r baseline.html
```

`chmod 777` is a convenience for the lab. In CI runners the checkout directory is already writable by uid 1000 on GitHub-hosted Ubuntu runners; on self-hosted runners use `--user $(id -u)` or `chown 1000`.

### Reaching an app on the host

| Platform | Address for "localhost on the host" from inside the container |
|---|---|
| Docker Desktop (macOS, Windows) | `http://host.docker.internal:8000` |
| Linux Docker Engine | `--add-host=host.docker.internal:host-gateway` then `http://host.docker.internal:8000`, or `--network host` and `http://localhost:8000` |
| kind cluster | Port-forward the service to the host first, or run ZAP as a Kubernetes Job inside the cluster and target `http://inventory-api.capstone.svc:8000` |

---

## 3. Start the Capstone Target

Any application you own works. The capstone's FastAPI service, run from its repository:

```bash
git clone https://github.com/example-org/capstone.git
cd capstone/services/inventory-api
docker compose up -d        # starts inventory-api on :8000 and postgres
curl -s http://localhost:8000/healthz
```

```text
{"status":"ok"}
```

OpenAPI is served at `http://localhost:8000/openapi.json`. If you are instead running in kind:

```bash
kubectl --namespace capstone port-forward svc/inventory-api 8000:8000 &
```

---

## 4. First Baseline Scan

```bash
docker run --rm --add-host=host.docker.internal:host-gateway \
  -v "$(pwd)/zap-out:/zap/wrk/:rw" \
  ghcr.io/zaproxy/zaproxy:stable zap-baseline.py \
  -t http://host.docker.internal:8000 \
  -r baseline.html -J baseline.json -w baseline.md
```

Expected output (abridged):

```text
2026-10-27 10:02:11,431 Params: ['zap-x.sh', '-daemon', '-port', '40165', '-host', '0.0.0.0', '-config', 'api.disablekey=true', ...]
Total of 14 URLs
PASS: Vulnerable JS Library (Powered by Retire.js) [10003]
PASS: Cookie No HttpOnly Flag [10010]
PASS: Cookie Without Secure Flag [10011]
...
WARN-NEW: Content Security Policy (CSP) Header Not Set [10038] x 6
    http://host.docker.internal:8000/ (200 OK)
    http://host.docker.internal:8000/docs (200 OK)
    ...
WARN-NEW: Missing Anti-clickjacking Header [10020] x 3
WARN-NEW: X-Content-Type-Options Header Missing [10021] x 6
WARN-NEW: Permissions Policy Header Not Set [10063] x 6
WARN-NEW: Server Leaks Version Information via "Server" HTTP Response Header Field [10036] x 14
FAIL-NEW: 0     FAIL-INPROG: 0  WARN-NEW: 5     WARN-INPROG: 0  INFO: 0 IGNORE: 0       PASS: 55
```

Exit code semantics for every packaged scan:

| Exit code | Meaning |
|---|---|
| 0 | No WARN or FAIL (or `-I` was passed and there were only WARNs) |
| 1 | At least one FAIL |
| 2 | At least one WARN (and `-I` not set) |
| 3 | ZAP itself failed (could not start, target unreachable, bad arguments) |

Open `zap-out/baseline.html` in a browser. Each alert has description, evidence, CWE id, WASC id, solution, and reference links.

---

## 5. Desktop Install (Optional but Recommended for Learning)

| OS | Install |
|---|---|
| Windows | Download the installer from https://www.zaproxy.org/download/ (requires Java 17+; the installer bundles it) or `winget install ZAP.ZAP` |
| macOS | `brew install --cask zap` |
| Linux | `.tar.gz` from the download page, `./zap.sh`; Flatpak/Snap also available |

First run prompts about persisting the session; choose "No, I do not want to persist this session" for exploration. Set the local proxy under Tools, Options, Network, Local Servers/Proxies (default `localhost:8080`). The desktop ships a pre-configured browser launcher (Firefox/Chrome via the Selenium add-on) that trusts the ZAP root CA, so manual explore works without certificate juggling.

### Export the ZAP CA for your own browser

Tools, Options, Network, Server Certificates, Save. Import `zap_root_ca.cer` into the browser's authority store. Only install this certificate in a test browser profile; it allows ZAP to decrypt TLS for that profile.

---

## 6. Headless Daemon with API Key

For driving ZAP from Python or from pytest, start it as a daemon:

```bash
docker run --rm -d --name zap -p 8090:8080 \
  --add-host=host.docker.internal:host-gateway \
  -v "$(pwd)/zap-out:/zap/wrk/:rw" \
  ghcr.io/zaproxy/zaproxy:stable zap.sh -daemon -host 0.0.0.0 -port 8080 \
  -config api.key=<ZAP_API_KEY> \
  -config api.addrs.addr.name=.* \
  -config api.addrs.addr.regex=true
```

- `api.key` protects every mutating API call. Generate it in the pipeline (`openssl rand -hex 16`) and pass it as a secret; never hardcode it.
- `api.addrs.addr.*` allows callers other than `127.0.0.1` (needed because the container's localhost is not yours).

Verify:

```bash
curl -s "http://localhost:8090/JSON/core/view/version/?apikey=<ZAP_API_KEY>"
```

```text
{"version":"2.15.0"}
```

Install the Python client:

```bash
python -m pip install zaproxy
python - <<'EOF'
from zapv2 import ZAPv2
zap = ZAPv2(apikey="<ZAP_API_KEY>", proxies={"http": "http://localhost:8090", "https": "http://localhost:8090"})
print(zap.core.version)
EOF
```

Stop it with `docker stop zap`.

---

## 7. Automation Framework Smoke Test

Create `zap-out/zap.yaml`:

```yaml
env:
  contexts:
    - name: inventory-api
      urls:
        - http://host.docker.internal:8000
      includePaths:
        - http://host.docker.internal:8000/.*
  parameters:
    failOnError: true
    failOnWarning: false
    progressToStdout: true
jobs:
  - type: passiveScan-config
    parameters:
      maxAlertsPerRule: 10
  - type: openapi
    parameters:
      apiUrl: http://host.docker.internal:8000/openapi.json
      targetUrl: http://host.docker.internal:8000
      context: inventory-api
  - type: passiveScan-wait
    parameters:
      maxDuration: 5
  - type: report
    parameters:
      template: traditional-html
      reportDir: /zap/wrk
      reportFile: af-smoke
      reportTitle: inventory-api AF smoke
  - type: exitStatus
    parameters:
      errorLevel: High
      warnLevel: Medium
```

Run it:

```bash
docker run --rm --add-host=host.docker.internal:host-gateway \
  -v "$(pwd)/zap-out:/zap/wrk/:rw" \
  ghcr.io/zaproxy/zaproxy:stable zap.sh -cmd -autorun /zap/wrk/zap.yaml
```

Expected tail of output:

```text
Job openapi started
Job openapi finished, time taken: 00:00:02
Job passiveScan-wait started
Job passiveScan-wait finished, time taken: 00:00:04
Job report started
Job report finished, time taken: 00:00:01
Job exitStatus started
Job exitStatus finished, time taken: 00:00:00
Automation plan succeeded!
```

`echo $?` prints `0`, or `2` if any Medium alert existed (the `warnLevel`), or `1` on High.

---

## 8. Running ZAP Inside kind as a Job

When the target is only reachable inside the cluster, run ZAP there:

```yaml
# zap-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: zap-baseline
  namespace: capstone
spec:
  backoffLimit: 0
  template:
    spec:
      restartPolicy: Never
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 1000
      containers:
        - name: zap
          image: ghcr.io/zaproxy/zaproxy:stable
          command: ["zap-baseline.py"]
          args: ["-t", "http://inventory-api.capstone.svc:8000", "-J", "/zap/wrk/report.json", "-I"]
          resources:
            requests: { cpu: "500m", memory: "1Gi" }
            limits: { cpu: "2", memory: "3Gi" }
          volumeMounts:
            - name: wrk
              mountPath: /zap/wrk
      volumes:
        - name: wrk
          emptyDir: {}
```

```bash
kubectl apply -f zap-job.yaml
kubectl --namespace capstone wait --for=condition=complete job/zap-baseline --timeout=600s
kubectl --namespace capstone logs job/zap-baseline | tail -n 5
```

The `emptyDir` disappears with the pod; in the labs you will copy the report out with `kubectl cp` before the pod is deleted or write to a PVC.

---

## 9. Verification Checklist

| Check | Command | Expected |
|---|---|---|
| Image runs | `docker run --rm ghcr.io/zaproxy/zaproxy:stable zap.sh -version` | `ZAP 2.x` |
| Target reachable from container | `docker run --rm --add-host=host.docker.internal:host-gateway curlimages/curl -s http://host.docker.internal:8000/healthz` | `{"status":"ok"}` |
| Baseline writes report | `ls zap-out/baseline.html` | File exists |
| Daemon API answers | `curl -s "http://localhost:8090/JSON/core/view/version/?apikey=<ZAP_API_KEY>"` | JSON version |
| AF plan runs | `zap.sh -cmd -autorun /zap/wrk/zap.yaml` | `Automation plan succeeded!` |

---

## 10. Security Notes for the Setup

- The ZAP root CA you export is a private CA capable of intercepting TLS for any browser that trusts it. Keep it in a dedicated test profile; delete it when done.
- The API key is a secret: store in GitHub Actions secrets or Vault ([../../gate-7-runtime-and-secrets/vault/](../../gate-7-runtime-and-secrets/vault/README.md)), inject at runtime.
- `api.disablekey=true` (which the packaged scans set internally on a random port bound to the container) is acceptable only because nothing else can reach that port. Never expose a key-less daemon on a routable interface.
- Everything above targeted `localhost` or a service in a cluster you own. Keep it that way.

Next: [02-core-concepts.md](02-core-concepts.md).
