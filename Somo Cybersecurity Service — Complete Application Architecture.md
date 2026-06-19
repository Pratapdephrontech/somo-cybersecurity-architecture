# Somo Cybersecurity Service — Complete Application Architecture

**32 Scanners · Full OWASP Top 10 Coverage**
**Generated: 2026-06-19**

---

## Overview

The Somo Cybersecurity Service is a standalone web application that accepts a target URL as input, runs a complete automated penetration test using 32 security scanners organized into 8 groups, and produces a scored PDF/JSON report with remediation recommendations.

### Technology Stack

| Component | Technology |
|---|---|
| Frontend | React + Vite + Redux Toolkit + RTK Query |
| Backend API | Node.js + Express |
| Queue | Bull + Redis (async scan jobs) |
| Database | PostgreSQL (scan history + findings) |
| Storage | AWS S3 (PDF reports, 7-day TTL) |
| Proxy | Nginx (TLS termination + routing) |
| Containers | Docker + AWS ECS |

---

## Architecture Layers

The system is organized into 7 layers, each with a specific responsibility.

---

## Layer 1 — Client (React Dashboard)

The React frontend has 4 main pages:

**Page 1 — URL Input**
- Target URL entry field
- Optional scan config (scope, auth tokens, exclusions)
- Scan scope selector: Passive Only / Full Scan / Custom

**Page 2 — Scan Progress (Live via WebSocket)**
- Progress bar per scanner group (8 groups)
- Live status showing which scanner is currently running
- Completed group checkmarks
- Estimated time remaining

**Page 3 — Report Dashboard**
- Score Card: 0–100 score + A / B / C / D / F grade
- OWASP Top 10 coverage grid (per-category scores)
- Findings Table (grouped by severity: Critical / High / Medium / Low)
- Remediation steps per finding
- Download PDF and Download JSON buttons

**Page 4 — History**
- List of all past scans with scores
- Compare two scans side by side (trend view)
- Score trend line chart over time

**State Management**

Redux Store + RTK Query (cyberApiSlice) with these endpoints:

| Endpoint | Method | Purpose |
|---|---|---|
| startScan | POST /api/scan | Start a new scan job |
| getScan | GET /api/scan/:id | Poll job status and partial results |
| getReport | GET /api/report/:id | Fetch full report JSON |
| getScanHistory | GET /api/history | List all past scans |
| WebSocket | WS /api/scan/:id/progress | Live progress stream |

---

## Layer 2 — API Gateway (Nginx)

Nginx sits in front of the Scanner API and handles:

- TLS termination (HTTPS only, HTTP redirected)
- Rate limiting at the network level (first line of defence)
- Request logging (access logs forwarded to CloudWatch)
- WAF rules blocking common attack signatures
- Reverse proxy to Scanner API on port 4500
- Static file serving for the React Dashboard

---

## Layer 3 — Scanner API (Node.js + Express, Port 4500)

**Middleware Chain** — applied to every incoming request:

```
validateUrl → requireApiKey → rateLimiter (5/min) → helmet → cors → logger
```

**Security controls on the API itself:**
- validateUrl: SSRF guard — blocks private IPs (10.x, 192.168.x, 127.x, localhost)
- requireApiKey: SHA-256 hashed key lookup in PostgreSQL
- rateLimiter: 5 scans per minute per IP via express-rate-limit
- helmet: sets all secure HTTP headers on API responses
- Error handler: never exposes stack traces in production

**API Routes:**

| Method | Route | Description |
|---|---|---|
| POST | /api/scan | Queue scan job, return jobId immediately (HTTP 202) |
| GET | /api/scan/:id | Poll job status and partial results |
| GET | /api/report/:id | Full report JSON |
| GET | /api/report/:id/pdf | Download PDF from S3 pre-signed URL |
| GET | /api/history | Past scans list |
| WS | /api/scan/:id/progress | Live scanner progress via WebSocket |
| GET | /health | Health check (no auth required) |

