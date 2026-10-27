# OWASP ZAP: Interview Questions

| | |
|---|---|
| **Cluster** | 08 DevSecOps |
| **Role** | Anchor (Gate 6 DAST) |
| **Prerequisites** | The rest of this folder |
| **Target version** | ZAP 2.15+ |
| **Capstone phase** | Phase 5 (Months 17-20) |

> Thirty questions from junior to principal level with model answers. Use them to check understanding, not to memorise; interviewers at the senior end care about trade-offs and failure modes.

---

## Junior

### Q1. What is DAST and how does it differ from SAST? (Junior)

**Answer.** DAST tests a running application from the outside over HTTP, with no knowledge of source code; it finds misconfigurations, missing headers, exposed endpoints, and injection flaws that manifest in responses. SAST analyses source or bytecode without running it and finds code-level issues such as taint flows. DAST is language-agnostic and sees the composed system (proxies, framework defaults, deployment config); SAST sees only your code. They are complementary gates (2 and 6 in the capstone).

### Q2. What is the difference between ZAP's passive and active scanner? (Junior)

**Answer.** The passive scanner inspects requests and responses that already flowed through ZAP and never sends additional traffic; it detects header, cookie, and information-disclosure issues and is safe wherever you are authorised to observe. The active scanner replays requests with attack payloads (SQLi, XSS, traversal) and mutates state; it is only for disposable or staging environments you own.

### Q3. Name the three packaged scan scripts and when to use each. (Junior)

**Answer.** `zap-baseline.py` (spider plus passive scan, minutes, per PR), `zap-full-scan.py` (spider plus active scan, long, nightly), `zap-api-scan.py` (imports an OpenAPI/GraphQL/SOAP definition and runs an API-tuned active scan, post-merge on staging).

### Q4. What do exit codes 0, 1, 2, and 3 mean from a packaged scan? (Junior)

**Answer.** 0: no WARN or FAIL (or only WARN with `-I`). 1: at least one FAIL. 2: at least one WARN. 3: ZAP itself failed to run (unreachable target, bad arguments, startup failure).

### Q5. What is `rules.tsv`? (Junior)

**Answer.** A tab-separated file passed with `-c` mapping scan rule ids to actions: IGNORE, INFO, WARN, FAIL, or OUTOFSCOPE (with a URL regex). It is how the packaged scans are tuned; unlisted rules default to WARN.

### Q6. Why did a baseline scan of a JSON API find only two URLs? (Junior)

**Answer.** The traditional spider follows HTML links; a JSON API has none. Coverage for APIs comes from importing the OpenAPI/GraphQL spec, which creates one request per operation with valid example bodies.

### Q7. How do you make ZAP send an `Authorization: Bearer` header on every request in a packaged scan? (Junior)

**Answer.** Use the Replacer via `-z "-config replacer.full_list(0).matchtype=REQ_HEADER -config replacer.full_list(0).matchstr=Authorization -config replacer.full_list(0).replacement='Bearer <TOKEN>' ..."` (plus `description`, `enabled=true`, `regex=false`). In the Automation Framework use the `replacer` job.

### Q8. Which report format does DefectDojo's "ZAP Scan" parser accept? (Junior)

**Answer.** The XML report (`-x report.xml` or AF template `traditional-xml`). JSON and Markdown are for other consumers.

---

## Mid-level

### Q9. Explain contexts and why almost every advanced ZAP feature depends on them. (Mid-level)

**Answer.** A context is a named scope (include/exclude regexes) carrying authentication method, session management, users, technology set, structural modifiers, and alert filters. Spidering as a user, active scanning as a user, scanning only in scope, and technology-based payload pruning all take a context. Without a context ZAP has no notion of "logged in" or "in scope".

### Q10. Describe how ZAP keeps an authenticated session alive during a long scan. (Mid-level)

**Answer.** The context's verification settings define logged-in/logged-out regexes and a method (`response`, `request`, `both`, `poll` with a poll URL and frequency). When verification indicates the session is gone, ZAP re-runs the authentication method (form, JSON, script, browser) for the user and the session management method (cookie or header) extracts and reapplies the new session token. Without verification configured there is no re-authentication.

### Q11. What are attack strength and alert threshold, and how do they interact? (Mid-level)

**Answer.** Attack strength (Low/Medium/High/Insane) controls how many payload variants an active rule sends per parameter: it drives time and coverage. Alert threshold (Off/Low/Medium/High) controls how much evidence a rule needs before raising an alert: it drives false positives versus false negatives. A scan policy sets both per rule. For a PR gate you want Low strength and Medium threshold on a short allow-list; nightly you raise strength.

### Q12. Why does the Automation Framework exist when the REST API already does everything? (Mid-level)

**Answer.** The API requires imperative orchestration (start spider, poll, wait for passive queue, start active scan, poll, generate report) that every team re-implemented in brittle scripts. The AF is a declarative, versionable YAML plan with environment, contexts, ordered jobs, variable substitution from the environment, built-in tests (coverage assertions), alert filters, multiple report templates, and exit-status mapping. It is reviewable in a PR and exportable from the desktop UI.

