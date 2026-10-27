# OWASP ZAP: Hands-On Labs

| | |
|---|---|
| **Cluster** | 08 DevSecOps |
| **Role** | Anchor (Gate 6 DAST) |
| **Prerequisites** | [01-installation-and-setup.md](01-installation-and-setup.md), [03-commands-and-cheatsheet.md](03-commands-and-cheatsheet.md), Docker, kind, kubectl, the capstone repo |
| **Target version** | ZAP 2.15+ |
| **Capstone phase** | Phase 5 (Months 17-20) |

> Seven progressive labs. Every target is your own capstone service (`inventory-api` on FastAPI, `orders-service` on Spring Boot) running on `localhost` via Docker Compose or in the `capstone` kind cluster. Do not point any of these commands at a system you do not own or have written authorisation to test. Labs 1-3 are passive or spec-driven; Labs 4-7 are active scans and must only run against the disposable lab environment.

Lab environment setup (once):

```bash
git clone https://github.com/example-org/capstone.git && cd capstone
make kind-up                                  # kind create cluster --name capstone + Argo CD bootstrap
make deploy-local                             # inventory-api :8000, orders-service :8080, postgres
kubectl --namespace capstone port-forward svc/inventory-api 8000:8000 >/dev/null 2>&1 &
kubectl --namespace capstone port-forward svc/orders-service 8080:8080 >/dev/null 2>&1 &
curl -s localhost:8000/healthz; curl -s localhost:8080/actuator/health
mkdir -p zap-out && chmod 777 zap-out
export ZAP="docker run --rm --add-host=host.docker.internal:host-gateway -v $(pwd)/zap-out:/zap/wrk/:rw ghcr.io/zaproxy/zaproxy:stable"
export T=http://host.docker.internal:8000 S=http://host.docker.internal:8080
```

If you do not have the capstone repo yet, any FastAPI/Spring Boot app of your own on those ports works.

---

## Lab 1: Baseline Scan and Reading a Report

**Objective.** Run a passive baseline scan against `inventory-api`, read every alert, and classify each as fix / accept / false positive.

**Prerequisites.** Environment above.

**Time.** 30 minutes.

### Steps

1. Run the baseline with all four report formats.

   ```bash
   $ZAP zap-baseline.py -t $T -r lab1.html -J lab1.json -x lab1.xml -w lab1.md -m 2 -j
   echo "exit=$?"
   ```

   Expected output (abridged):

   ```text
   Total of 23 URLs
   WARN-NEW: Content Security Policy (CSP) Header Not Set [10038] x 4
   WARN-NEW: Missing Anti-clickjacking Header [10020] x 2
   WARN-NEW: X-Content-Type-Options Header Missing [10021] x 9
   WARN-NEW: Server Leaks Version Information via "Server" HTTP Response Header Field [10036] x 23
   WARN-NEW: Permissions Policy Header Not Set [10063] x 4
   WARN-NEW: Storable and Cacheable Content [10049] x 11
   WARN-NEW: Re-examine Cache-control Directives [10015] x 6
   FAIL-NEW: 0     FAIL-INPROG: 0  WARN-NEW: 7     WARN-INPROG: 0  INFO: 0 IGNORE: 0       PASS: 53
   exit=2
   ```

2. Open `zap-out/lab1.html`. For each WARN read: description, the first instance URL, evidence, solution, CWE.

3. Extract the URLs ZAP found.

   ```bash
   jq -r '.site[].alerts[].instances[].uri' zap-out/lab1.json | sort -u
   ```

   You should see `/docs`, `/openapi.json`, `/redoc`, `/healthz`, and whatever the spider parsed from `/docs`. Note that none of the real API routes (`/items`, `/orders`) appear: the spider cannot discover JSON API endpoints. Lab 2 fixes this.

4. Write the triage decision in `zap-out/rules.tsv`:

   ```text
   10036	WARN	(Server header - fix: uvicorn --no-server-header)
   10038	INFO	(CSP - API returns JSON only; /docs is Swagger UI, accepted)
   10020	INFO	(Clickjacking - API only)
   10063	INFO	(Permissions Policy - API only)
   10049	IGNORE	(Cacheable - static Swagger assets)
   10015	IGNORE	(Cache-control - static assets)
   10021	FAIL	(X-Content-Type-Options must be set)
   ```

