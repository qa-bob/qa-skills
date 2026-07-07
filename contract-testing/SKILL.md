---
name: contract-testing
description: Design and implement consumer-driven contract testing between services using Pact — consumer-side expectation testing, provider verification, Pact Broker workflows, and CI/CD integration with can-i-deploy checks. Use this skill whenever the user is testing interactions between microservices/APIs, wants to catch breaking API changes before deployment, mentions Pact, consumer-driven contracts, or asks how to test service integrations without spinning up the whole dependency chain in every test run.
---

# Contract Testing

Contract testing exists to solve a specific gap between two other kinds of testing. Full integration testing (spinning up both the consumer and the provider service together) catches real integration bugs but is slow, flaky, and expensive to run on every PR — nobody wants to boot fifteen microservices to test one endpoint. Isolated unit testing (mocking the dependency entirely) is fast but the mock is only as good as the assumption behind it — if the real provider's behavior diverges from what the mock assumes, unit tests stay green while production breaks. Contract testing sits between the two: each side is tested in isolation, fast, but against a real, versioned, continuously-verified contract instead of an unchecked assumption.

**Consumer-driven** is the key phrase: the consumer of an API defines what it actually needs from the provider (not the provider unilaterally documenting what it offers), and that captured expectation becomes an executable contract the provider must continue to satisfy. This flips the usual documentation-drift problem — instead of an API spec that silently goes stale, the contract is enforced by a test that fails the moment the provider stops meeting it.

---

## When to reach for each part of this skill

| Situation | Do this |
|---|---|
| Writing the consumer side — defining what your service expects from a dependency | Read `references/consumer-side-testing.md` |
| Writing the provider side — verifying your service satisfies contracts consumers depend on | Read `references/provider-verification.md` |
| Setting up a Pact Broker, versioning contracts, or wiring `can-i-deploy` into CI/CD | Read `references/pact-broker-and-cicd.md` |
| Testing event-driven / message-based (not request-response) interactions | Read `references/message-and-async-contracts.md` |
| None of the above — just getting oriented | Keep reading this file |

---

## How Contract Testing Actually Works

```
1. Consumer test defines expected interactions:
   "when I send a GET to /users/123, I expect a 200 with a body matching this shape"
2. Running the consumer test generates a PACT FILE — a machine-readable record
   of every interaction the consumer expects, without ever calling the real provider
3. The pact file is published to a Pact Broker (a shared contract repository)
4. The PROVIDER's CI pipeline pulls the relevant pact file(s) and replays each
   interaction against the real provider service
5. If the provider's actual response doesn't match what the consumer expects,
   provider verification FAILS — this is the mechanism that catches a breaking
   change before it reaches production, on the provider's own CI run, often before
   the consumer team even notices anything changed
```

Both sides run independently, in their own CI pipelines, without ever needing the other service running live — that's what makes this fast and non-flaky compared to full integration testing, while still being a *real* test against real behavior rather than an assumption baked into a hand-written mock.

---

## Core Principle: Test Interactions, Not Implementation

A contract test should specify the shape and content of an interaction that actually matters to the consumer — not every field the provider happens to return. Over-specifying a contract (asserting on every field, exact field order, or incidental values that don't affect the consumer's actual logic) creates false-positive contract failures whenever the provider adds a new field or reorders a response — changes that are genuinely non-breaking for this consumer, but the over-strict contract fails anyway.

```javascript
// TOO STRICT — will break if the provider adds any new field, even though
// this consumer doesn't care about anything beyond user_id and status
{
  "user_id": 123,
  "name": "Jane Smith",
  "email": "jane@example.com",
  "status": "active",
  "created_at": "2026-01-15T10:00:00Z"
}

// APPROPRIATELY SCOPED — matches type/shape for what this consumer actually
// uses, ignores fields it doesn't consume
{
  "user_id": like(123),        // matcher: any integer, not exactly 123
  "status": term("active|inactive|suspended", "active"),  // matcher: one of these values
}
```

Using matchers (type/regex/enum matchers rather than exact-value matchers) for anything except values the consumer's logic genuinely branches on is the single biggest factor in whether a contract test suite stays useful over time or becomes a source of noisy, non-actionable failures that erode trust in the whole practice — the same trust-erosion risk called out for over-strict SAST rules in `security-testing` and for non-independent LLM judges in `llm-evaluation`.

---

## Decision Tree

```
Which side of the interaction am I testing?
├── I'm the consumer, defining what I expect from a dependency
│   → references/consumer-side-testing.md
├── I'm the provider, verifying I satisfy contracts consumers depend on
│   → references/provider-verification.md
├── I need to know whether it's SAFE to deploy given current contract state
│   → references/pact-broker-and-cicd.md (can-i-deploy)
└── The interaction isn't request-response (it's a queue message, event, webhook)
    → references/message-and-async-contracts.md
```

---

## Checklist

```
- [ ] Contracts specify interactions the consumer actually depends on — not every field the provider happens to return
- [ ] Matchers (type/regex/enum) used instead of exact-value assertions for anything the consumer doesn't branch its logic on
- [ ] Provider verification runs in the provider's own CI, against the real provider service, not a re-mocked version of it
- [ ] Contract state is checked (can-i-deploy) before deployment, not just before merge
- [ ] Contracts are versioned/tagged per environment (not a single global "latest"), so a consumer's staging contract doesn't block a provider's production deploy incorrectly
```
