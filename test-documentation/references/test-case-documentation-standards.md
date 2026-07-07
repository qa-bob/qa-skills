# Test Case Documentation Standards

The granular record produced during testing — detailed enough to be traceable back to a requirement and re-executable by someone other than the original author, without being so heavyweight that maintaining it becomes its own burden.

---

## Standard Test Case Format

```markdown
## Test Case: TC-0142
**Title:** Checkout rejects expired discount code
**Requirement traced to:** REQ-089 (Discount codes must respect expiration date)
**Preconditions:**
  - A discount code "SUMMER20" exists in the system with expiration date in the past
  - A test customer account with at least one item in cart
**Steps:**
  1. Log in as test customer
  2. Add item to cart, proceed to checkout
  3. Enter discount code "SUMMER20" in the promo code field
  4. Click "Apply"
**Expected Result:**
  System displays "This code has expired" and does not apply any discount
**Actual Result:** [filled in during execution]
**Status:** [Pass / Fail / Blocked / Not Run]
**Executed by:** [name/automation job ID]
**Executed date:** [timestamp]
**Evidence:** [screenshot, log excerpt, or automation run link]
```

Every field here earns its place by answering a specific question a later reader will have: "what was actually tested" (steps), "what should have happened" (expected), "did it" (actual/status), "who verified this and when" (executed by/date), "can I see proof" (evidence). Drop fields that don't serve a real downstream reader for a given context, but don't drop status/evidence — those two are what make the record actually verifiable rather than just an assertion that testing happened.

---

## Traceability: Requirement → Test Case → Result

The single most valuable structural property of test case documentation, especially for audit/compliance contexts, is a working traceability chain: every requirement maps to at least one test case, and every test case's current result is visible.

```markdown
## Traceability Matrix (excerpt)

| Requirement | Test Case(s) | Status |
|---|---|---|
| REQ-089: Discount codes respect expiration | TC-0142, TC-0143 | Pass, Pass |
| REQ-090: Discount codes are single-use per customer | TC-0144 | Fail (see DEF-221) |
| REQ-091: Multiple discount codes cannot stack | TC-0145 | Not Run |
```

This matrix answers the question an auditor (or a product owner double-checking coverage before a release) actually asks: "is every requirement actually covered, and what's the current status of that coverage" — a list of test cases with no link back to requirements can't answer that question, no matter how well each individual test case is documented.

**Building this without excessive manual maintenance:** where automated test code exists, tag test functions with the requirement ID they trace to (e.g. a `@requirement("REQ-089")` decorator or an equivalent convention) and generate the traceability matrix from that tagging via a script, rather than maintaining it as a hand-updated spreadsheet that inevitably drifts out of sync with the actual test suite.

---

## Automated vs. Manual Test Case Documentation

For automated tests, the "test case" and the "test code" risk becoming two separate, divergent artifacts if documentation is maintained by hand alongside the code. Prefer generating the human-readable documentation from the automated test itself where possible:

```python
def test_checkout_rejects_expired_discount_code():
    """
    TC-0142: Checkout rejects expired discount code
    Traces to: REQ-089
    """
    # ... test implementation ...
```

A docstring/annotation convention like this, combined with a script that extracts these into a readable report, keeps the "documentation" synchronized with what's actually being executed by construction — a hand-maintained parallel spreadsheet of automated test cases is a common source of stale, inaccurate traceability claims.

For genuinely manual test cases (exploratory testing, cases that can't be automated), the full format above is worth maintaining directly, since there's no automated source of truth to generate it from.

---

## Checklist

```
- [ ] Every test case includes status and evidence, not just steps and expected result — status/evidence are what make it verifiable, not just asserted
- [ ] A traceability matrix links every requirement to at least one test case and shows current status
- [ ] For automated tests, documentation is generated from tagged test code where possible, not maintained as a separate hand-updated artifact that can drift out of sync
- [ ] Genuinely manual test cases (that can't be automated) get the full structured format directly, since no automated source of truth exists to generate it from
```
