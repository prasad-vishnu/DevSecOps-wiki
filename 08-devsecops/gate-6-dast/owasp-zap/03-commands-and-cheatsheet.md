# OWASP ZAP: Commands and Cheatsheet

| | |
|---|---|
| **Cluster** | 08 DevSecOps |
| **Role** | Anchor (Gate 6 DAST) |
| **Prerequisites** | [02-core-concepts.md](02-core-concepts.md) |
| **Target version** | ZAP 2.15+, `ghcr.io/zaproxy/zaproxy:stable` |
| **Capstone phase** | Phase 5 (Months 17-20) |

> Reference for the packaged scan scripts, `zap.sh` options, the Automation Framework job catalogue, `rules.tsv`, the REST API, the Python client, and the GitHub Actions. Every example targets the reader's own service on `localhost` / `host.docker.internal` or inside a kind cluster.

Shorthand used below: `ZAP="docker run --rm --add-host=host.docker.internal:host-gateway -v $(pwd)/zap-out:/zap/wrk/:rw ghcr.io/zaproxy/zaproxy:stable"` and `T=http://host.docker.internal:8000`.

---

## 1. Packaged Scan Scripts

### Common flags (all three scripts)

| Flag | Long meaning | Notes |
|---|---|---|
| `-t <target>` | Target URL (baseline/full) or spec URL/file (api-scan) | Required |
| `-r <file.html>` | HTML report | Written to `/zap/wrk/` |
| `-J <file.json>` | JSON report | Machine-readable; DefectDojo accepts XML, not this |
| `-x <file.xml>` | XML report | Use for DefectDojo `ZAP Scan` |
| `-w <file.md>` | Markdown report | Good for PR comments |
| `-c <rules.tsv>` | Rules file from `/zap/wrk/` | Tuning |
| `-u <url>` | Rules file fetched over HTTP | Central rules repo |
| `-g <rules.tsv>` | Generate a rules file with every rule set to WARN | Starting point |
| `-l PASS\|IGNORE\|INFO\|WARN\|FAIL` | Minimum level to show in output | `-l WARN` hides PASS lines |
| `-I` | Do not return failure on WARN | Exit 0 on WARN-only |
| `-i` | Default rules not in the rules file to INFO | Reduces noise |
| `-a` | Include alpha passive rules | Weekly image |
| `-d` | Debug output | Troubleshooting |
| `-s` | Short output (no PASS lines) | CI logs |
| `-n <context.context>` | Context file exported from desktop | Auth, scope |
| `-U <user>` | User (from the context) to scan as | Needs `-n` |
| `-P <port>` | Listen port for ZAP | Default random |
| `-D <seconds>` | Delay before starting | Wait for app readiness |
| `-T <minutes>` | Max total time | Hard cap |
| `-z "<options>"` | Raw `zap.sh` options, e.g. `-config` entries | Replacer, api settings |
| `--hook <file.py>` | Python hooks | Custom logic at each stage |
| `--auto` / `--autooff` | Use / do not use the Automation Framework internally | Default auto |

### `zap-baseline.py` specific

| Flag | Meaning |
|---|---|
| `-m <minutes>` | Spider duration (default 1) |
| `-j` | Also run the AJAX spider |
| `-O <host>` | Override the host in an imported spec |

```bash
$ZAP zap-baseline.py -t $T -c rules.tsv -r baseline.html -x baseline.xml -w baseline.md -m 2 -j -I -s
```

### `zap-full-scan.py` specific

| Flag | Meaning |
|---|---|
| `-m <minutes>` | Spider duration (default 1) |
| `-j` | AJAX spider |
| `-T <minutes>` | Max total time (important; active scans can run for hours) |

```bash
$ZAP zap-full-scan.py -t $T -c rules.tsv -r full.html -x full.xml -J full.json -m 5 -T 60 -z "-config scanner.threadPerHost=4"
```

### `zap-api-scan.py` specific