---

## Layer 4 — Queue and Workers (Bull + Redis)

**Bull Queue states:**

```
queued → active → completed
                → failed (retry up to 2 times)
                → stalled
```

**Scan Orchestrator (Worker):**
- Picks up jobs from the Bull queue
- Runs all 8 scanner groups in sequence
- Within each group, all scanners run in parallel (Promise.allSettled)
- Emits a WebSocket progress event after each group completes
- Group 1 (Recon) output is stored and passed into Group 8 (CVE scanner)
- Total scan timeout: 5 minutes per job
- Retry: 2 attempts on failure

---

## Layer 5 — Scanner Modules (32 Scanners in 8 Groups)

---

### Group 1 — Passive Recon

**Timeout: 15 seconds | Runs first | Output feeds Groups 7 and 8**

| Scanner | Tool | What It Checks |
|---|---|---|
| tech | Wappalyzer | Framework, CMS version, CDN, server software. Output used by CVE scanner |
| dns | Node dns module | A / MX / TXT / NS records, SPF, DKIM, DMARC, zone transfer test |
| ports | nmap | Open ports (80, 443, 8080, 8443, 3000, 4000, 22, 21, 25), banner grab |
| serverinfo | axios | Server header, X-Powered-By, X-AspNet-Version, version disclosure |

---

### Group 2 — Transport Security

**Timeout: 20 seconds**

| Scanner | Tool | What It Checks |
|---|---|---|
| ssl | ssl-checker npm | Certificate expiry, TLS version (1.2+ required), weak ciphers, chain validity |
| headers | axios | HSTS, X-Frame-Options, Content-Security-Policy, X-Content-Type-Options, Referrer-Policy, Permissions-Policy |
| redirect | axios | HTTP → HTTPS enforcement, open redirect in params, redirect chain loops, mixed content |

**Severity thresholds for SSL:**
- Certificate expiring in less than 14 days → Critical
- Certificate expiring in less than 30 days → High
- Missing CSP header → High
- Missing HSTS header → High

---

### Group 3 — Injection Attacks

**Timeout: 60 seconds**

| Scanner | What It Checks | Example Payloads |
|---|---|---|
| xss | Reflected XSS, Stored XSS probe, DOM-based XSS | `<script>alert(1)</script>`, `<img src=x onerror=alert(1)>` |
| sqli | Error-based, Boolean-blind, Time-based blind, UNION injection | `' OR 1=1--`, `UNION SELECT NULL--`, `1; WAITFOR DELAY '0:0:5'--` |
| cmdi | OS command injection via inputs and path parameters | `; ls`, `\| whoami`, `&& id`, backtick id backtick |
| ssti | Server-Side Template Injection (Jinja2, Twig, EJS, Freemarker) | `{{7*7}}`, `${7*7}`, `<%= 7*7 %>`, `#{7*7}` |
| xxe | XML External Entity on XML-accepting endpoints | `<!DOCTYPE foo [<!ENTITY xxe SYSTEM "file:///etc/passwd">]>` |

**All injection findings rated High or Critical.**

---

### Group 4 — Authentication and Session

**Timeout: 30 seconds**

| Scanner | What It Checks |
|---|---|
| auth | Default credentials (admin/admin, root/root), account lockout after failures, username enumeration via response differences, timing attack on login |
| jwt | alg:none attack, weak secret detection (wordlist), algorithm confusion (HS256 vs RS256 swap), missing exp claim, sensitive data in payload |
| session | Session fixation, session token entropy, session invalidation after logout, concurrent session limit, token in URL |
| csrf | CSRF token on state-changing forms, SameSite cookie enforcement, X-Requested-With header, double submit cookie pattern |

---

### Group 5 — Access Control

**Timeout: 30 seconds**