5. Re-run with the rules file.

   ```bash
   $ZAP zap-baseline.py -t $T -c rules.tsv -r lab1b.html -m 2 -j -s
   echo "exit=$?"
   ```

   Expected: `FAIL-NEW: 1 ... WARN-NEW: 1` and `exit=1`. The build would now fail on the missing `X-Content-Type-Options`.

### Verify

`grep -c FAIL-NEW zap-out/lab1b.html` is non-zero, and the FAIL is rule 10021 only.

### Cleanup

Keep `rules.tsv`; delete the reports: `rm zap-out/lab1*`.

### What you learned

The spider's view of an API is tiny. Tuning is a declared decision per rule, not a global off switch. Exit codes map straight onto a CI gate.

---

## Lab 2: Fix the Finding, Then Spec-Driven API Scan

**Objective.** Add the missing header in `inventory-api`, verify ZAP passes, then import the OpenAPI spec and see real route coverage.

**Prerequisites.** Lab 1.

**Time.** 40 minutes.

### Steps

1. Add security headers middleware in `services/inventory-api/app/main.py`:

   ```python
   from fastapi import FastAPI, Request

   app = FastAPI(title="inventory-api")

   @app.middleware("http")
   async def security_headers(request: Request, call_next):
       response = await call_next(request)
       response.headers["X-Content-Type-Options"] = "nosniff"
       response.headers["X-Frame-Options"] = "DENY"
       response.headers["Referrer-Policy"] = "no-referrer"
       response.headers["Cache-Control"] = "no-store"
       response.headers.setdefault("Content-Security-Policy", "default-src 'none'; frame-ancestors 'none'")
       return response
   ```

   And start uvicorn with `--no-server-header` in the Dockerfile `CMD`. Rebuild and redeploy (`make deploy-local`).

2. Re-run Lab 1's final command. Expected: `FAIL-NEW: 0 WARN-NEW: 0` and `exit=0`. (Swagger UI at `/docs` may now break under the strict CSP; FastAPI's docs need `'self'` plus the CDN. Either serve the docs only in non-production or loosen CSP for `/docs` only. This trade-off is the kind of thing DAST is for.)

3. Run the API scan in safe mode (no active rules) to measure coverage.

   ```bash
   $ZAP zap-api-scan.py -t $T/openapi.json -f openapi -O $T -S -J lab2.json -r lab2.html
   jq -r '.site[].alerts[].instances[].uri' zap-out/lab2.json | sort -u | head -n 20
   ```

   Expected: the real routes appear.

   ```text
   http://host.docker.internal:8000/items
   http://host.docker.internal:8000/items/1
   http://host.docker.internal:8000/orders
   http://host.docker.internal:8000/users/me
   ...
   ```

4. Inspect the console: almost every route returned `401`. ZAP is unauthenticated. Note which rules fired anyway (likely only header rules). Lab 3 adds a token.

5. Repeat for Spring Boot:

   ```bash
   $ZAP zap-api-scan.py -t $S/v3/api-docs -f openapi -O $S -S -r lab2-orders.html
   ```

   Look for alert 10036/10037 (`X-Powered-By`), and check whether `/actuator/env` or `/actuator/heapdump` appeared in the site tree: springdoc does not list them, but the spider started by the API scan may find `/actuator` from the `/actuator` discovery page if it is exposed. If so, that is a High-priority fix (`management.endpoints.web.exposure.include=health,prometheus`).

### Verify

`jq '[.site[].alerts[].instances[].uri] | unique | length' zap-out/lab2.json` is greater than 15.

### Cleanup

`rm zap-out/lab2*`.

### What you learned

Spec import is the coverage mechanism for APIs. Unauthenticated scans mostly test the auth middleware. Headers are cheap wins that DAST proves across every route.

---

## Lab 3: Authenticated Scan with a Bearer Token

**Objective.** Mint a token for a scan-only user, inject it with the Replacer, confirm authenticated coverage.

**Prerequisites.** Lab 2. The capstone seeds a user `scan-user` in the local environment (see `services/inventory-api/seed.py`).

**Time.** 30 minutes.

### Steps

1. Mint a token (the capstone exposes a password grant for local dev only).

   ```bash
   TOKEN=$(curl -s -X POST localhost:8000/auth/token \
     -H 'Content-Type: application/json' \
     -d '{"username":"scan-user","password":"<SCAN_PASSWORD>"}' | jq -r .access_token)
   echo ${TOKEN:0:12}...
   ```