| Flag | Meaning |
|---|---|
| `-f openapi\|soap\|graphql` | Spec format (required) |
| `-t <spec>` | URL to the spec, or file name inside `/zap/wrk/` |
| `-O <host>` | Host override (`http://host.docker.internal:8000`) when the spec's `servers` points elsewhere |
| `-S` | Safe mode: no active scan, only passive and import |

```bash
# FastAPI
$ZAP zap-api-scan.py -t $T/openapi.json -f openapi -O $T -c rules-api.tsv -r api.html -x api.xml -J api.json
# Spring Boot (springdoc)
$ZAP zap-api-scan.py -t http://host.docker.internal:8080/v3/api-docs -f openapi -O http://host.docker.internal:8080 -r orders-api.html
# GraphQL by introspection
$ZAP zap-api-scan.py -t $T/graphql -f graphql -r graphql.html
```

### Hooks (`--hook`)

```python
# zap-out/hooks.py
def zap_started(zap, target):
    # Register a scan user agent so the SOC can filter it
    zap.core.set_option_default_user_agent("capstone-dast-scanner/1.0")

def zap_pre_shutdown(zap):
    print("Alerts:", zap.core.number_of_alerts())
```

Available hooks: `cli_opts`, `zap_started`, `importing_openapi`, `zap_import_context`, `zap_spider`, `zap_ajax_spider`, `zap_active_scan`, `zap_get_alerts`, `zap_pre_shutdown`, `pre_exit`, `start_zap`, `start_docker_zap`, `load_config`, `print_rules`, `print_rules_wrap`.

---

## 2. `zap.sh` Options

| Option | Meaning |
|---|---|
| `-daemon` | Headless, stays running |
| `-cmd` | Headless, run then exit (with `-autorun`, `-quickurl`) |
| `-autorun <plan.yaml>` | Run an Automation Framework plan |
| `-host 0.0.0.0 -port 8080` | Bind the proxy/API |
| `-config key=value` | Override any option (repeatable) |
| `-configfile <file>` | Many `-config` lines from a file |
| `-newsession <path>` / `-session <path>` | Persist / reopen a session |
| `-addoninstall <id>` / `-addonuninstall <id>` / `-addonupdate` | Add-on management |
| `-addonlist` | List add-ons |
| `-quickurl <url> -quickout <report> -quickprogress` | Legacy one-shot scan |
| `-script <file>` | Run a standalone script |
| `-silent` | Do not check for updates |
| `-version` | Print version |

Useful `-config` keys:

```text
api.key=<ZAP_API_KEY>
api.addrs.addr.name=.*            api.addrs.addr.regex=true
api.disablekey=true               (only when unreachable from outside)
connection.timeoutInSecs=60
scanner.threadPerHost=5           scanner.hostPerScan=2
scanner.maxRuleDurationInMins=5   scanner.maxScanDurationInMins=60
scanner.delayInMs=0
spider.maxDuration=5              spider.maxDepth=5             spider.maxChildren=50
pscans.maxAlertsPerRule=10
view.mode=daemon
database.recoverylog=false
network.localServers.mainProxy.port=8080
replacer.full_list(0).description=auth
replacer.full_list(0).enabled=true
replacer.full_list(0).matchtype=REQ_HEADER
replacer.full_list(0).matchstr=Authorization
replacer.full_list(0).regex=false
replacer.full_list(0).replacement=Bearer <TOKEN>
replacer.full_list(1).matchtype=REQ_HEADER_STR     (for raw header string regex)
```

JVM memory: `-e ZAP_JVM_OPTIONS="-Xmx4g"` on `docker run`.

---

## 3. `rules.tsv`

