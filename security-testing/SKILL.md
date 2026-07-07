---
name: security-testing
description: Plan and execute security testing for applications and APIs — OWASP Top 10 test case design, SAST/DAST tool integration, secrets and container scanning, and penetration-test scoping. Use this skill whenever the user asks to test for vulnerabilities, write security test cases, set up a security scan in CI, plan a pentest, harden an app before release, or mentions OWASP, SAST, DAST, injection, auth bypass, secrets scanning, or container/dependency scanning — even if they don't use the words "security testing" explicitly (e.g. "make sure this API can't be broken into" or "check this before it goes to prod").
---

# Security Testing

Security testing answers one question with evidence: *can this system be made to do something it shouldn't?* That's a different question from functional correctness (does it do what it's supposed to?) and it needs a different mindset — actively trying to break the intended behavior rather than confirming it. This skill provides that mindset as a repeatable methodology: how to scope a security test effort, which test cases to write for which risk, how to wire automated scanning into a pipeline, and how to plan a penetration test engagement.

**Boundary:** this skill is for testing systems you own or have explicit authorization to test — the same authorization boundary that applies to any legitimate security assessment. It does not provide exploit development, malware, or techniques for accessing systems without authorization. Every technique here is standard defensive-QA practice: the kind of test a security-conscious engineering team runs against its own product before shipping it.

---

## When to reach for each part of this skill

| Situation | Do this |
|---|---|
| Writing test cases for a specific vulnerability class (injection, broken auth, XXE, etc.) | Read `references/owasp-top-10-test-cases.md` |
| Setting up automated static/dynamic scanning in CI | Read `references/sast-dast-integration.md` |
| Checking for hardcoded credentials or vulnerable dependencies/images | Read `references/secrets-container-scanning.md` |
| Scoping or coordinating a human-run penetration test | Read `references/pentest-planning.md` |
| None of the above — just getting oriented | Keep reading this file |

---

## The Four Layers of Security Testing

Security testing isn't one activity — it's four layers with different cost, speed, and depth tradeoffs. A mature test strategy uses all four; a resource-constrained one at least knows which layers it's skipping and why.

```
1. SAST (Static)      — scan source code without running it. Fastest, cheapest, catches
                         known-bad patterns. Misses anything that only manifests at runtime.
2. SCA (Dependencies)  — scan declared dependencies for known CVEs. Catches inherited risk
                         from libraries, not risk introduced by your own code.
3. DAST (Dynamic)      — attack a running instance like an external actor would. Catches
                         real runtime behavior SAST can't see. Slower, needs a live target.
4. Manual / Pentest    — a human (or an LLM operating with genuine adversarial intent, not
                         a checklist) actively tries to break business logic, auth flows,
                         and anything requiring contextual judgment no scanner has.
```

Layers 1-3 are automatable and belong in CI (see `sast-dast-integration.md`). Layer 4 requires either a scoped internal exercise or an external pentest (see `pentest-planning.md`) and is the only layer that reliably catches business-logic flaws — e.g. "a discount code can be applied twice by racing two requests," which no scanner will ever flag because nothing about it looks syntactically wrong.

---

## Scoping a Security Test Effort

Before writing a single test case, answer three questions — skipping this step is the most common reason security testing produces a pile of low-value findings instead of a risk-reduction plan:

1. **What's actually exposed?** Enumerate the real attack surface: authenticated vs. unauthenticated endpoints, anything that accepts user-supplied input (including headers, file uploads, and deserialized payloads — not just form fields), anything with a trust boundary crossing (client→server, service→service, tenant→tenant).
2. **What's the worst plausible outcome per surface?** Rank by impact, not by how interesting the bug is to find. A stored XSS in an internal admin tool used by three trusted employees is a different risk than an auth bypass on a public payment endpoint, even though both might score identically on a generic severity scale.
3. **What's already covered, and by which layer?** Check whether SAST/SCA is already running in CI before writing manual test cases for things a scanner already catches for free. Manual effort should concentrate on what automation structurally cannot find: business logic, auth/authz edge cases, multi-step workflows, and anything requiring domain knowledge of what "correct" means for this specific product.

This produces a short, risk-ordered list of what to actually test — not "run the OWASP Top 10 against everything," which treats a public marketing page and a payment API as equally worth the same test budget.

---

## Core Technique: Prove It, Don't Just Flag It

The single most common failure mode in security testing — automated or manual — is reporting a *theoretical* weakness as if it were a *demonstrated* one. "This endpoint doesn't validate input length" is an observation. "This endpoint accepts a 50MB payload with no limit, and repeated requests degrade response time from 40ms to 4s, demonstrating a resource-exhaustion path" is a finding. Every security test case in this skill's reference files is written to produce the second kind of statement: a concrete payload, an expected pass/fail signal, and — where relevant — a measurable impact.

This matters for two reasons: it's the difference between a report engineering teams act on and one they triage into a backlog forever, and it's the only way to avoid false positives that erode trust in the whole test effort (a team that gets burned by three "critical" findings that turn out to be non-exploitable will start ignoring the fourth one, which might be real).

---

## Decision Tree — Which Layer to Reach For

```
Is this a pre-merge / pre-deploy gate?
├── Yes → SAST + SCA in CI (fast, deterministic, blocks on known-bad patterns and CVEs)
│         → references/sast-dast-integration.md, references/secrets-container-scanning.md
└── No, this is a pre-release or periodic assessment
    ├── Target is a running service/environment → DAST pass
    │   → references/sast-dast-integration.md (DAST section)
    ├── Testing a specific vulnerability class someone flagged → OWASP-mapped manual test cases
    │   → references/owasp-top-10-test-cases.md
    └── Need business-logic / auth-flow / multi-step-workflow coverage,
        or this is a compliance/audit requirement → scope a pentest
        → references/pentest-planning.md
```

---

## Reporting

Whatever layer produced the finding, report it the same way — consistency is what lets a team triage findings from four different tools/testers without re-learning a new format each time:

```markdown
### [SEVERITY] Short, specific title (not "SQL Injection" — "SQL injection in /api/orders?sort=")
**Where:** exact endpoint / file:line / parameter
**Proof:** the exact request/payload used and the observed response that demonstrates impact
**Impact:** what an attacker could actually do with this — be concrete, not generic
**Fix:** the specific remediation (not "sanitize input" — "use parameterized queries via the existing ORM's query builder, see similar pattern in src/db/users.py:88")
**Retest:** the exact step to confirm the fix worked
```

Severity should reflect *exploitability × impact* in this system's actual context, not a generic CVSS-style default. A reflected XSS on an internal tool behind SSO and a network ACL is not "Critical" just because the OWASP category is usually severe — context changes the number.

---

## Checklist

```
- [ ] Attack surface enumerated before writing test cases (not "test everything against the OWASP list")
- [ ] Findings ranked by actual impact in this system's context, not generic severity defaults
- [ ] SAST/SCA running in CI before investing manual effort on what automation already covers
- [ ] Every finding includes a concrete proof (request/payload + observed response), not just a description
- [ ] Business-logic and multi-step-workflow risks identified as needing manual/pentest coverage, not assumed to be caught by scanners
- [ ] Report includes a specific fix and a retest step, not just a description of the problem
```
