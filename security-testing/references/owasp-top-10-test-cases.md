# OWASP Top 10 — Concrete Test Cases

Each category below gives a specific payload or technique, the observable pass/fail signal, and what it actually proves. These are written for testing systems you own or are authorized to test. Adapt payloads to the specific input format (JSON body, query param, header, form field) — the technique matters more than the exact string.

Reference categories follow OWASP Top 10 (2021 edition, still current as of this writing); check owasp.org for the latest edition before a formal engagement, as OWASP periodically revises the list.

---

## A01: Broken Access Control

**What to test:** Can a user reach data or actions outside their authorization scope?

| Test | How | Pass/Fail signal |
|---|---|---|
| Horizontal privilege escalation | As User A, request User B's resource by ID: `GET /api/orders/12345` where 12345 belongs to another user | FAIL if 200 + B's data returned. Should be 403/404 |
| Vertical privilege escalation | As a non-admin, call an admin endpoint directly: `POST /api/admin/users/delete` | FAIL if the action succeeds |
| IDOR via sequential IDs | Increment/decrement a resource ID in an authenticated request | FAIL if adjacent resources belonging to other tenants/users are accessible |
| Missing function-level check | Access a UI-hidden feature's underlying API directly (the button is hidden, but is the endpoint actually protected?) | FAIL if the endpoint doesn't independently enforce the same authorization the UI implies |
| CORS misconfiguration | Send `Origin: https://evil.example.com` and check the `Access-Control-Allow-Origin` response header | FAIL if it reflects the arbitrary origin AND `Access-Control-Allow-Credentials: true` is also set |

## A02: Cryptographic Failures

**What to test:** Is sensitive data protected in transit and at rest, using primitives that are actually still considered strong?

| Test | How | Pass/Fail signal |
|---|---|---|
| Transport encryption | Attempt the same request over plain HTTP if any endpoint accepts it | FAIL if sensitive data is accepted/returned over unencrypted transport |
| Weak/deprecated algorithms | Inspect TLS config (`nmap --script ssl-enum-ciphers`) and any custom crypto in code | FAIL on MD5/SHA1 for password hashing, ECB mode, or TLS 1.0/1.1 still enabled |
| Sensitive data in logs | Trigger a login failure, a payment, or a password-reset flow and grep the resulting logs | FAIL if raw passwords, full card numbers, or tokens appear in plaintext logs |
| Key management | Check whether encryption keys are hardcoded, checked into source, or rotated on a defined schedule | FAIL if keys are static and embedded rather than pulled from a secrets manager |

## A03: Injection

**What to test:** Does user-controlled input ever reach an interpreter (SQL, OS shell, LDAP, template engine, NoSQL query) without being treated as pure data?

| Test | How | Pass/Fail signal |
|---|---|---|
| SQL injection (error-based) | Submit `' OR '1'='1` or `'; DROP TABLE users; --` in any field that reaches a query | FAIL if a DB error leaks schema info, or the boolean logic visibly changes results |
| SQL injection (blind, time-based) | Submit `' OR SLEEP(5)--` (or DB-appropriate equivalent) | FAIL if response time increases by ~5s, proving the payload executed |
| Command injection | Submit `; whoami` or `` `id` `` in any field that might reach a shell call | FAIL if command output leaks into the response, or a measurable side effect occurs |
| NoSQL injection | Submit `{"$gt": ""}` as a login password in a MongoDB-backed app | FAIL if authentication succeeds without a valid password |
| Template injection (SSTI) | Submit `{{7*7}}` or `${7*7}` where user input reaches a template renderer | FAIL if the response contains `49` instead of the literal string |
| Second-order injection | Store a payload via one field (e.g. display name), then trigger a *different* feature that reads it (e.g. an admin report) | FAIL if the payload executes in the second context, even though the first context escaped it correctly |

## A04: Insecure Design

**What to test:** Is the *intended* behavior exploitable, independent of any implementation bug? This is the category manual testing exists for — no scanner finds these.

| Test | How | Pass/Fail signal |
|---|---|---|
| Business logic abuse | Race two requests that should be mutually exclusive (redeem the same discount code twice, withdraw funds twice before balance updates) | FAIL if both succeed when only one should |
| Missing rate limiting on sensitive action | Hammer a password-reset or OTP-verification endpoint | FAIL if there's no lockout/backoff, enabling brute force |
| Workflow step-skipping | Call step 3 of a multi-step flow (e.g. checkout) directly, skipping steps 1-2 (e.g. payment authorization) | FAIL if the system accepts the out-of-order call as if prior steps completed |
| Trust boundary in client-side logic | Look for price, discount, or permission decisions made in client-side JS and re-submit a tampered value server-side | FAIL if the server trusts the client-supplied value instead of recomputing it |

## A05: Security Misconfiguration

**What to test:** Are defaults, debug features, or verbose errors exposed that shouldn't be?