```text
# zap-out/rules.tsv  --  id<TAB>action<TAB>comment or URL regex (OUTOFSCOPE)
10021	IGNORE	(X-Content-Type-Options - set by ingress in staging, tracked as DOJO-1234)
10036	WARN	(Server header version leak - fix in uvicorn config)
10038	FAIL	(CSP header missing - must be present)
10020	FAIL	(Anti-clickjacking header)
10035	FAIL	(HSTS - staging terminates TLS at ingress)
10063	INFO	(Permissions-Policy - informational for API-only service)
10096	IGNORE	(Timestamp disclosure - build time in /version is intended)
10049	OUTOFSCOPE	.*/openapi\.json
40012	FAIL	(Reflected XSS)
40014	FAIL	(Persistent XSS)
40018	FAIL	(SQL injection)
90020	FAIL	(OS command injection)
6	FAIL	(Path traversal)
10202	WARN	(Absence of anti-CSRF tokens - API uses bearer tokens)
```

Generate a complete starting file with every rule:

```bash
$ZAP zap-baseline.py -t $T -g rules-all.tsv -I >/dev/null; wc -l zap-out/rules-all.tsv
```

### Frequently tuned rule ids

| Id | Name | Typical disposition for an API |
|---|---|---|
| 10010 / 10011 | Cookie No HttpOnly / Without Secure | FAIL for session cookies |
| 10015 | Re-examine Cache-control Directives | INFO |
| 10017 | Cross-Domain JavaScript Source File Inclusion | WARN |
| 10020 | Missing Anti-clickjacking Header | FAIL for HTML, IGNORE for pure JSON API |
| 10021 | X-Content-Type-Options Header Missing | FAIL |
| 10024 | Information Disclosure: Sensitive Information in URL | FAIL |
| 10027 | Information Disclosure: Suspicious Comments | INFO |
| 10035 | Strict-Transport-Security Header Not Set | FAIL (where TLS terminates at the app) |
| 10036 | Server Leaks Version Information | WARN |
| 10037 | Server Leaks Information via X-Powered-By | WARN |
| 10038 | CSP Header Not Set | FAIL for HTML; INFO for API |
| 10049 | Storable and Cacheable Content | INFO |
| 10054 | Cookie without SameSite Attribute | WARN |
| 10055 | CSP: various weaknesses | WARN |
| 10063 | Permissions Policy Header Not Set | INFO |
| 10096 | Timestamp Disclosure | IGNORE (noisy) |
| 10098 | Cross-Domain Misconfiguration (CORS `*`) | FAIL |
| 10104 | User Agent Fuzzer | INFO |
| 10109 | Modern Web Application | INFO |
| 10110 | Dangerous JS Functions | WARN |
| 10202 | Absence of Anti-CSRF Tokens | WARN |
| 90022 | Application Error Disclosure | FAIL |
| 40012 / 40014 / 40016 / 40017 | XSS reflected / persistent / prime / spider | FAIL |
| 40018-40027 | SQL injection family | FAIL |
| 90019 / 90020 | Server-side code injection / OS command injection | FAIL |
| 6 / 7 | Path traversal / Remote file inclusion | FAIL |
| 40003 | CRLF injection | FAIL |
| 20019 | External redirect | FAIL |
| 90034 | Cloud metadata potentially exposed (SSRF to 169.254.169.254) | FAIL |

Full list: https://www.zaproxy.org/docs/alerts/

---

## 4. Automation Framework Job Reference

### `env`

```yaml
env:
  contexts:
    - name: inventory-api
      urls: ["${BASE_URL}"]
      includePaths: ["${BASE_URL}/.*"]
      excludePaths: ["${BASE_URL}/metrics", "${BASE_URL}/auth/logout.*"]
      technology:
        include: [Language.Python, Db.PostgreSQL, OS.Linux]
      authentication:
        method: json
        parameters:
          loginPageUrl: "${BASE_URL}/docs"
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
  vars:
    BASE_URL: http://host.docker.internal:8000
  parameters:
    failOnError: true
    failOnWarning: false
    progressToStdout: true
```

### Jobs

