# Consumer-Side Contract Testing

The consumer defines what it needs from a provider, and that definition becomes the executable contract. Getting the consumer side right is mostly about scoping interactions correctly (see the matcher guidance in `SKILL.md`) and structuring tests so the generated pact file stays a useful, stable artifact.

---

## Writing a Consumer Test (JavaScript / Pact-JS example)

```javascript
const { PactV3, MatchersV3 } = require('@pact-foundation/pact');
const { like, term, eachLike } = MatchersV3;

const provider = new PactV3({
  consumer: 'OrderService',
  provider: 'UserService',
});

describe('User Service contract', () => {
  it('returns a user profile for a valid user ID', () => {
    provider
      .given('a user with ID 123 exists')  // provider state — see provider-verification.md
      .uponReceiving('a request for user 123')
      .withRequest({
        method: 'GET',
        path: '/users/123',
      })
      .willRespondWith({
        status: 200,
        headers: { 'Content-Type': 'application/json' },
        body: {
          user_id: like(123),
          status: term('active|inactive|suspended', 'active'),
        },
      });

    return provider.executeTest(async (mockServer) => {
      const client = new UserServiceClient(mockServer.url);
      const user = await client.getUser(123);
      expect(user.status).toBe('active');
    });
  });
});
```

Running this test does two things: it verifies the consumer's own client code correctly handles the modeled response, AND it generates a pact file capturing the interaction — the same test authors both the unit-level check and the contract artifact.

---

## Modeling Multiple Scenarios

Cover the shapes of interaction the consumer actually needs to handle correctly, not just the happy path:

```javascript
// Happy path
.given('a user with ID 123 exists')
.uponReceiving('a request for user 123')
...

// Not found — consumer needs to handle this correctly, so it needs a contract too
.given('no user with ID 999 exists')
.uponReceiving('a request for user 999')
.willRespondWith({ status: 404 })

// Error state the consumer has specific handling for
.given('the user service is experiencing a downstream database timeout')
.uponReceiving('a request for user 123')
.willRespondWith({ status: 503 })
```

Each distinct provider state the consumer's code branches on should get its own interaction — if the consumer has different handling for 200/404/503, the contract should verify the provider actually produces distinguishable, correctly-shaped responses for all three, not just the success case.

---

## Avoiding the Over-Specification Trap

Beyond the matcher guidance in `SKILL.md`, watch for these specific over-specification patterns:

| Anti-pattern | Problem | Fix |
|---|---|---|
| Asserting exact array length when the consumer just needs "at least one item" | Breaks when the provider's test/real data changes size | Use `eachLike()` / a minimum-length matcher instead of an exact count |
| Asserting field order in a JSON object | JSON objects have no meaningful order; asserting it couples the contract to incidental serialization behavior | Assert on presence and value/type per field, not order |
| Copying a full real API response into the contract verbatim | Captures every incidental field, most of which the consumer doesn't use | Trim to what's actually consumed; use `like()`/type matchers for the rest |
| One contract test per endpoint covering every possible parameter combination | Contract tests aren't the place for exhaustive parameter coverage — that's what direct unit/integration tests for the client library are for | Keep the contract test set focused on distinct provider *behaviors* (success, not-found, error state), not exhaustive input coverage |

---

## What Happens to the Generated Pact File

The pact file produced by running these tests needs to reach the provider somehow — normally via publishing to a Pact Broker (see `pact-broker-and-cicd.md`), tagged with the consumer's version and the environment/branch it came from. Treat pact file publication as part of the consumer's CI pipeline, not a manual step — a contract that isn't published is invisible to the provider's verification and provides zero actual protection.

---

## Checklist

```
- [ ] Every provider state the consumer's code has distinct handling for has its own interaction defined
- [ ] Matchers used appropriately (see SKILL.md) — no over-specified exact values, array lengths, or field order
- [ ] Interactions model provider behaviors (success/not-found/error), not exhaustive input parameter combinations
- [ ] Generated pact files are published automatically as part of CI, not a manual step someone has to remember
```
