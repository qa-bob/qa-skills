# Behavioral & Consistency Testing

Tests whether the system does the right thing, and keeps doing it — across repeated runs of the same input, across a multi-turn conversation, and across the range of phrasings a real user would actually type.

---

## Behavioral Correctness: Testing Properties, Not Strings

Since outputs are non-deterministic, assertions need to target the underlying property that must hold rather than the literal text produced.

| Instead of asserting... | Assert this |
|---|---|
| The output equals a fixed string | The output contains the required fact/entity/number, regardless of phrasing |
| The output matches a regex for tone | An LLM-judge or a classifier scores the tone within an acceptable band |
| The refund policy text is word-for-word identical to the source doc | The stated refund window (e.g. "30 days") is present and correct, however it's phrased |
| The agent calls exactly this one tool in exactly this order | The agent reaches the correct end state, and if order matters, the *dependencies* between tool calls are respected (can't call `refund()` before `verify_order()` regardless of intermediate steps) |

### Example test case

```yaml
test_id: refund-policy-basic
input: "How many days do I have to return this?"
must_contain: ["30 days"]     # property: correct fact, any phrasing
must_not_contain: ["forever", "no limit"]  # property: doesn't invent a wrong policy
tone_check: professional      # graded separately by an LLM judge, not string-matched
```

---

## Consistency Testing: Same Input, Repeated Runs

Run the same input N times (N=5-10 is a reasonable default for a first pass; increase for anything safety-critical) and measure the **pass rate**, not just whether any single run passed.

```
consistency_score = (# runs meeting the required property) / (total runs)
```

A system that gets a critical fact right 8/10 times has a real, measurable 20% failure rate — a single passing run in manual testing would have completely hidden this. This is the most common reason LLM bugs escape to production: someone tried it once, it worked, and it shipped.

**What to do with a partial pass rate:** don't just report the number — investigate *which* runs failed and look for a pattern (a specific phrasing that triggers it, a specific fact that's unstable, a specific position in a longer conversation). Random-looking failures with no discoverable pattern are lower priority to chase than a systematic one-in-three failure tied to a specific trigger.

---

## Multi-Turn / Conversational Testing

Single-turn testing only covers the easiest case. Real usage is conversational, and specific failure modes only appear across turns:

| Failure mode | How to test it |
|---|---|
| Context loss | Establish a fact in turn 1 ("my order number is 12345"), ask a question in turn 4 that requires remembering it, without repeating it |
| Instruction drift | Give a constraint early ("always respond in bullet points"), continue the conversation for several turns, and check the constraint is still honored later |
| Compounding hallucination | Ask a question with a subtly false premise in turn 1; check whether the system corrects it or builds increasingly elaborate answers on top of the false premise in later turns |
| State/tool-call coherence (agents) | For tool-using agents, verify that state established by an earlier tool call (e.g. "add to cart") is correctly available to a later step (e.g. "checkout") without re-fetching or losing it |
| Recovery from user correction | Deliberately correct the system mid-conversation ("actually I meant next Tuesday, not this one") and verify it adopts the correction rather than reverting to the earlier assumption in a later turn |

### Long-conversation / context-window testing

For systems with long expected conversations, specifically test behavior as the conversation approaches the model's context window limit: does critical early information (system instructions, key facts established early) get correctly retained, or does it get silently dropped as the context fills ("lost in the middle")? This needs a deliberately long synthetic conversation as a test fixture — it won't show up in short manual spot-checks.

---

## Coverage Strategy

Behavioral test suites for LLM systems benefit from a deliberate mix, not just the phrasing a developer happens to think of first:

```
1. Canonical phrasing — the "textbook" way to ask each core question
2. Realistic variation — typos, informal phrasing, non-native-speaker patterns, partial
   information a real user might actually provide
3. Edge cases — boundary values, empty/minimal input, maximum-length input
4. Adjacent-but-different requests — inputs that are similar to a covered case but
   should actually get a different answer (tests that the system discriminates
   correctly, not just that it pattern-matches keywords)
```

---

## Checklist

```
- [ ] Assertions target properties (facts, constraints, end states), not exact strings
- [ ] Consistency measured via repeated runs with a reported pass rate, not a single run
- [ ] Multi-turn tests cover context retention, instruction drift, and correction handling
- [ ] Long-conversation/context-limit behavior tested explicitly if relevant to the product
- [ ] Test input set includes realistic variation and adjacent-but-different cases, not just canonical phrasing
```