| Job | Key parameters |
|---|---|
| `addOns` | `updateAddOns: false`, `install: [graphql, openapi]`, `uninstall: []` |
| `passiveScan-config` | `maxAlertsPerRule`, `scanOnlyInScope: true`, `maxBodySizeInBytesToScan`, `enableTags`, `disableAllRules`, `rules: [{id, threshold}]` |
| `replacer` | `rules: [{description, url, matchType: req_header, matchString: Authorization, replacementString: "Bearer ${TOKEN}", tokenProcessing: false}]` |
| `requestor` | `requests: [{url, method, data, headers, responseCode}]`, `user` |
| `openapi` | `apiFile` or `apiUrl`, `targetUrl`, `context`, `user` |
| `graphql` | `endpoint`, `schemaUrl`/`schemaFile`, `maxQueryDepth`, `lenientMaxQueryDepthEnabled`, `maxArgsDepth`, `optionalArgsEnabled`, `argsType: both`, `querySplitType: leaf`, `requestMethod: post_json` |
| `soap` | `wsdlFile`/`wsdlUrl` |
| `import` | `type: har|url|modsec2`, `fileName` |
| `spider` | `context`, `user`, `url`, `maxDuration`, `maxDepth`, `maxChildren`, `acceptCookies`, `handleODataParametersVisited`, `handleParameters: ignore_completely|ignore_value|use_all`, `parseComments`, `parseRobotsTxt`, `parseSitemapXml`, `postForm`, `processForm`, `sendRefererHeader`, `threadCount`, `userAgent` |
| `spiderAjax` | `context`, `user`, `url`, `maxDuration`, `maxCrawlDepth`, `maxCrawlStates`, `numberOfBrowsers`, `browserId: firefox-headless`, `clickDefaultElems`, `clickElemsOnce`, `eventWait`, `reloadWait`, `inScopeOnly`, `runOnlyIfModern`, `elements`, `excludedElements` |
| `passiveScan-wait` | `maxDuration` (minutes) |
| `activeScan-config` | `maxRuleDurationInMins`, `maxScanDurationInMins`, `maxAlertsPerRule`, `defaultPolicy`, `handleAntiCSRFTokens`, `injectPluginIdInHeader`, `threadPerHost`, `inputVectors: {urlQueryStringAndDataDrivenNodes: {enabled, addParam, odata}, postData: {enabled, multiPartFormData, xml, json: {enabled, scanNullValues}, googleWebToken}, urlPath, httpHeaders: {enabled, allRequests}, cookieData: {enabled, encodeCookieValues}, scripts}` |
| `activeScan-policy` | `name`, `policyDefinition: {defaultStrength, defaultThreshold, rules: [{id, name, strength, threshold}]}` |
| `activeScan` | `context`, `user`, `url`, `policy`, `maxRuleDurationInMins`, `maxScanDurationInMins`, `addQueryParam`, `defaultPolicy`, `delayInMs`, `handleAntiCSRFTokens`, `injectPluginIdInHeader`, `scanHeadersAllRequests`, `threadPerHost`, inline `policyDefinition` |
| `alertFilter` | `deleteGlobalAlerts`, `alertFilters: [{ruleId, newRisk: False Positive|Info|Low|Medium|High, context, url, urlRegex, parameter, parameterRegex, attack, attackRegex, evidence, evidenceRegex, methods}]` |
| `report` | `template`, `theme`, `reportDir`, `reportFile`, `reportTitle`, `reportDescription`, `displayReport`, `risks: [high, medium, low, info]`, `confidences`, `sections`, `sites` |
| `exportdata` | `type: har|url`, `source: history|all`, `context`, `fileName` |
| `exitStatus` | `errorLevel`, `warnLevel`, `okExitValue: 0`, `errorExitValue: 1`, `warnExitValue: 2` |
| `script` | `action: add|remove|run|enable|disable`, `type`, `engine`, `name`, `source`, `inline` |
| `delay` | `time: "0:00:10"`, `fileName` (resume when file appears) |
| `postman` | `collectionFile`/`collectionUrl`, `variables` |

### Report templates