| Scanner | What It Checks |
|---|---|
| idor | Sequential ID enumeration (/api/users/1, /api/users/2), UUID vs integer patterns, object reference swapping, forced browsing |
| methods | HTTP verb abuse — PUT, DELETE, PATCH, TRACE, OPTIONS on restricted endpoints, method tunnelling via X-HTTP-Method-Override, TRACE XST attack |
| dirtraversal | Path traversal (../../../../etc/passwd), null byte injection (%00), URL encoding bypass (%2e%2e%2f), double URL encoding |
| cors | Origin reflection (send evil.com, check if reflected), wildcard * with credentials header, preflight bypass |

---

### Group 6 — Information Disclosure

**Timeout: 20 seconds**

| Scanner | What It Checks |
|---|---|
| exposure | Sensitive file probing: /.env, /.git/config, /.git/HEAD, /backup.zip, /.DS_Store, /config.yml, /wp-config.php, /phpinfo.php, /swagger.json, directory listing |
| errors | Send malformed requests to trigger 400/500 errors, check for stack traces in response, DB error message leakage, internal path disclosure |
| cookies | Secure flag, HttpOnly flag, SameSite attribute, cookie expiry duration, sensitive data in cookie value |
| waf | WAF vendor detection (Cloudflare, AWS WAF, Imperva), basic evasion test, bypass patterns (provides context for other scanners) |

---

### Group 7 — API and Services

**Timeout: 40 seconds**