2. Run the API scan with the Replacer injecting the header. Still safe mode.

   ```bash
   $ZAP zap-api-scan.py -t $T/openapi.json -f openapi -O $T -S -J lab3.json -r lab3.html \
     -z "-config replacer.full_list(0).description=auth \
         -config replacer.full_list(0).enabled=true \
         -config replacer.full_list(0).matchtype=REQ_HEADER \
         -config replacer.full_list(0).matchstr=Authorization \
         -config replacer.full_list(0).regex=false \
         -config replacer.full_list(0).replacement='Bearer $TOKEN'"
   ```

3. Confirm the responses were 200s, not 401s.

   ```bash
   jq -r '.site[].alerts[] | select(.pluginid=="10049" or .pluginid=="10015") | .instances[].uri' zap-out/lab3.json | sort -u | wc -l
   ```

   More robust: run the daemon and query history (optional):

   ```bash
   docker run -d --rm --name zap -p 8090:8080 --add-host=host.docker.internal:host-gateway ghcr.io/zaproxy/zaproxy:stable \
     zap.sh -daemon -host 0.0.0.0 -port 8080 -config api.key=<ZAP_API_KEY> -config api.addrs.addr.name=.* -config api.addrs.addr.regex=true
   sleep 15
   curl -s "http://localhost:8090/JSON/replacer/action/addRule/?apikey=<ZAP_API_KEY>&description=auth&enabled=true&matchType=REQ_HEADER&matchRegex=false&matchString=Authorization&replacement=Bearer%20$TOKEN"
   curl -s "http://localhost:8090/JSON/openapi/action/importUrl/?apikey=<ZAP_API_KEY>&url=$T/openapi.json&hostOverride=$T"
   sleep 10
   curl -s "http://localhost:8090/JSON/core/view/messages/?apikey=<ZAP_API_KEY>&baseurl=$T&start=0&count=200" \
     | jq -r '.messages[] | .responseHeader | split("\r\n")[0]' | sort | uniq -c
   docker stop zap
   ```

   Expected:

   ```text
      31 HTTP/1.1 200 OK
       4 HTTP/1.1 201 Created
       6 HTTP/1.1 404 Not Found
       2 HTTP/1.1 422 Unprocessable Entity
   ```

   The 404s are spec-generated ids that do not exist; the 422s are schema validation. No 401s means auth works.

### Verify

Zero `401` in the status histogram.

### Cleanup

`docker stop zap 2>/dev/null; rm zap-out/lab3*; unset TOKEN`.

### What you learned

The Replacer is the simplest auth mechanism when the pipeline can mint a token. The token's TTL must exceed the scan; there is no re-login.

---

## Lab 4: Automation Framework with JSON Login, Verification, and Active Scan

**Objective.** Replace flags with a declarative `zap.yaml` that logs in, verifies session, imports the spec, runs a scoped active scan, filters a known false positive, writes HTML + XML + SARIF, and sets the exit code.

**Prerequisites.** Lab 3. **This lab is an active scan: run it only against the local lab environment.**

**Time.** 60 minutes.

### Steps