| Test | How | Pass/Fail signal |
|---|---|---|
| Debug mode in production | Request a path likely to trigger an unhandled exception | FAIL if a stack trace, framework version, or file paths leak in the response |
| Default credentials | Check any admin panel, database, or management interface for unchanged defaults | FAIL if `admin/admin` or vendor-default creds work |
| Directory listing | Request a directory path directly (`/uploads/`, `/backup/`) | FAIL if the server returns a file listing instead of 403/404 |
| Unnecessary exposed services/ports | Port-scan the target's public-facing IPs | FAIL if internal-only services (DB, admin dashboards, metrics endpoints) are reachable externally |
| Missing security headers | Inspect response headers on any page | FAIL if `Content-Security-Policy`, `X-Content-Type-Options`, `Strict-Transport-Security` are absent where they're expected for the risk profile |

## A06: Vulnerable and Outdated Components

**What to test:** Are declared dependencies and container base images free of known CVEs? Covered in depth in `secrets-container-scanning.md` — the short version: run `npm audit` / `pip-audit` / `trivy image` / Snyk / equivalent against the manifest and the built artifact, not just the source tree, since transitive dependencies and OS packages baked into a container image are a different attack surface than the direct `package.json`.

## A07: Identification and Authentication Failures

**What to test:** Can authentication be bypassed, brute-forced, or session integrity broken?

| Test | How | Pass/Fail signal |
|---|---|---|
| Credential stuffing / brute force | Submit repeated login attempts with varying passwords for one account | FAIL if there's no rate limit, CAPTCHA, or account lockout after N attempts |
| Session fixation | Set a known session ID before login, then check if it's reused post-login | FAIL if the session ID doesn't rotate on privilege change (login) |
| Session token exposure | Check whether session tokens appear in URLs, browser history, or referrer headers | FAIL if a token that grants access appears anywhere logs/history would capture it |
| Weak password policy | Attempt to set a password like `password1` or a password matching the account's own email | FAIL if accepted with no complexity/breach-list check |
| MFA bypass | After enabling MFA, attempt to reach an authenticated resource via an older/alternate auth flow (e.g. an API token that predates MFA enrollment) | FAIL if the alternate path skips the MFA requirement entirely |

## A08: Software and Data Integrity Failures

**What to test:** Can the update/deserialization/CI pipeline be tampered with?

| Test | How | Pass/Fail signal |
|---|---|---|
| Insecure deserialization | Submit a crafted serialized object (language-specific — e.g. a Python pickle, Java serialized object, PHP object string) where user input reaches a deserializer | FAIL if the deserializer executes arbitrary logic instead of rejecting untrusted input formats |
| Unsigned/unverified updates | Check whether auto-update mechanisms verify a signature before applying | FAIL if any binary/package is fetched and applied without integrity verification |
| CI/CD pipeline integrity | Check whether build artifacts are signed/attested and whether pipeline config changes require review | FAIL if a single compromised credential could alter build output undetected |

## A09: Security Logging and Monitoring Failures

**What to test:** Would an actual attack be detected? This isn't a single payload — it's an audit.

| Test | How | Pass/Fail signal |
|---|---|---|
| Auth failure visibility | Trigger 10 failed logins for one account | FAIL if no log entry, alert, or metric reflects this pattern |
| Privilege-escalation attempt visibility | Attempt an A01-style unauthorized access | FAIL if the attempt isn't logged with enough context (who, what, when) to investigate later |
| Log tamper resistance | Check whether logs are append-only / shipped off-host in near-real-time | FAIL if an attacker with app-level access could plausibly delete or rewrite local logs before detection |

## A10: Server-Side Request Forgery (SSRF)

**What to test:** Can the server be tricked into making requests to attacker-chosen destinations, especially internal-only ones?

| Test | How | Pass/Fail signal |
|---|---|---|
| Basic SSRF | Submit a URL-accepting field (webhook URL, image-fetch-by-URL, PDF-from-URL feature) pointing to an internal address, e.g. `http://169.254.169.254/latest/meta-data/` (cloud instance metadata) or `http://localhost:<internal-port>/` | FAIL if the response reflects internal data back, proving the server fetched it |
| DNS rebinding / redirect bypass | If the app validates the URL's host but then follows redirects, host a redirect from an allowed domain to an internal address | FAIL if the app follows the redirect to the internal target despite the initial validation |
| Protocol smuggling | Submit `file://`, `gopher://`, or `dict://` schemes where only `http(s)://` is intended | FAIL if the server doesn't restrict to an explicit allow-list of schemes |

---

## Using these test cases well

Map each finding back to the scoping exercise in `SKILL.md` — a working SSRF against an internal metadata endpoint on a cloud-hosted service is a very different severity than the same technique against a service with no cloud metadata endpoint to reach. The payload proves the mechanism; the context determines whether it matters.
