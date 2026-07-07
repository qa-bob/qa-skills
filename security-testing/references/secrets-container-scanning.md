# Secrets & Container/Image Scanning

Two narrow, high-signal scans that belong in every pipeline regardless of how mature the broader security testing program is — both catch mistakes rather than sophisticated attacks, which is exactly why they're cheap to run and embarrassing to skip.

---

## Secrets Scanning

**What it catches:** hardcoded API keys, database credentials, private keys, and tokens committed to source — either in the current tree or buried in git history.

### Tool selection

| Tool | Notes |
|---|---|
| Gitleaks | Fast, good default rule set, easy pre-commit hook integration |
| TruffleHog | Strong at verifying whether a found secret is still *live* (calls the provider's API to check), not just pattern-matching — dramatically cuts false-positive triage time |
| GitHub Secret Scanning / GitLab Secret Detection | Native, zero-setup if already on that platform; partners with the platform to auto-revoke some credential types on detection |

### Where to run it

```
1. Pre-commit hook (local) — catches the mistake before it ever reaches a shared branch
2. CI on every push/PR — catches anything that slipped past a skipped or missing hook
3. Full history scan (periodic, not per-commit) — catches secrets committed in the past
   that were later "removed" in a subsequent commit but still live in git history,
   retrievable by anyone who clones the repo
```

Full-history scans matter because deleting a file in a new commit does not delete it from history — `git log -p` (or a targeted scan tool) can still surface it. If a secret is found in history, the fix is not "delete it in a new commit," it's rotate the credential immediately, then decide separately whether history rewriting is worth the disruption.

### Verifying a finding before treating it as urgent

Not every pattern match is a live secret — plenty are expired test keys, obviously-fake example values (`sk-test-000...`), or values already rotated. Prioritize triage using verification-capable tools (TruffleHog's live-check) or a manual check against the provider, but treat *any* finding in a public or broadly-accessible repo as urgent to triage quickly even if it turns out to be a false positive — the cost of checking is minutes, the cost of an unrotated live credential sitting exposed is not.

### CI wiring pattern

```yaml
- name: Secrets scan
  uses: gitleaks/gitleaks-action@v2
  env:
    GITLEAKS_ENABLE_UPLOAD_ARTIFACT: false  # don't persist findings if scanning a private repo with sensitive matches
```

---

## Container / Image Scanning

**What it catches:** known CVEs in OS packages and libraries baked into a container image — a different surface than SCA (which scans your application's declared dependencies), since a base image carries its own OS-level package set that your `package.json`/`requirements.txt` never mentions.

### Tool selection

| Tool | Notes |
|---|---|
| Trivy | Fast, broad coverage (OS packages, language dependencies, IaC misconfig, and secrets in a single tool), good default |
| Grype | Similar coverage to Trivy, strong integration with Syft for SBOM generation |
| Docker Scout | Native to Docker Desktop/Hub workflows |

### What to scan, and when

```
1. Base image itself, before building on top of it — a stale/vulnerable base image
   propagates every CVE it carries into everything built from it
2. Final built image, in CI, before push to registry — catches anything introduced
   by the build (added packages, copied artifacts) on top of the base image
3. Images already running in production, periodically — CVE databases update daily;
   an image that was clean at build time can become "vulnerable" a week later
   purely because a new CVE was disclosed against a package it contains
```

### CI wiring pattern

```yaml
- name: Container scan
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: 'myapp:${{ github.sha }}'
    severity: 'CRITICAL,HIGH'
    exit-code: '1'  # fail the build on high/critical — tune down if this is the first rollout
```

### Practical calibration

Same lesson as SAST in `sast-dast-integration.md`: a first-time scan against a mature image often surfaces dozens of findings, many in base-OS packages the team doesn't control and can't immediately fix. Baseline the current state, gate new builds only on *new* criticals introduced by the current change, and treat the base-image backlog as a separate, prioritized cleanup rather than a blocker on every subsequent build.

---

## Checklist

```
- [ ] Secrets scanning runs pre-commit AND in CI — not just one or the other
- [ ] At least one full-history secrets scan has been run, not just scans of the current tree
- [ ] Any live-credential finding is rotated immediately, independent of whether history gets rewritten
- [ ] Container scanning covers the base image AND the final built artifact, not just the Dockerfile source
- [ ] Scan severity gates are calibrated (baseline + new-findings-only) so the pipeline blocks on signal, not noise
```