1. Create `zap-out/zap-inventory.yaml`:

   ```yaml
   env:
     contexts:
       - name: inventory-api
         urls: ["${BASE_URL}"]
         includePaths: ["${BASE_URL}/.*"]
         excludePaths:
           - "${BASE_URL}/metrics"
           - "${BASE_URL}/auth/logout.*"
           - "${BASE_URL}/docs.*"
           - "${BASE_URL}/redoc.*"
         technology:
           include: [Language.Python, Db.PostgreSQL, OS.Linux, WS.Nginx]
         authentication:
           method: json
           parameters:
             loginRequestUrl: "${BASE_URL}/auth/token"
             loginRequestBody: '{"username":"{%username%}","password":"{%password%}"}'
           verification:
             method: poll
             pollUrl: "${BASE_URL}/users/me"
             pollFrequency: 60
             pollUnits: requests
             loggedInRegex: '"username"'
             loggedOutRegex: '\Q"detail":"Not authenticated"\E'
         sessionManagement:
           method: headers
           parameters:
             Authorization: "Bearer {%json:access_token%}"
         users:
           - name: scan-user
             credentials:
               username: "${SCAN_USER}"
               password: "${SCAN_PASSWORD}"
     parameters:
       failOnError: true
       failOnWarning: false
       progressToStdout: true
   jobs:
     - type: passiveScan-config
       parameters:
         maxAlertsPerRule: 10
         scanOnlyInScope: true
     - type: openapi
       parameters:
         apiUrl: "${BASE_URL}/openapi.json"
         targetUrl: "${BASE_URL}"
         context: inventory-api
         user: scan-user
     - type: spider
       parameters:
         context: inventory-api
         user: scan-user
         maxDuration: 2
       tests:
         - name: "coverage: at least 15 URLs"
           type: stats
           statistic: automation.spider.urls.added
           operator: ">="
           value: 15
           onFail: error
     - type: passiveScan-wait
       parameters:
         maxDuration: 5
     - type: activeScan-config
       parameters:
         maxRuleDurationInMins: 3
         maxScanDurationInMins: 20
         threadPerHost: 4
         handleAntiCSRFTokens: false
         inputVectors:
           postData:
             enabled: true
             json:
               enabled: true
               scanNullValues: false
           urlQueryStringAndDataDrivenNodes:
             enabled: true
           urlPath: false
           httpHeaders:
             enabled: false
           cookieData:
             enabled: false
     - type: activeScan
       parameters:
         context: inventory-api
         user: scan-user
         policy: capstone-api
       policyDefinition:
         defaultStrength: Low
         defaultThreshold: Off
         rules:
           - { id: 40018, name: "SQL Injection", strength: Medium, threshold: Medium }
           - { id: 40019, name: "SQL Injection - MySQL", strength: Low, threshold: Off }
           - { id: 40022, name: "SQL Injection - PostgreSQL", strength: Medium, threshold: Medium }
           - { id: 40012, name: "Cross Site Scripting (Reflected)", strength: Low, threshold: Medium }
           - { id: 90020, name: "Remote OS Command Injection", strength: Low, threshold: Medium }
           - { id: 6,     name: "Path Traversal", strength: Low, threshold: Medium }
           - { id: 40003, name: "CRLF Injection", strength: Low, threshold: Medium }
           - { id: 20019, name: "External Redirect", strength: Low, threshold: Medium }
           - { id: 90034, name: "Cloud Metadata Potentially Exposed", strength: Low, threshold: Medium }
           - { id: 90019, name: "Server Side Code Injection", strength: Low, threshold: Medium }
           - { id: 40009, name: "Server Side Include", strength: Low, threshold: Medium }
           - { id: 10104, name: "User Agent Fuzzer", strength: Low, threshold: Off }
     - type: alertFilter
       parameters:
         deleteGlobalAlerts: false
       alertFilters:
         - ruleId: 10096
           newRisk: False Positive
           context: inventory-api
           urlRegex: ".*/version"
         - ruleId: 10038
           newRisk: Info
           context: inventory-api
     - type: report
       alwaysRun: true
       parameters: { template: traditional-html, reportDir: /zap/wrk, reportFile: lab4, reportTitle: "inventory-api AF" }
     - type: report
       alwaysRun: true
       parameters: { template: traditional-xml, reportDir: /zap/wrk, reportFile: lab4 }
     - type: report
       alwaysRun: true
       parameters: { template: sarif-json, reportDir: /zap/wrk, reportFile: lab4 }
     - type: exitStatus
       alwaysRun: true
       parameters:
         errorLevel: High
         warnLevel: Medium
   ```

2. Run it with secrets from the environment.

   ```bash
   docker run --rm --add-host=host.docker.internal:host-gateway -v $(pwd)/zap-out:/zap/wrk/:rw \
     -e BASE_URL=$T -e SCAN_USER=scan-user -e SCAN_PASSWORD='<SCAN_PASSWORD>' \
     ghcr.io/zaproxy/zaproxy:stable zap.sh -cmd -autorun /zap/wrk/zap-inventory.yaml
   echo "exit=$?"
   ```

   Expected (abridged):

   ```text
   Job openapi started
   Job openapi finished, time taken: 00:00:03
   Job spider started
   Job spider found 27 URLs
   Job spider finished, time taken: 00:00:41
   Job passiveScan-wait finished, time taken: 00:00:05
   Job activeScan started
   Active Scan progress: 100%
   Job activeScan finished, time taken: 00:06:12
   Job alertFilter started
   Job alertFilter finished, time taken: 00:00:00
   Job report finished ...
   Job exitStatus started
   Job exitStatus finished, time taken: 00:00:00
   Automation plan succeeded!
   exit=0
   ```

   If the capstone's intentionally vulnerable branch `lab/sqli` is checked out instead, expect `Job exitStatus: High alert found: SQL Injection - PostgreSQL` and `exit=1`.