| Template | Output | Use |
|---|---|---|
| `traditional-html`, `traditional-html-plus` | HTML | Humans; plus adds request/response |
| `modern` | HTML (newer look) | Humans |
| `risk-confidence-html` | HTML grid | Triage |
| `traditional-json`, `traditional-json-plus` | JSON | Automation |
| `traditional-xml`, `traditional-xml-plus` | XML | DefectDojo `ZAP Scan` |
| `traditional-md` | Markdown | PR comments |
| `sarif-json` | SARIF 2.1 | GitHub code scanning |
| `high-level-report` | HTML summary | Management |
| `traditional-pdf` | PDF | Audit |

### Tests on jobs

```yaml
  - type: spider
    parameters: { context: inventory-api, maxDuration: 3 }
    tests:
      - name: "at least 20 urls"
        type: stats
        statistic: automation.spider.urls.added
        operator: ">="
        value: 20
        onFail: error
  - type: activeScan
    tests:
      - name: "no high risk"
        type: alert
        action: passIfAbsent
        scanRuleId: 40018
        risk: High
        onFail: error
```

---

## 5. REST API Quick Reference

Prefix: `http://localhost:8090` (daemon mapped port), `apikey=<ZAP_API_KEY>` on every call.

```bash
K=<ZAP_API_KEY>; Z=http://localhost:8090; T=http://host.docker.internal:8000
curl -s "$Z/JSON/core/view/version/?apikey=$K"
curl -s "$Z/JSON/core/action/accessUrl/?apikey=$K&url=$T/&followRedirects=true"
curl -s "$Z/JSON/openapi/action/importUrl/?apikey=$K&url=$T/openapi.json&hostOverride=$T"
curl -s "$Z/JSON/spider/action/scan/?apikey=$K&url=$T&recurse=true&maxChildren=50"          # -> {"scan":"0"}
curl -s "$Z/JSON/spider/view/status/?apikey=$K&scanId=0"                                      # -> {"status":"100"}
curl -s "$Z/JSON/pscan/view/recordsToScan/?apikey=$K"                                          # -> {"recordsToScan":"0"}
curl -s "$Z/JSON/ascan/action/scan/?apikey=$K&url=$T&recurse=true&inScopeOnly=true&scanPolicyName=API-Policy"
curl -s "$Z/JSON/ascan/view/status/?apikey=$K&scanId=0"
curl -s "$Z/JSON/ascan/view/scanProgress/?apikey=$K&scanId=0" | jq '.scanProgress[1].HostProcess[] | {name: .Plugin[0], status: .Plugin[3], time: .Plugin[4]}'
curl -s "$Z/JSON/alert/view/alertsSummary/?apikey=$K&baseurl=$T"                              # -> {"alertsSummary":{"High":0,"Medium":2,...}}
curl -s "$Z/JSON/alert/view/alerts/?apikey=$K&baseurl=$T&start=0&count=100" | jq '.alerts[] | {pluginId, risk, name, url}'
curl -s "$Z/OTHER/core/other/xmlreport/?apikey=$K" -o zap-out/report.xml
curl -s "$Z/JSON/reports/action/generate/?apikey=$K&title=inventory-api&template=sarif-json&reportDir=/zap/wrk&reportFileName=zap.sarif"
curl -s "$Z/JSON/core/action/newSession/?apikey=$K&name=&overwrite=true"
curl -s "$Z/JSON/core/action/shutdown/?apikey=$K"
```

Context and auth via API:

