# Test Plan Authoring

The test plan is written before testing starts — it defines scope, approach, and the criteria that determine when testing is done. A lightweight adaptation of IEEE 829's structure, scaled to what the effort actually needs rather than filled out exhaustively by default.

---

## Core Sections

```markdown
# Test Plan — [Feature/Release/System Name]

## 1. Scope
What's being tested, and — equally important — what's explicitly NOT being tested
in this cycle and why (e.g. "third-party payment provider's internal logic is out
of scope; we test our integration with it, not their system").

## 2. Approach
Which test types apply and why (unit/integration/API/UI/performance/security/etc.),
and the risk-based reasoning behind prioritization — reference the risk assessment
that drove this (see test-thinker-style risk-based prioritization if that's part of
the broader practice) rather than restating scope decisions with no rationale.

## 3. Entry Criteria
What must be true before testing can start — e.g. "feature branch merged to a
testable environment," "test data seeded," "API contract finalized." Testing
starting before entry criteria are met is a common source of wasted effort
re-testing the same broken setup repeatedly.

## 4. Exit Criteria
What must be true for testing to be considered DONE — this is the most
consequential section, because it's what prevents "testing" from becoming an
open-ended activity with no defined completion. See the Exit Criteria section below.

## 5. Test Environment
Where testing happens, and any environment-specific caveats (e.g. "staging uses
synthetic payment provider sandbox, not production-equivalent behavior for X").

## 6. Roles & Responsibilities
Who does what — especially important when testing spans multiple people/teams
(e.g. automated regression owned by QE, exploratory testing owned by product,
security testing owned by a separate security team).

## 7. Risks & Contingencies
Known risks to the test effort itself (not risks in the product) — e.g. "test
environment historically unstable," "key SME unavailable during test window" —
and the mitigation/contingency plan for each.
```

---

## Writing Effective Exit Criteria

Vague exit criteria ("testing is complete when the team feels confident") produce open-ended testing efforts with no clear end point and no defensible basis for a ship decision. Effective exit criteria are specific and measurable:

```markdown
## Exit Criteria

- [ ] All P1/P2 test cases executed with a Pass result (P3/P4 may carry known
      issues forward with documented risk acceptance)
- [ ] Code coverage ≥ 80% on new/changed code (per test-metrics-analyst if that
      tooling exists in the broader practice)
- [ ] Zero unresolved Critical/High severity defects; Medium/Low defects have an
      explicit disposition (fixed, deferred with owner+date, or accepted risk)
- [ ] Security scan (SAST/DAST/dependency) shows zero unresolved Critical/High
      findings — see the security-testing skill for the underlying scan methodology
- [ ] Performance benchmarks meet defined SLO thresholds under expected load
```

Each criterion should be checkable by someone who wasn't in the room — "80% coverage" is checkable against a coverage report; "the team feels good about it" isn't checkable by anyone else at all.

---

## Scoping Decisions Need Their Rationale Recorded

The "why" behind a scope decision is often more valuable long-term than the scope decision itself, because scope decisions get questioned later (by a new team member, by an auditor, by the same team six months on when memory has faded) and an undocumented "we decided this was out of scope" with no rationale looks arbitrary even when it wasn't:

```markdown
## Scope — Rationale

Third-party payment provider's fraud-detection logic: OUT OF SCOPE.
Rationale: we have no visibility into or control over their fraud model; testing
it would require live transactions against their production system, which the
provider's terms of service prohibit for testing purposes. We instead test that
our system correctly handles the range of response codes the provider's API
documents (approved / declined / flagged-for-review), verified against their
sandbox environment.
```

---

## Right-Sizing the Plan

A test plan for a two-day bug-fix cycle and a test plan for a six-month platform migration shouldn't use the same level of detail — scale the plan to the actual risk and duration of the effort:

| Effort size | Appropriate plan depth |
|---|---|
| Small (single bug fix, small feature) | A few bullet points covering scope and exit criteria — a full IEEE-829-style document is disproportionate overhead |
| Medium (typical feature release) | The full section structure above, but concisely — a page or two |
| Large (major release, migration, compliance-driven effort) | Full detail, likely with sub-plans per test type (a security test plan, a performance test plan) referenced from a master plan |

---

## Checklist

```
- [ ] Scope explicitly states what's OUT of scope, not just what's in scope
- [ ] Exit criteria are specific and checkable by someone who wasn't part of the testing itself
- [ ] Scope and risk-acceptance decisions record their rationale, not just the decision
- [ ] Plan depth is scaled to the actual size/risk of the effort, not a fixed template applied uniformly regardless of scale
```