3. Open `lab4.html` and verify alerts show `Authorization` evidence (requests were authenticated). Open `lab4.sarif` and check `runs[0].results[].ruleId`.

### Verify

```bash
ls zap-out/lab4.html zap-out/lab4.xml zap-out/lab4.sarif
jq '.runs[0].results | length' zap-out/lab4.sarif
```

### Cleanup

`rm zap-out/lab4.*`. Keep `zap-inventory.yaml`; it becomes `.zap/zap-inventory.yaml` in the repo.

### What you learned

The AF encodes scope, auth, coverage assertions, policy, triage, and output in one reviewable file. Policy `defaultThreshold: Off` plus an explicit allow list is the fastest way to a predictable active scan.

---

## Lab 5: ZAP in GitHub Actions with Review Environment, SARIF, and PR Issue

**Objective.** Wire the baseline into a PR workflow with a per-PR kind namespace, upload SARIF to code scanning, and open an issue on WARN/FAIL.

**Prerequisites.** Lab 4 committed as `.zap/zap-inventory.yaml` and `.zap/rules.tsv`; a GitHub repository you control.

**Time.** 60 minutes.

### Steps

1. Create `.github/workflows/gate-6-dast.yaml`:

   ```yaml
   name: gate-6-dast
   on:
     pull_request:
       paths: ["services/**", ".zap/**"]
   permissions:
     contents: read
     issues: write
     security-events: write
   concurrency:
     group: dast-${{ github.ref }}
     cancel-in-progress: true
   jobs:
     dast-baseline:
       runs-on: ubuntu-latest
       timeout-minutes: 25
       steps:
         - uses: actions/checkout@v4
         - name: Create kind cluster
           uses: helm/kind-action@v1
           with:
             cluster_name: capstone
         - name: Deploy PR image of inventory-api
           run: |
             kubectl create namespace pr-${{ github.event.number }}
             helm upgrade --install inventory-api ./charts/inventory-api \
               --namespace pr-${{ github.event.number }} \
               --set image.tag=pr-${{ github.event.number }} \
               --set seedData=true --wait --timeout 5m
             kubectl --namespace pr-${{ github.event.number }} port-forward svc/inventory-api 8000:8000 &
             sleep 5; curl -sf localhost:8000/healthz
         - name: Mint scan token
           id: tok
           run: |
             TOKEN=$(curl -s -X POST localhost:8000/auth/token -H 'Content-Type: application/json' \
               -d '{"username":"scan-user","password":"${{ secrets.DAST_SCAN_PASSWORD }}"}' | jq -r .access_token)
             echo "::add-mask::$TOKEN"
             echo "token=$TOKEN" >> "$GITHUB_OUTPUT"
         - name: ZAP baseline
           uses: zaproxy/action-baseline@v0.14.0
           with:
             target: http://localhost:8000
             docker_name: ghcr.io/zaproxy/zaproxy:stable
             rules_file_name: .zap/rules.tsv
             cmd_options: >-
               -a -j -m 2 -x zap-report.xml -I
               -z "-config replacer.full_list(0).description=auth -config replacer.full_list(0).enabled=true -config replacer.full_list(0).matchtype=REQ_HEADER -config replacer.full_list(0).matchstr=Authorization -config replacer.full_list(0).regex=false -config replacer.full_list(0).replacement='Bearer ${{ steps.tok.outputs.token }}'"
             allow_issue_writing: true
             issue_title: "ZAP baseline findings: PR #${{ github.event.number }}"
             token: ${{ secrets.GITHUB_TOKEN }}
             fail_action: false
             artifact_name: zap-baseline-pr-${{ github.event.number }}
         - name: ZAP AF (SARIF)
           run: |
             docker run --rm --network host -v "$PWD:/zap/wrk/:rw" \
               -e BASE_URL=http://localhost:8000 -e SCAN_USER=scan-user -e SCAN_PASSWORD='${{ secrets.DAST_SCAN_PASSWORD }}' \
               ghcr.io/zaproxy/zaproxy:stable zap.sh -cmd -autorun /zap/wrk/.zap/zap-inventory-pr.yaml || true
         - name: Upload SARIF
           uses: github/codeql-action/upload-sarif@v3
           with:
             sarif_file: zap-pr.sarif
             category: zap-inventory-api
         - name: Upload to DefectDojo
           if: always()
           run: |
             curl -sf -X POST "${{ vars.DEFECTDOJO_URL }}/api/v2/reimport-scan/" \
               -H "Authorization: Token ${{ secrets.DEFECTDOJO_API_KEY }}" \
               -F scan_type="ZAP Scan" -F file=@zap-report.xml \
               -F product_type_name="Capstone" -F product_name="inventory-api" \
               -F engagement_name="${{ github.head_ref }}" -F auto_create_context=true \
               -F branch_tag="${{ github.head_ref }}" -F commit_hash="${{ github.sha }}" \
               -F build_id="${{ github.run_id }}" -F close_old_findings=true -F test_title="ZAP baseline PR"
   ```

   `.zap/zap-inventory-pr.yaml` is Lab 4's plan with the `activeScan` job removed and the SARIF `reportFile: zap-pr`. On GitHub-hosted runners `--network host` lets the container reach the port-forward on `localhost`.