```bash
curl -s "$Z/JSON/context/action/newContext/?apikey=$K&contextName=inv"
curl -s "$Z/JSON/context/action/includeInContext/?apikey=$K&contextName=inv&regex=$T/.*"
curl -s "$Z/JSON/authentication/action/setAuthenticationMethod/?apikey=$K&contextId=1&authMethodName=jsonBasedAuthentication&authMethodConfigParams=loginUrl%3D$T%2Fauth%2Ftoken%26loginRequestData%3D%7B%22username%22%3A%22%7B%25username%25%7D%22%2C%22password%22%3A%22%7B%25password%25%7D%22%7D"
curl -s "$Z/JSON/authentication/action/setLoggedInIndicator/?apikey=$K&contextId=1&loggedInIndicatorRegex=%22username%22"
curl -s "$Z/JSON/users/action/newUser/?apikey=$K&contextId=1&name=scan-user"
curl -s "$Z/JSON/users/action/setAuthenticationCredentials/?apikey=$K&contextId=1&userId=0&authCredentialsConfigParams=username%3D<SCAN_USER>%26password%3D<SCAN_PASSWORD>"
curl -s "$Z/JSON/users/action/setUserEnabled/?apikey=$K&contextId=1&userId=0&enabled=true"
curl -s "$Z/JSON/spider/action/scanAsUser/?apikey=$K&contextId=1&userId=0&url=$T&recurse=true"
curl -s "$Z/JSON/ascan/action/scanAsUser/?apikey=$K&url=$T&contextId=1&userId=0&recurse=true"
```

Scan policy via API:

```bash
curl -s "$Z/JSON/ascan/action/addScanPolicy/?apikey=$K&scanPolicyName=API-Policy&alertThreshold=MEDIUM&attackStrength=LOW"
curl -s "$Z/JSON/ascan/action/disableAllScanners/?apikey=$K&scanPolicyName=API-Policy"
curl -s "$Z/JSON/ascan/action/enableScanners/?apikey=$K&scanPolicyName=API-Policy&ids=40018,40012,90020,6,7,40003,20019,90034,90019"
curl -s "$Z/JSON/ascan/action/setScannerAttackStrength/?apikey=$K&id=40018&attackStrength=HIGH&scanPolicyName=API-Policy"
curl -s "$Z/JSON/ascan/view/scanners/?apikey=$K&scanPolicyName=API-Policy" | jq '.scanners[] | select(.enabled=="true") | {id, name, attackStrength, alertThreshold}'
```

---

## 6. Python Client (`zaproxy`)

```python
import os, time
from zapv2 import ZAPv2

API_KEY = os.environ["ZAP_API_KEY"]
TARGET = os.environ.get("TARGET", "http://host.docker.internal:8000")
zap = ZAPv2(apikey=API_KEY, proxies={"http": "http://localhost:8090", "https": "http://localhost:8090"})

zap.openapi.import_url(TARGET + "/openapi.json", hostoverride=TARGET)
sid = zap.spider.scan(TARGET, recurse=True, contextname=None)
while int(zap.spider.status(sid)) < 100:
    time.sleep(2)
while int(zap.pscan.records_to_scan) > 0:
    time.sleep(2)

asid = zap.ascan.scan(TARGET, recurse=True, inscopeonly=False, scanpolicyname="API-Policy")
while int(zap.ascan.status(asid)) < 100:
    print("active scan", zap.ascan.status(asid), "%")
    time.sleep(10)

summary = zap.alert.alerts_summary(baseurl=TARGET)
print(summary)                          # {'High': 0, 'Medium': 3, 'Low': 5, 'Informational': 4}
with open("zap-out/report.xml", "w") as f:
    f.write(zap.core.xmlreport())
zap.reports.generate(title="inventory-api", template="sarif-json", reportdir="/zap/wrk", reportfilename="zap.sarif")
high = [a for a in zap.alert.alerts(baseurl=TARGET) if a["risk"] == "High"]
raise SystemExit(1 if high else 0)
```

---

## 7. GitHub Actions

| Action | Wraps | Key inputs |
|---|---|---|
| `zaproxy/action-baseline@v0.14.0` | `zap-baseline.py` | `target`, `rules_file_name`, `cmd_options`, `docker_name`, `token`, `issue_title`, `allow_issue_writing`, `fail_action`, `artifact_name` |
| `zaproxy/action-full-scan@v0.12.0` | `zap-full-scan.py` | same |
| `zaproxy/action-api-scan@v0.9.0` | `zap-api-scan.py` | `target` (spec URL), `format: openapi\|soap\|graphql`, plus the above |
| `zaproxy/action-af@v0.2.0` | `zap.sh -cmd -autorun` | `plan` (path to YAML), `docker_name`, `docker_env_vars` |