| Scanner | What It Checks |
|---|---|
| api | Endpoint discovery via fuzzing (common paths wordlist), auth required on all endpoints, mass assignment (extra fields in POST body), undocumented parameters |
| graphql | Introspection enabled (exposes full schema), query depth limit (DoS via deep nesting), batch query abuse, field suggestion leakage |
| websocket | Auth required on WS upgrade, origin header validation on handshake, message flooding/rate limit, protocol abuse (ws:// vs wss://) |
| ratelimit | Send 100+ rapid requests to auth endpoints, X-Forwarded-For IP spoofing bypass, account lockout behaviour, distributed bypass test |

---

### Group 8 — Third-Party and Dependencies

**Timeout: 30 seconds | Uses Group 1 tech.scanner output**

| Scanner | Data Source | What It Checks |
|---|---|---|
| cve | NVD API + OSV database | Known CVEs for detected software versions, CVSS score lookup, exploit availability |
| sri | Page HTML | script tags without integrity= attribute, link tags without integrity=, external CDN assets without SRI |
| thirdparty | Page HTML | Tracking/analytics scripts inventory, scripts loaded over HTTP, known malicious domains in page source |

---

## Layer 6 — Report Engine

The Report Generator package aggregates all 32 scanner outputs and produces a structured report.

**Score Calculator**
- 0–100 overall security score
- Per-group score (8 sub-scores, one per scanner group)
- Letter grade: A (90–100) / B (75–89) / C (60–74) / D (40–59) / F (below 40)
- CVSS-weighted scoring — Critical findings reduce score more than Low findings
- Trend comparison vs previous scan on same target

**Findings Aggregator**
- Deduplicates overlapping findings from multiple scanners
- Sorts by severity (Critical → High → Medium → Low → Pass)
- Assigns a remediation tip to each finding
- Applies confidence scoring to filter false positives
- Groups findings by OWASP Top 10 category

**OWASP Mapper**
- Maps each finding to its OWASP Top 10 category (A01–A10)
- Maps each finding to its CWE ID (Common Weakness Enumeration)
- Adds CVE references where applicable

**PDF Builder (pdfkit)**
- Cover page: target URL, scan date, score, grade
- Executive summary (non-technical overview for management)
- OWASP Top 10 coverage grid with per-category scores
- Detailed findings pages (one page per finding with evidence)
- Remediation roadmap (prioritised fix list)
- Charts: severity distribution, category breakdown

**JSON Export**
- Structured report for programmatic consumption
- Compatible with JIRA / Linear ticket creation workflows
- Webhook delivery option for CI/CD pipeline integration

---

## Layer 7 — Data Layer

### Redis

| Purpose | Details |
|---|---|
| Job queue state | Bull queue job lifecycle (queued / active / completed / failed) |
| Live progress | WebSocket progress data per active scan |
| Result cache | Scan result cache with 24-hour TTL |
| Rate limiting | Per-IP request counters |
| WebSocket sessions | Active WS connection mapping |

### PostgreSQL Tables

**scan_jobs**

| Column | Type | Description |
|---|---|---|
| id | UUID PK | Unique scan identifier |
| target_url | TEXT | The URL that was scanned |
| scan_scope | JSONB | Scope configuration |
| status | ENUM | queued / active / completed / failed |
| score | INT | Overall security score 0–100 |
| grade | CHAR(1) | A / B / C / D / F |
| started_at | TIMESTAMP | When the scan started |
| completed_at | TIMESTAMP | When the scan finished |
| duration_ms | INT | Total scan duration in milliseconds |
| initiated_by | TEXT | API key ID of the requester |

**scan_findings**

| Column | Type | Description |
|---|---|---|
| id | UUID PK | Finding identifier |
| scan_id | UUID FK | References scan_jobs |
| group_name | TEXT | injection / auth / access / etc. |
| scanner | TEXT | xss / sqli / jwt / etc. |
| name | TEXT | Human-readable finding name |
| severity | ENUM | critical / high / medium / low / pass |
| owasp_id | TEXT | e.g. A03 |
| cwe_id | TEXT | e.g. CWE-79 |
| cvss_score | DECIMAL | CVSS numeric score |
| detail | JSONB | Full finding detail |
| evidence | TEXT | Request / response snippet |
| remediation | TEXT | Fix recommendation |

**scan_group_scores**

| Column | Type | Description |
|---|---|---|
| id | UUID PK | Row identifier |
| scan_id | UUID FK | References scan_jobs |
| group | TEXT | Scanner group name |
| score | INT | Group score 0–100 |
| grade | CHAR(1) | Group letter grade |
| findings | INT | Number of findings in group |

**api_keys**

| Column | Type | Description |
|---|---|---|
| id | UUID PK | Key identifier |
| key_hash | TEXT | SHA-256 hash (raw key never stored) |
| owner | TEXT | Owner name or email |
| scans_limit | INT | Max scans per day |
| is_active | BOOL | Whether the key is enabled |
| created_at | TIMESTAMP | Creation date |

### AWS S3

| Item | Detail |
|---|---|
| Bucket | somo-cyber-reports |
| Report path | /reports/{scanId}.pdf and /reports/{scanId}.json |
| Access | Pre-signed URLs only — no public bucket access |
| Expiry | 7-day auto-delete lifecycle rule |

---

## OWASP Top 10 Coverage Map

| OWASP Category | Coverage | Scanners |
|---|---|---|
| A01 Broken Access Control | Full | idor, methods, dirtraversal, cors |
| A02 Cryptographic Failures | Full | ssl, jwt, sri, cookies |
| A03 Injection | Full | xss, sqli, cmdi, ssti, xxe |
| A04 Insecure Design | Full | ratelimit, auth, csrf, api |
| A05 Security Misconfiguration | Full | headers, cors, exposure, errors |
| A06 Vulnerable and Outdated Components | Full | cve, tech, thirdparty, sri |
| A07 Identification and Auth Failures | Full | auth, jwt, session, csrf |
| A08 Software and Data Integrity Failures | Full | sri, thirdparty, cve |
| A09 Security Logging and Monitoring Failures | Partial | errors, serverinfo |
| A10 Server-Side Request Forgery (SSRF) | Full | redirect, api, xxe |

**Overall: 9 of 10 categories fully covered, 1 partially covered.**

---

## Scanner Count Summary

| Group | Scanners | Count |
|---|---|---|
| 1 — Passive Recon | tech, dns, ports, serverinfo | 4 |
| 2 — Transport Security | ssl, headers, redirect | 3 |
| 3 — Injection Attacks | xss, sqli, cmdi, ssti, xxe | 5 |
| 4 — Auth and Session | auth, jwt, session, csrf | 4 |
| 5 — Access Control | idor, methods, dirtraversal, cors | 4 |
| 6 — Info Disclosure | exposure, errors, cookies, waf | 4 |
| 7 — API and Services | api, graphql, websocket, ratelimit | 4 |
| 8 — Dependencies | cve, sri, thirdparty | 3 |
| **Total** | | **32** |

---

## Data Flow — Single Scan Lifecycle

1. User enters target URL and optional config in React Dashboard
2. React validates URL format client-side (basic format check)
3. POST /api/scan sent with URL and scope options
4. validateUrl middleware resolves DNS and blocks private / internal IPs (SSRF guard)
5. requireApiKey middleware looks up hashed API key in PostgreSQL
6. rateLimiter enforces maximum 5 scans per minute per IP
7. Scan job added to Bull queue — API returns HTTP 202 with jobId immediately
8. React opens WebSocket connection to ws://api/scan/:jobId/progress
9. Bull Worker picks up the job from the queue
10. Worker runs Group 1 (Recon) — results stored and passed to Group 8
11. Worker runs Groups 2 through 7 in sequence — scanners within each group run in parallel
12. WebSocket emits a progress event after each group completes (e.g. 25%, 50%, 75%)
13. After all 8 groups complete, Report Generator aggregates all 32 scanner outputs
14. Findings are deduplicated, mapped to OWASP, scored, and written to PDF
15. Report saved to PostgreSQL, PDF uploaded to AWS S3
16. Bull marks job as completed
17. React receives the final WebSocket event and renders the full Report Dashboard
18. User views the report and downloads PDF or JSON

---

## Project Folder Structure

```
somo-cybersecurity/
├── apps/
│   ├── scanner-api/
│   │   └── src/
│   │       ├── scanners/
│   │       │   ├── group1-recon/
│   │       │   │   ├── tech.scanner.js
│   │       │   │   ├── dns.scanner.js
│   │       │   │   ├── ports.scanner.js
│   │       │   │   └── serverinfo.scanner.js
│   │       │   ├── group2-transport/
│   │       │   │   ├── ssl.scanner.js
│   │       │   │   ├── headers.scanner.js
│   │       │   │   └── redirect.scanner.js
│   │       │   ├── group3-injection/
│   │       │   │   ├── xss.scanner.js
│   │       │   │   ├── sqli.scanner.js
│   │       │   │   ├── cmdi.scanner.js
│   │       │   │   ├── ssti.scanner.js
│   │       │   │   └── xxe.scanner.js
│   │       │   ├── group4-auth/
│   │       │   │   ├── auth.scanner.js
│   │       │   │   ├── jwt.scanner.js
│   │       │   │   ├── session.scanner.js
│   │       │   │   └── csrf.scanner.js
│   │       │   ├── group5-access/
│   │       │   │   ├── idor.scanner.js
│   │       │   │   ├── methods.scanner.js
│   │       │   │   ├── dirtraversal.scanner.js
│   │       │   │   └── cors.scanner.js
│   │       │   ├── group6-disclosure/
│   │       │   │   ├── exposure.scanner.js
│   │       │   │   ├── errors.scanner.js
│   │       │   │   ├── cookies.scanner.js
│   │       │   │   └── waf.scanner.js
│   │       │   ├── group7-api/
│   │       │   │   ├── api.scanner.js
│   │       │   │   ├── graphql.scanner.js
│   │       │   │   ├── websocket.scanner.js
│   │       │   │   └── ratelimit.scanner.js
│   │       │   └── group8-deps/
│   │       │       ├── cve.scanner.js
│   │       │       ├── sri.scanner.js
│   │       │       └── thirdparty.scanner.js
│   │       ├── queue/
│   │       │   ├── scan.queue.js
│   │       │   └── scan.worker.js
│   │       ├── routes/
│   │       │   ├── scan.routes.js
│   │       │   └── report.routes.js
│   │       ├── middleware/
│   │       │   ├── validateUrl.js
│   │       │   ├── requireApiKey.js
│   │       │   └── rateLimiter.js
│   │       └── server.js
│   └── cyber-dashboard/
│       └── src/
│           ├── pages/
│           │   ├── ScanPage.jsx
│           │   ├── ScanProgressPage.jsx
│           │   ├── ReportPage.jsx
│           │   └── HistoryPage.jsx
│           ├── components/
│           │   ├── ScoreCard.jsx
│           │   ├── FindingsTable.jsx
│           │   ├── OWASPGrid.jsx
│           │   └── ScanProgress.jsx
│           └── store/
│               └── cyberApiSlice.js
├── packages/
│   └── report-generator/
│       └── src/
│           ├── index.js
│           ├── scoreCalculator.js
│           ├── findingsAggregator.js
│           ├── owaspMapper.js
│           └── pdfBuilder.js
├── infrastructure/
│   ├── docker/
│   │   └── nginx.conf
│   └── ecs/
│       └── task-definition.json
├── docker-compose.yml
├── docker-compose.dev.yml
└── package.json
```

---

## Deployment Architecture (AWS ECS)

**Services:**

| ECS Task | Purpose | Inbound |
|---|---|---|
| Nginx + React Dashboard | Serves static React files, reverse proxy to API | Port 80 / 443 via ALB |
| Scanner API | Node.js Express API, handles scan requests | Port 4500 via Nginx |
| Scan Worker (x2 tasks) | Processes scan jobs from Bull queue | None — private subnet only |

**Data services:**

| Service | Purpose |
|---|---|
| RDS PostgreSQL | Scan history, findings, API keys |
| ElastiCache Redis | Bull queue state, result cache, rate limit counters |
| AWS S3 | PDF and JSON report storage (pre-signed URLs only) |

**Security configuration:**
- Scanner API task role: s3:PutObject to reports bucket prefix only
- Worker tasks: no IAM permissions, no inbound ports, private subnet
- All secrets stored in AWS Secrets Manager (not in task definition env vars)
- No public S3 access — all report downloads via pre-signed URLs
- Scan Worker containers have no inbound network access

---

## Build Priority Order

### Phase 1 — Foundation (Week 1)

| Step | Task |
|---|---|
| 1 | Project scaffold: pnpm workspace, Docker, Redis, PostgreSQL |
| 2 | validateUrl middleware (SSRF protection — build before any scanner) |
| 3 | Bull queue + scan worker skeleton |
| 4 | Group 1 scanners: tech, dns, ports, serverinfo |
| 5 | Group 2 scanners: ssl, headers, redirect |
| 6 | Basic report generator (score calculation + JSON output) |

### Phase 2 — Core Security Tests (Week 2)

| Step | Task |
|---|---|
| 7 | Group 6 scanners: exposure, errors, cookies, waf |
| 8 | Group 5 scanners: cors, methods, dirtraversal, idor |
| 9 | Group 4 scanners: auth, jwt, session, csrf |
| 10 | Group 7 scanners: api, ratelimit, websocket, graphql |

### Phase 3 — Injection and Dependencies (Week 3)

| Step | Task |
|---|---|
| 11 | Group 3 scanners: xss, sqli, cmdi, ssti, xxe |
| 12 | Group 8 scanners: cve, sri, thirdparty |
| 13 | Full report engine with PDF export (pdfkit) |

### Phase 4 — Frontend and Deployment (Week 4)

| Step | Task |
|---|---|
| 14 | React Dashboard: all 4 pages (URL Input, Progress, Report, History) |
| 15 | WebSocket live progress integration |
| 16 | Auth hardening: API key management, rate limit tuning |
| 17 | ECS deployment + CI/CD pipeline via AWS CodePipeline |