2. Push a branch, open a PR, watch the job. Expected: a check named `gate-6-dast / dast-baseline`, an artifact with the reports, an issue titled "ZAP baseline findings: PR #N" if any WARN, and alerts in Security, Code scanning with category `zap-inventory-api`.

3. Introduce a regression (comment out the `X-Content-Type-Options` line), push, and observe `FAIL-NEW: 1` in the step log. Because `fail_action: false` the step itself passes; add a follow-up step that reads the exit code if you want a hard block:

   ```yaml
         - name: Gate on FAIL
           run: |
             if grep -q 'FAIL-NEW: [1-9]' report_md.md; then echo "DAST FAIL"; exit 1; fi
   ```

### Verify

Code scanning shows ZAP alerts; DefectDojo shows a test "ZAP baseline PR" under engagement `<branch>`.

### Cleanup

Close the PR; the kind cluster lives only in the runner. Delete the DefectDojo engagement if you created a throwaway product.

### What you learned

The review environment is created and destroyed inside the job; the token never touches the repo; three consumers (issue, code scanning, DefectDojo) read the same scan.

---

## Lab 6: Spring Boot `orders-service` with Form Login and AJAX Spider

**Objective.** Scan the Spring Boot service, which has a server-rendered admin UI (Thymeleaf) behind form login and a REST API with JWT, using a single AF plan with two contexts.

**Prerequisites.** Lab 4. `orders-service` running on `:8080` with the seeded `scan-user`. **Active scan: lab environment only.**

**Time.** 60 minutes.

### Steps

1. Create `zap-out/zap-orders.yaml`:

   ```yaml
   env:
     contexts:
       - name: orders-ui
         urls: ["${BASE_URL}/admin"]
         includePaths: ["${BASE_URL}/admin.*", "${BASE_URL}/login.*"]
         excludePaths: ["${BASE_URL}/logout.*", "${BASE_URL}/actuator.*"]
         authentication:
           method: form
           parameters:
             loginPageUrl: "${BASE_URL}/login"
             loginRequestUrl: "${BASE_URL}/login"
             loginRequestBody: "username={%username%}&password={%password%}&_csrf={%_csrf%}"
           verification:
             method: both
             loggedInRegex: '\Qhref="/logout"\E'
             loggedOutRegex: '\Qaction="/login"\E'
         sessionManagement:
           method: cookie
         users:
           - name: admin-scan
             credentials: { username: "${SCAN_USER}", password: "${SCAN_PASSWORD}" }
       - name: orders-api
         urls: ["${BASE_URL}/api"]
         includePaths: ["${BASE_URL}/api/.*"]
         authentication:
           method: json
           parameters:
             loginRequestUrl: "${BASE_URL}/api/auth/login"
             loginRequestBody: '{"username":"{%username%}","password":"{%password%}"}'
           verification:
             method: poll
             pollUrl: "${BASE_URL}/api/me"
             pollFrequency: 60
             pollUnits: requests
             loggedInRegex: '"username"'
             loggedOutRegex: '"status":401'
         sessionManagement:
           method: headers
           parameters: { Authorization: "Bearer {%json:token%}" }
         users:
           - name: api-scan
             credentials: { username: "${SCAN_USER}", password: "${SCAN_PASSWORD}" }
     parameters: { failOnError: true, failOnWarning: false, progressToStdout: true }
   jobs:
     - type: passiveScan-config
       parameters: { scanOnlyInScope: true, maxAlertsPerRule: 10 }
     - type: openapi
       parameters: { apiUrl: "${BASE_URL}/v3/api-docs", targetUrl: "${BASE_URL}", context: orders-api, user: api-scan }
     - type: spider
       parameters: { context: orders-ui, user: admin-scan, maxDuration: 3, postForm: true }
     - type: spiderAjax
       parameters: { context: orders-ui, user: admin-scan, maxDuration: 4, browserId: firefox-headless, numberOfBrowsers: 1, inScopeOnly: true }
     - type: passiveScan-wait
       parameters: { maxDuration: 5 }
     - type: activeScan
       parameters: { context: orders-ui, user: admin-scan, policy: Default Policy, maxScanDurationInMins: 15, maxRuleDurationInMins: 2 }
     - type: activeScan
       parameters: { context: orders-api, user: api-scan, policy: Default Policy, maxScanDurationInMins: 15, maxRuleDurationInMins: 2 }
     - type: report
       alwaysRun: true
       parameters: { template: traditional-html-plus, reportDir: /zap/wrk, reportFile: lab6, reportTitle: orders-service }
     - type: report
       alwaysRun: true
       parameters: { template: traditional-xml, reportDir: /zap/wrk, reportFile: lab6 }
     - type: exitStatus
       alwaysRun: true
       parameters: { errorLevel: High, warnLevel: Medium }
   ```

   The form login includes `_csrf`; ZAP's form-based auth automatically fetches `loginPageUrl` first and fills any anti-CSRF token it recognises from the page (`_csrf` is in its default list).

