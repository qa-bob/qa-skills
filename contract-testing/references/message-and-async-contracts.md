# Message & Async Contracts

Everything covered elsewhere in this skill assumes a request-response interaction (HTTP request in, HTTP response out). Event-driven and message-based systems (queues, pub/sub, webhooks) need the same consumer-driven discipline, but the mechanics differ because there's no synchronous request to model — instead, the contract is about the shape and meaning of a message.

---

## Why Request-Response Tooling Doesn't Directly Apply

In a request-response contract, the consumer defines "when I send X, I expect Y back," and verification replays X against a live provider to check Y. In a message-based system:

- There's no "request" the consumer sends to trigger the message — the provider publishes a message whenever its own internal logic decides to, independent of any specific consumer action
- The consumer's contract is really about **the message itself**: "when the provider publishes an `OrderCreated` event, I expect it to have this shape"
- Verification means checking that the provider is *capable of producing* a message matching that shape — not replaying a live request/response cycle

Pact supports this via **message pacts**, which model this differently from HTTP interactions.

---

## Consumer-Side Message Pact

```javascript
const { MessageConsumerPact, MatchersV3 } = require('@pact-foundation/pact');
const { like, term } = MatchersV3;

const messagePact = new MessageConsumerPact({
  consumer: 'NotificationService',
  provider: 'OrderService',
});

describe('OrderCreated event contract', () => {
  it('accepts a valid OrderCreated event', () => {
    return messagePact
      .expectsToReceive('an OrderCreated event')
      .withContent({
        event_type: term('OrderCreated', 'OrderCreated'),
        order_id: like('ord_12345'),
        customer_id: like('cust_67890'),
        total: like(49.99),
      })
      .verify(async (message) => {
        // Run the consumer's actual message handler against the modeled message,
        // proving the consumer's code correctly processes this shape
        const result = await handleOrderCreatedEvent(message.contents);
        expect(result.processed).toBe(true);
      });
  });
});
```

This still generates a pact file, structured for a message interaction rather than an HTTP one, and still gets published to the broker the same way.

---

## Provider-Side Message Pact Verification

Provider verification for messages works differently from HTTP verification: instead of replaying a request against a running service and checking the HTTP response, the provider verification step invokes the actual code path that *produces* the message and checks that output against the contract.

```javascript
const { Verifier } = require('@pact-foundation/pact');

new Verifier({
  provider: 'OrderService',
  pactBrokerUrl: 'https://your-org.pactflow.io',
  messageProviders: {
    'an OrderCreated event': async () => {
      // Call the actual provider logic that constructs this event,
      // returning the message content it would actually publish
      return buildOrderCreatedEvent({ order_id: 'ord_12345', customer_id: 'cust_67890', total: 49.99 });
    },
  },
}).verifyProvider();
```

This proves the provider's message-construction logic produces something matching what the consumer expects — it does not prove the message actually gets delivered through the real queue/broker infrastructure (Kafka, SQS, SNS, RabbitMQ, Service Bus), which is a separate concern.

---

## What This Doesn't Cover: Delivery Infrastructure

Message contract testing verifies the *shape* of the message, not the delivery mechanism. Test the queue/broker infrastructure itself separately — ordering guarantees, at-least-once vs. exactly-once delivery semantics, dead-letter-queue handling, and partition/topic routing are infrastructure-level concerns that need their own test coverage (often via the messaging platform's own testing tools, or a lightweight integration test against a real or containerized broker instance) independent of the contract test suite.

---

## Schema Evolution for Long-Lived Message Contracts

Message-based systems often have longer-lived contracts than request-response APIs (a consumer might process events published months ago that are still in a queue, or replay historical events). This makes backward-compatible schema evolution particularly important:

| Safe change | Unsafe change |
|---|---|
| Adding a new optional field | Removing or renaming an existing field a consumer depends on |
| Adding a new possible value to an enum, if consumers are required to handle an "unknown value" case gracefully | Changing the meaning of an existing field without a version bump |
| Widening a type (e.g. int to a wider numeric type) | Narrowing a type or changing a field's fundamental type |

Require consumers to explicitly handle "unrecognized enum value" and "unexpected extra field" gracefully in their message-handling code — this one consumer-side discipline is what makes many otherwise-breaking provider changes safe in practice, and it's worth testing for directly (send a message with an extra unexpected field or an unrecognized enum value, verify the consumer doesn't crash).

---

## Checklist

```
- [ ] Message contracts model the message shape a consumer depends on, not a request-response interaction shoehorned into the wrong tool
- [ ] Provider verification invokes the real message-construction code path, not a hand-written stand-in for it
- [ ] Delivery infrastructure (ordering, delivery guarantees, DLQ handling) is tested separately from message shape contracts
- [ ] Consumers are tested for graceful handling of unrecognized fields/enum values, since this is what makes additive provider changes safe without a contract bump
```
