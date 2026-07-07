# Provider Verification

The provider's job is to prove it actually satisfies every contract published by its consumers — by replaying each recorded interaction against the real, running provider service (not a mock of it) and confirming the response matches what the consumer expects.

---

## Setting Up Provider Verification

```javascript
const { Verifier } = require('@pact-foundation/pact');

new Verifier({
  provider: 'UserService',
  providerBaseUrl: 'http://localhost:8080',  // the REAL service, running locally in CI
  pactBrokerUrl: 'https://your-org.pactflow.io',
  pactBrokerToken: process.env.PACT_BROKER_TOKEN,
  publishVerificationResult: true,
  providerVersion: process.env.GIT_COMMIT_SHA,
  stateHandlers: {
    'a user with ID 123 exists': async () => {
      await db.seed('users', { id: 123, status: 'active' });
    },
    'no user with ID 999 exists': async () => {
      await db.ensureAbsent('users', { id: 999 });
    },
    'the user service is experiencing a downstream database timeout': async () => {
      await mockDownstreamTimeout();
    },
  },
}).verifyProvider();
```

The provider must actually be running (as a real HTTP service, even if in-process for the test) for verification to mean anything — the whole point is testing real behavior, not a stand-in for it.

---

## Provider States: Setting Up the World the Consumer Assumed

Each `given()` clause on the consumer side (e.g. `"a user with ID 123 exists"`) needs a matching state handler on the provider side that actually puts the system into that state before the interaction is replayed. This is usually the most involved part of provider verification to set up — it means the provider's test setup needs the ability to seed itself into arbitrary specific states on demand, not just start from one fixed baseline.

| Approach | How |
|---|---|
| Direct DB seeding | State handler inserts/deletes rows directly to represent the required state — simplest for straightforward CRUD-backed states |
| API-driven seeding | State handler calls the provider's own internal APIs to create the state, exercising more of the real code path | 
| Test doubles for downstream dependencies | For states representing a downstream failure (like the timeout example above), the provider's own dependencies need to be put into a failure mode — via a test double/fault injection, not by actually breaking a real downstream service |

---

## Verifying Against Multiple Consumer Versions

A provider often has multiple consumers, and each consumer may have multiple deployed versions in flight (production still on v3, staging testing v4). Provider verification should check against the contracts each consumer currently has published for the environments that matter, not just "the latest contract from each consumer" — otherwise a provider change could break a consumer version still running in production while looking clean against that consumer's newer, not-yet-deployed contract.

This is what `pact-broker-and-cicd.md`'s versioning/tagging conventions and the `can-i-deploy` check are specifically designed to handle — see that reference for the mechanics.

---

## When Verification Fails

A failed verification means the provider's actual behavior no longer matches what a consumer depends on. Two possible resolutions, and it matters which one applies:

1. **The provider introduced a genuine breaking change** — fix the provider to restore compatibility, or coordinate with the consumer team on a deliberate, versioned breaking change (new API version, migration plan) rather than shipping silently.
2. **The consumer's contract was over-specified** (asserted something it doesn't actually need, per the over-specification anti-patterns in `consumer-side-testing.md`) — in this case, the fix is in the consumer's test, not the provider's code. Don't reflexively "fix" the provider to satisfy an overly strict contract that doesn't reflect a real consumer requirement.

Getting this distinction right requires provider and consumer teams to actually talk to each other when verification fails — contract testing surfaces the disagreement clearly and fast, but resolving it still needs a real conversation about what changed and whether it should have.

---

## Checklist

```
- [ ] Verification runs against the real, running provider service, not a re-mocked stand-in
- [ ] Every provider state referenced by a consumer's `given()` clause has a corresponding, working state handler
- [ ] Downstream-failure states are simulated via test doubles/fault injection, not by breaking a real dependency
- [ ] Verification checks contracts from the consumer versions actually deployed/in-flight, not just each consumer's latest
- [ ] Failed verifications are triaged as either "provider broke a real contract" or "consumer over-specified" before deciding which side to fix
```