2. Run (the AJAX spider needs more memory):

   ```bash
   docker run --rm --add-host=host.docker.internal:host-gateway -v $(pwd)/zap-out:/zap/wrk/:rw \
     -e ZAP_JVM_OPTIONS="-Xmx3g" -e BASE_URL=$S -e SCAN_USER=scan-user -e SCAN_PASSWORD='<SCAN_PASSWORD>' \
     ghcr.io/zaproxy/zaproxy:stable zap.sh -cmd -autorun /zap/wrk/zap-orders.yaml
   ```

   Expected: `Job spiderAjax found 48 URLs` (varies), two active scans, `Automation plan succeeded!`.

3. Look specifically for: 10202 (CSRF) on the admin forms; 10054 (SameSite) on `JSESSIONID`; 90022 (stack trace disclosure, Spring Whitelabel error page); 10036/10037 headers. If `/actuator` was reachable, 10 or 40 alerts will reference `/actuator/env`; fix with `management.endpoints.web.exposure.include=health,info,prometheus` and `management.endpoint.env.show-values=never`.

### Verify

`grep -c 'JSESSIONID' zap-out/lab6.html` is non-zero, proving the UI context authenticated.

### Cleanup

`rm zap-out/lab6.*`.

### What you learned

Two contexts with different auth in one plan; the AJAX spider is what finds JavaScript-driven admin routes; Spring Boot defaults (Whitelabel page, actuator) are exactly what DAST catches.

---

## Lab 7: Nightly Full Scan in kind as a Kubernetes Job, Results to DefectDojo

**Objective.** Run ZAP inside the cluster against the in-cluster service names, persist the report, upload to DefectDojo, and schedule it with a CronJob.

**Prerequisites.** Labs 4 and 6; DefectDojo reachable (see [../../aspm-and-siem/defectdojo/](../../aspm-and-siem/defectdojo/README.md)). **Active scan: staging/lab cluster only.**

**Time.** 45 minutes.

### Steps

1. Store the plan and credentials in the cluster (lab shortcut: a plain Secret; in production the Vault Agent injects these, see [../../gate-7-runtime-and-secrets/vault/](../../gate-7-runtime-and-secrets/vault/README.md)).

   ```bash
   sed 's#http://host.docker.internal:8000#http://inventory-api.capstone.svc:8000#' zap-out/zap-inventory.yaml > zap-out/zap-incluster.yaml
   kubectl --namespace capstone create configmap zap-plan --from-file=zap.yaml=zap-out/zap-incluster.yaml
   kubectl --namespace capstone create secret generic zap-scan-creds \
     --from-literal=SCAN_USER=scan-user --from-literal=SCAN_PASSWORD='<SCAN_PASSWORD>' \
     --from-literal=DEFECTDOJO_API_KEY='<DEFECTDOJO_API_KEY>'
   ```