### Q13. How do you assert that a scan actually covered the application? (Mid-level)

**Answer.** Add a `tests` block to the `spider`/`spiderAjax`/`openapi` job with a `stats` test on `automation.spider.urls.added` (or the import's URL count) with `operator: ">="` and `onFail: error`. Also compare the status code distribution: mostly 2xx means authenticated coverage; mostly 401 means the scan tested only the auth middleware.

### Q14. A rule produces a real but accepted finding on one endpoint. How do you suppress it correctly? (Mid-level)

**Answer.** Use the narrowest filter: an `alertFilter` with `ruleId`, `urlRegex`, and optionally `parameter`, setting `newRisk: Info` (still visible) rather than `False Positive`, with a comment referencing the DefectDojo risk acceptance id and an expiry. The change goes through the repository with security CODEOWNERS review. Never a global IGNORE for the rule.

### Q15. Why run ZAP as a Kubernetes Job inside the cluster rather than from the CI runner? (Mid-level)

**Answer.** In-cluster the scanner reaches services on cluster DNS without port-forwards (which die under load), inherits NetworkPolicies limiting it to the target namespace, can be scheduled on spot nodes as a CronJob, and can read credentials through the Vault Agent sidecar instead of CI secrets. The cost is shipping reports out (DefectDojo upload from the Job).

### Q16. How do you integrate ZAP output with GitHub code scanning? (Mid-level)

**Answer.** Generate SARIF with the AF `report` job `template: sarif-json` (or the reports API), then `github/codeql-action/upload-sarif` with a per-service `category`. DAST SARIF has no file locations; the URL is in the result message. Keep results under GitHub's size limits via `maxAlertsPerRule` and excluding Informational.

### Q17. What is OAST and when do you need it? (Mid-level)

**Answer.** Out-of-band application security testing: for blind vulnerabilities (SSRF, blind XXE, blind command injection) where the response reveals nothing, ZAP's `oast` add-on embeds unique callback URLs from an external service (Interactsh/BOAST) in payloads; a DNS/HTTP callback proves the flaw. Self-host the callback server inside the cluster so staging never calls out to a third party.

### Q18. What is the difference between the traditional spider and the AJAX spider, and when do you use each? (Mid-level)

**Answer.** The traditional spider parses HTML/JS text for links and submits forms; fast, deterministic, blind to runtime-generated URLs. The AJAX spider drives a headless browser, clicking elements to trigger JavaScript routes and XHR; slow, memory-heavy, non-deterministic, but necessary for SPAs. Use the traditional spider plus spec import for APIs; add the AJAX spider for JavaScript front-ends with a tight `maxDuration`.

---

## Senior

### Q19. Design the DAST stage for a monorepo with 12 services and a 10-minute PR budget. (Senior)

**Answer.** Matrix job per changed service (path filters) deploying to `pr-<N>` namespaces on a shared cluster via Argo CD ApplicationSet; passive-only AF plan with spec import, coverage test, bearer token minted per job, `warnLevel: High` so only severe passives block; SARIF per service category; XML reimport to DefectDojo with `test_title` per service. Active scanning moves to a post-merge staging job (`capstone-api` policy, 20-minute cap) and a nightly CronJob with the full policy. A reusable workflow and a base plan with per-service overlays keep configuration DRY; central rules via `-u` plus local `rules.tsv`.

### Q20. The nightly full scan takes 4 hours. Walk through how you would get it under 45 minutes. (Senior)

**Answer.** Measure per-job and per-rule time from the log and `scanProgress`. Replace spidering with spec import; cap the AJAX spider at 3 minutes and `clickElemsOnce`; switch the policy to `defaultThreshold: Off` with an explicit rule list; set `technology.include` to prune payloads; raise `threadPerHost` to 6-8 while watching app p99; disable header/cookie/path input vectors; set `maxRuleDurationInMins: 3` and `maxScanDurationInMins: 40`; exclude static assets and metrics; give the JVM 4 GB. If still too slow, split into per-context plans run in parallel Jobs.

### Q21. How would you test authorisation (BOLA/IDOR) with ZAP rather than Burp? (Senior)

**Answer.** Define two users in the context (user A, user B) and use the Access Control Testing add-on: spider as both, define access rules per node, run the test which replays each user's requests as the other user and compares responses. Alternatively, run the spider as user A, export URLs, then `requestor` them as user B and diff status codes. ZAP's support is workable but Burp's Autorize is faster for a human; the choice depends on whether it must run in CI.

### Q22. What are the security risks of running ZAP itself, and how do you mitigate them? (Senior)

**Answer.** The API key grants full control of a proxy that holds request/response bodies including tokens: random key per run, never `api.disablekey` on a reachable port, no port exposure in CI. Reports contain sensitive evidence: short-retention artifacts, plain (not `-plus`) templates for shared outputs, token masking. The container runs as uid 1000 non-root with dropped capabilities; the image is pinned by digest and scanned with Trivy. OAST callbacks are self-hosted. Scanner credentials have least privilege and come from Vault.

### Q23. How do dedup and `close_old_findings` behave when you reimport ZAP results into DefectDojo nightly? (Senior)

**Answer.** `reimport-scan` to the same test (matched by `test_title` within the engagement) updates existing findings matched by the parser's hash code (title, CWE, endpoint for ZAP), creates new ones, and with `close_old_findings=true` marks findings absent from the new report as mitigated. The result is a stable finding set with history rather than N copies per night; false positives and risk acceptances persist across reimports.

### Q24. A team claims ZAP has a 40% false-positive rate and wants to turn it off. How do you respond? (Senior)

**Answer.** Quantify: list the rules behind the noise. Usually a handful of passive rules on static assets or Informational-confidence actives at Low threshold. Fix by scope (`excludePaths`), threshold (Medium/High for noisy rules), narrow `alertFilter`s, `maxAlertsPerRule`, and DefectDojo dedup so the same instance is one finding. Keep the gate but temporarily `warnLevel: High`, then tighten. Make suppressions reviewable in Git. Then show the true positives it did catch (headers, actuator exposure) to re-establish trust.

---

## Principal

### Q25. Argue for and against making the DAST gate blocking for production promotion. (Principal)

**Answer.** For: it is the only gate that validates the composed runtime configuration; a High active finding (SQLi, SSRF to metadata) is release-blocking by any reasonable risk standard; the gate produces audit evidence. Against: DAST is slow and partially non-deterministic, so a blocking gate on every deploy creates pressure to suppress; coverage is inherently partial, so a passing gate gives false assurance. Resolution: block on High/Critical from the post-merge staging scan with a hard time cap and coverage assertion; run the full scan nightly non-blocking but SLA-tracked in DefectDojo; make the block overridable only via a recorded risk acceptance with expiry.

### Q26. Where does ZAP fit relative to Nuclei and Burp in a mature AppSec programme, and what would you drop if you had to? (Principal)

**Answer.** ZAP is the automated default: proxy, spiders, spec import, auth, active rules, free, CI-native. Nuclei adds fast, explicit checks for known CVEs and misconfigurations and is ideal for continuous attack-surface regression; it has no crawl or auth flows. Burp is for humans: business logic, multi-step auth, BOLA. If forced to drop one, drop Nuclei: ZAP's active rules and custom scripts can approximate its checks, albeit slower, while neither ZAP nor Nuclei replaces a human with Burp. If cost were the constraint, Burp Enterprise would go before Burp Pro seats.

### Q27. How would you make ZAP results trustworthy enough for SOC 2 / ISO 27001 evidence? (Principal)

**Answer.** Reproducibility: image pinned by digest, add-on auto-update off, plan and rules in Git with review history. Completeness: coverage assertions per plan, URL-count trend alerts, spec-vs-site-tree diff to detect shadow endpoints. Traceability: `commit_hash`, `build_id`, `branch_tag`, `environment` on every DefectDojo import; retention policy on reports; SLA metrics exported. Independence: the security team owns the policy and rules repo; service teams cannot merge suppressions alone. Periodic calibration against manual Burp tests to estimate false-negative rate.

### Q28. Describe how you would detect that a DAST scan silently lost coverage three months ago. (Principal)

**Answer.** Trend the number of URLs added and requests sent per plan per service (from AF stats or `exportdata`), and the ratio of 2xx to 401/403 responses, into Prometheus; alert on a drop greater than 30% week-over-week. Enforce `tests` assertions so a drop fails the job. Periodically diff the spec operations against the scanned site tree. In DefectDojo watch for a sudden mass mitigation (`close_old_findings` closing everything) which is a classic sign of a zero-coverage scan.

### Q29. How does running ZAP interact with Gate 7 runtime detection, and how do you avoid blinding the SOC? (Principal)

**Answer.** Active scans look like attacks: Falco rules such as outbound connections from the scanner pod, Wazuh rules on auth failures and 5xx bursts, WAF blocks. Options: suppress by identity (Falco exceptions keyed on scanner pod labels, Wazuh rules matching the scanner user agent, WAF allow-list for the scanner IP) with a pre-announced window; or, better, do not suppress but *correlate*: tag scan windows in the SIEM so alerts in the window are auto-triaged as "expected: DAST" while anything from a non-scanner identity still pages. The second approach also validates that Gate 7 would detect a real attacker performing the same actions.

### Q30. You inherit a ZAP setup that uses `-I`, a 300-line `rules.tsv` of IGNOREs, and scans production nightly. What do you do in the first month? (Principal)

**Answer.** Week 1: stop the production scan immediately (authorisation and data integrity risk), confirm with change management, and redirect to staging. Week 2: baseline the current rules file against an untuned run to see what the IGNOREs hide; triage real findings into DefectDojo with risk acceptances. Week 3: rewrite as an AF plan with contexts, coverage tests, and narrow alert filters; remove `-I`, start with `warnLevel: High`. Week 4: wire SARIF, DefectDojo reimport, Falco/Wazuh correlation, and ownership (CODEOWNERS). Report metrics before and after: finding counts, scan duration, and suppressions with justification.