```yaml
- name: ZAP baseline against PR review env
  uses: zaproxy/action-baseline@v0.14.0
  with:
    target: ${{ steps.review.outputs.url }}
    rules_file_name: .zap/rules.tsv
    cmd_options: "-a -j -m 2 -x zap-report.xml -J zap-report.json -I"
    allow_issue_writing: true
    issue_title: "ZAP baseline: ${{ github.head_ref }}"
    token: ${{ secrets.GITHUB_TOKEN }}
    fail_action: false
    artifact_name: zap-baseline
```

The actions write `report_html.html`, `report_json.json`, `report_md.md` into the workspace and (with `allow_issue_writing`) open or update a single GitHub issue summarising WARN/FAIL. SARIF needs the AF `report` job or the API; see labs.

---

## 8. Desktop UI Shortcuts

| Task | Where |
|---|---|
| Set scope | Right-click a site, Include in Context |
| Export context for `-n` | File, Export Context |
| Build AF plan | Automation tab, New Plan, Save |
| Scan policy manager | Analyse, Scan Policy Manager |
| Alert filters | Context, Alert Filters |
| Manual explore with browser | Quick Start, Manual Explore, Launch Browser |
| Replay/modify a request | Right-click history, Open/Resend with Request Editor |
| Fuzz a parameter | Highlight value in request, right-click, Fuzz |
| Break (intercept) | Toolbar green/red circle |
| HUD (in-browser overlay) | Quick Start, Manual Explore, enable HUD |
| Encode/decode/hash | Tools, Encode/Decode/Hash |
| Scripts console | Tools, Options, Scripts; or the Script Console tab |

---

## 9. One-Liners

```bash
# Quick passive-only check of a local service, exit 0 regardless
$ZAP zap-baseline.py -t $T -I -s

# Fail only on High, show only WARN/FAIL lines
$ZAP zap-baseline.py -t $T -c rules.tsv -l WARN -s

# Bearer token injection with packaged scan
$ZAP zap-api-scan.py -t $T/openapi.json -f openapi -O $T -r api.html \
  -z "-config replacer.full_list(0).description=auth -config replacer.full_list(0).enabled=true -config replacer.full_list(0).matchtype=REQ_HEADER -config replacer.full_list(0).matchstr=Authorization -config replacer.full_list(0).regex=false -config replacer.full_list(0).replacement='Bearer <TOKEN>'"

# AF plan with secrets from environment
docker run --rm --add-host=host.docker.internal:host-gateway -v $(pwd)/zap-out:/zap/wrk/:rw \
  -e SCAN_USER=<SCAN_USER> -e SCAN_PASSWORD=<SCAN_PASSWORD> -e BASE_URL=$T \
  ghcr.io/zaproxy/zaproxy:stable zap.sh -cmd -autorun /zap/wrk/zap.yaml

# Count alerts by risk from a JSON report
jq '[.site[].alerts[] | {riskdesc, name, count: (.instances|length)}] | group_by(.riskdesc) | map({risk: .[0].riskdesc, rules: length})' zap-out/api.json

# Convert JSON alerts to a Markdown table for a PR comment
jq -r '.site[].alerts[] | "| \(.pluginid) | \(.riskdesc) | \(.name) | \(.instances|length) |"' zap-out/api.json

# Upload XML to DefectDojo
curl -s -X POST "https://defectdojo.example.com/api/v2/reimport-scan/" \
  -H "Authorization: Token <DEFECTDOJO_API_KEY>" \
  -F scan_type="ZAP Scan" -F file=@zap-out/api.xml \
  -F product_type_name="Capstone" -F product_name="inventory-api" \
  -F engagement_name="main" -F auto_create_context=true \
  -F close_old_findings=true -F minimum_severity=Low -F test_title="ZAP API scan"
```

Next: [04-hands-on-labs.md](04-hands-on-labs.md).