2. Create `zap-cronjob.yaml`:

   ```yaml
   apiVersion: batch/v1
   kind: CronJob
   metadata:
     name: zap-nightly
     namespace: capstone
   spec:
     schedule: "0 2 * * *"
     concurrencyPolicy: Forbid
     successfulJobsHistoryLimit: 3
     failedJobsHistoryLimit: 3
     jobTemplate:
       spec:
         backoffLimit: 0
         activeDeadlineSeconds: 5400
         template:
           spec:
             restartPolicy: Never
             securityContext: { runAsNonRoot: true, runAsUser: 1000, fsGroup: 1000, seccompProfile: { type: RuntimeDefault } }
             containers:
               - name: zap
                 image: ghcr.io/zaproxy/zaproxy:stable
                 command: ["/bin/bash", "-c"]
                 args:
                   - |
                     set -o pipefail
                     zap.sh -cmd -autorun /zap/plan/zap.yaml -config api.disablekey=true; rc=$?
                     ls -l /zap/wrk
                     curl -sf -X POST "$DEFECTDOJO_URL/api/v2/reimport-scan/" \
                       -H "Authorization: Token $DEFECTDOJO_API_KEY" \
                       -F scan_type="ZAP Scan" -F file=@/zap/wrk/lab4.xml \
                       -F product_type_name=Capstone -F product_name=inventory-api \
                       -F engagement_name=nightly-staging -F auto_create_context=true \
                       -F close_old_findings=true -F test_title="ZAP nightly full scan" -F environment=Staging
                     exit $rc
                 env:
                   - { name: BASE_URL, value: "http://inventory-api.capstone.svc:8000" }
                   - { name: DEFECTDOJO_URL, value: "http://defectdojo-nginx.defectdojo.svc:8080" }
                   - { name: ZAP_JVM_OPTIONS, value: "-Xmx3g" }
                 envFrom:
                   - secretRef: { name: zap-scan-creds }
                 resources:
                   requests: { cpu: "1", memory: "2Gi" }
                   limits: { cpu: "3", memory: "4Gi" }
                 securityContext: { allowPrivilegeEscalation: false, capabilities: { drop: ["ALL"] } }
                 volumeMounts:
                   - { name: plan, mountPath: /zap/plan, readOnly: true }
                   - { name: wrk, mountPath: /zap/wrk }
             volumes:
               - name: plan
                 configMap: { name: zap-plan }
               - name: wrk
                 emptyDir: {}
   ```

   Note: `exitStatus` sets a non-zero code on Medium; `set -o pipefail` plus `rc` preserves it while still uploading.

3. Apply and trigger once manually.

   ```bash
   kubectl apply -f zap-cronjob.yaml
   kubectl --namespace capstone create job zap-manual --from=cronjob/zap-nightly
   kubectl --namespace capstone logs -f job/zap-manual | tail -n 20
   ```

   Expected tail:

   ```text
   Automation plan succeeded!
   -rw-r--r-- 1 zap zap 184233 Oct 27 02:14 lab4.html
   -rw-r--r-- 1 zap zap  61240 Oct 27 02:14 lab4.xml
   {"test":42,"test_id":42,"engagement_id":7,"product_id":3,...}
   ```

4. In DefectDojo open Product `inventory-api`, Engagement `nightly-staging`, Test "ZAP nightly full scan". Findings carry CWE, endpoint, and the ZAP evidence. Re-run the job: the finding count should stay constant (reimport dedup).

### Verify

```bash
kubectl --namespace capstone get jobs | grep zap
curl -s -H "Authorization: Token <DEFECTDOJO_API_KEY>" "$DEFECTDOJO_URL/api/v2/findings/?test__engagement__name=nightly-staging&active=true&limit=1" | jq .count
```

### Cleanup

```bash
kubectl --namespace capstone delete cronjob zap-nightly job zap-manual configmap zap-plan secret zap-scan-creds
kubectl --namespace capstone delete deployment,svc -l app.kubernetes.io/part-of=capstone   # or make kind-down
kind delete cluster --name capstone
rm -rf zap-out
```

### What you learned

In-cluster scanning needs no port-forward and reaches services on cluster DNS; the CronJob plus DefectDojo reimport gives a stable nightly posture; Vault should own the credentials in anything beyond a lab.

---

## Where Next

[05-production-patterns.md](05-production-patterns.md) covers how to keep these scans fast, trustworthy, and safe as the number of services grows.
