# SAST & DAST Integration

Automated scanning is the highest-leverage layer of security testing — it's the only layer that runs on every commit at near-zero marginal cost per run. The goal here isn't to pick "the best tool," it's to wire scanning in so it actually blocks bad changes rather than producing a report nobody reads.

---

## SAST (Static Application Security Testing)

Scans source code for known-bad patterns without executing it. Fast enough to run pre-merge.

### Tool selection

| Tool | Best for | Notes |
|---|---|---|
| Semgrep | Fast, low-noise, custom rules per codebase | Good default — free tier covers most languages, rules are readable and easy to tune |
| CodeQL | Deep, cross-file taint tracking | Slower, more thorough; GitHub-native if already on GitHub Actions |
| Bandit | Python-specific | Lightweight, good for a Python-only codebase where a general tool is overkill |
| SonarQube | Combined code-quality + security | Good if the team already wants a quality-gate dashboard beyond just security |

### CI wiring pattern

```yaml
# .github/workflows/sast.yml (GitHub Actions example — same shape applies to GitLab/Jenkins/Azure)
name: SAST
on: [pull_request]
jobs:
  semgrep:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: returntocorp/semgrep-action@v1
        with:
          config: p/owasp-top-ten  # curated ruleset; add custom rules alongside
```

### Making it actually block bad changes

The single biggest reason SAST scanning fails to change anything: it runs, produces 200 findings on day one, the team can't triage that volume, and everyone starts ignoring the report. Avoid this:

1. **Baseline first.** Run the scan once against the current codebase, accept the existing findings as a known baseline (most tools support a baseline/ignore file), and gate only on *new* findings from that point forward.
2. **Start with a narrow, high-confidence ruleset** (e.g. `p/owasp-top-ten` in Semgrep) rather than every rule the tool ships with. Expand once the team trusts the signal.
3. **Fail the build only on high/critical severity initially.** Route medium/low to a dashboard for periodic triage rather than blocking every PR on them — a scanner that blocks merges on low-signal findings trains developers to bypass it.

---

## SCA (Software Composition Analysis)

Scans declared and transitive dependencies for known CVEs. Distinct from SAST — SAST looks at *your* code, SCA looks at *code you depend on*.

| Tool | Ecosystem |
|---|---|
| `npm audit` / `pnpm audit` | Node.js |
| `pip-audit` | Python |
| OWASP Dependency-Check | Java, and cross-ecosystem |
| Snyk | Multi-ecosystem, good CVE-to-fix-PR automation |
| Dependabot / Renovate | Automated PRs for vulnerable dependency bumps — pairs with any scanner above |

Run SCA against the **lockfile**, not just the manifest — `package.json` declares intent, `package-lock.json`/`poetry.lock`/`go.sum` declares what's actually resolved and shipped, including transitive dependencies that never appear in the manifest at all.

---

## DAST (Dynamic Application Security Testing)

Attacks a running instance the way an external actor would — catches runtime behavior (auth flows, session handling, actual server responses) that static analysis structurally cannot see.

### Tool selection

| Tool | Best for | Notes |
|---|---|---|
| OWASP ZAP | General-purpose, free, scriptable | Good default; supports both automated baseline scans and authenticated scans with a login script |
| Burp Suite | Manual-assisted testing, deep proxy/repeater workflow | Better for the human-in-the-loop layer 4 work than for pure CI automation (Community edition has automation limits) |
| Nuclei | Fast, template-based scanning against known CVE signatures | Good for quickly checking a target against a large, community-maintained template library |

### CI wiring pattern (baseline scan against a staging deploy)

```yaml
# Run after deploy-to-staging, against a live URL — DAST needs a running target
- name: ZAP Baseline Scan
  uses: zaproxy/action-baseline@v0.12.0
  with:
    target: 'https://staging.example.com'
    rules_file_name: '.zap/rules.tsv'  # tune out known false positives
```

### Authenticated scanning matters

An unauthenticated DAST scan only covers the public surface — the login page and anything reachable without a session. Most of the real attack surface (everything behind auth) is invisible to it. Configure the scanner with a login script or session token so it crawls and attacks the authenticated application, not just the marketing pages in front of it.

### DAST needs a real environment, with real constraints

Never point DAST at production without an explicit, informed go-ahead from whoever owns that environment — active scanning sends real attack payloads and can trigger rate limits, WAF blocks, alerting noise, or in rare cases actual side effects (a scanner blindly fuzzing every form field might submit real orders, real emails, or real account-lockout triggers). Standard practice: DAST against a staging environment that mirrors production, with synthetic test accounts and — for anything genuinely production-only — a narrowly scoped, pre-agreed exception window.

---

## Sequencing in a pipeline

```
PR opened
  → SAST + SCA run (fast, source-only, no live target needed) — block merge on high/critical
  → merge → deploy to staging
    → DAST baseline scan against staging — block promotion to prod on high/critical
      → (periodic, not per-deploy) manual/pentest layer — see pentest-planning.md
```

SAST/SCA gate the merge; DAST gates the promotion; manual/pentest is a periodic or milestone-triggered activity, not a per-deploy gate — it's too slow and too expensive to run at that cadence.
