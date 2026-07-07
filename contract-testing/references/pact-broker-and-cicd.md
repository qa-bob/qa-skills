# Pact Broker & CI/CD Integration

The Pact Broker is the shared repository that makes contract testing work as a *system* rather than a pile of disconnected local test artifacts — it's where consumers publish contracts, where providers pull them from for verification, and where the deployment-safety question ("can I actually deploy this?") gets answered with real data instead of a guess.

---

## What the Broker Tracks

```
- Every published pact file, per consumer, per consumer version
- Every verification result, per provider, per provider version, against each pact
- Tags — human-meaningful labels (e.g. "production", "main", a branch name) applied
  to specific consumer/provider versions, marking what's actually deployed where
- The full compatibility matrix this produces: which consumer versions are
  known-compatible with which provider versions, based on real verification results
```

This compatibility matrix is the actual value of the broker — without it, "is it safe to deploy" is a question teams answer by asking around in Slack; with it, it's a query against real, automatically-collected verification data.

---

## Versioning and Tagging Convention

Tag every published pact and every verification result with the environment it represents, not just the raw version/commit SHA:

```bash
# Consumer publishing a pact, tagged with both its git branch and (once merged) production
pact-broker publish ./pacts \
  --consumer-app-version=$GIT_COMMIT_SHA \
  --branch=$GIT_BRANCH \
  --broker-base-url=$PACT_BROKER_URL

# After deploying to production, tag that specific version as deployed there —
# this is what makes "can-i-deploy" checks meaningful later
pact-broker record-deployment \
  --pacticipant=OrderService \
  --version=$GIT_COMMIT_SHA \
  --environment=production
```

Without accurate "what's deployed where" tagging, `can-i-deploy` has nothing real to check against — the tagging discipline is what the whole safety check depends on, so it needs to be wired into the actual deployment pipeline (run automatically on every real deploy), not treated as an optional bookkeeping step.

---

## The `can-i-deploy` Check

Before deploying either a consumer or a provider, ask the broker whether that specific version is verified-compatible with whatever's currently deployed in the target environment:

```bash
pact-broker can-i-deploy \
  --pacticipant=OrderService \
  --version=$GIT_COMMIT_SHA \
  --to-environment=production
```

This returns a pass/fail based on real verification results — if `OrderService` at this version hasn't been verified compatible with whatever `UserService` version is currently tagged `production`, the check fails and the deployment should be blocked. This is the mechanism that turns contract testing from "a nice test suite" into an actual deployment gate, the same way `security-testing`'s CI gates block a merge on unresolved high-severity findings.

---

## CI/CD Pipeline Shape

```
Consumer CI:
  run consumer contract tests → generate pact file → publish pact to broker (tagged with branch/version)
  → can-i-deploy check against target environment → deploy if passed

Provider CI:
  pull relevant pacts from broker → run provider verification against real running provider
  → publish verification results back to broker → can-i-deploy check → deploy if passed
```

Both pipelines independently gate on the same shared broker state — neither side needs to trigger the other's pipeline directly, which is what keeps consumer and provider teams able to ship independently while still being protected from breaking each other.

---

## Webhook-Triggered Re-Verification

Configure the broker to fire a webhook to the provider's CI pipeline whenever a consumer publishes a *new or changed* pact — this closes the loop so a provider finds out about a new consumer expectation quickly, rather than only discovering it the next time the provider happens to run its own CI for an unrelated change. Without this, a newly published contract can sit unverified for an arbitrary length of time, quietly providing no actual protection until something else triggers a provider CI run.

---

## Checklist

```
- [ ] Every consumer/provider deployment is tagged in the broker (record-deployment or equivalent), not just published from CI with no environment tracking
- [ ] can-i-deploy is a real, enforced gate in the deployment pipeline, not an optional manual check
- [ ] Provider CI is triggered by broker webhooks on new/changed consumer contracts, not left to discover them only incidentally
- [ ] Tagging convention distinguishes branch/version tags from environment-deployed tags — these answer different questions and shouldn't be conflated
```
