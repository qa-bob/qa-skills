---
name: test-data-management
description: Generate, anonymize, and manage test data for automated testing and QA environments — synthetic data generation, PII masking/anonymization of production-derived data, referential integrity across related fixtures, and test data lifecycle (provisioning, refresh, teardown). Use this skill whenever the user needs test fixtures, sample/seed data, a way to safely use production-like data in a test or staging environment, mock records for development, or mentions PII masking, data anonymization, synthetic data, or "can we use real customer data for testing" — even if they don't ask for it as a formal "test data strategy."
---

# Test Data Management

Good test data has to satisfy two goals that are often in tension: it needs to be **realistic** enough to catch real bugs (correct distributions, correct edge cases, correct relationships between fields), and it needs to be **safe** enough that using it never creates a privacy or compliance incident. Most test data problems come from picking one goal at the expense of the other — either painfully unrealistic hand-typed fixtures that miss real bugs, or a raw copy of production data that's realistic but a liability sitting in a test environment nobody scrutinizes as carefully as production.

This skill covers how to get both: synthetic data generation that's realistic without ever touching real PII, safe techniques for the cases where production-derived data genuinely is needed, and the lifecycle discipline (referential integrity, determinism, provisioning/teardown) that keeps test data useful over time instead of turning into unmaintainable fixture sprawl.

---

## When to reach for each part of this skill

| Situation | Do this |
|---|---|
| Generating fixtures/mock records from scratch (no real data involved) | Read `references/synthetic-data-generation.md` |
| Deciding whether/how production data can safely inform a test dataset | Read `references/pii-anonymization-and-masking.md` |
| Test data spans multiple related tables/entities (orders + customers + products, etc.) | Read `references/referential-integrity-and-fixtures.md` |
| Provisioning, refreshing, or tearing down test environments and their data | Read `references/test-data-lifecycle-and-environments.md` |
| None of the above — just getting oriented | Keep reading this file |

---

## The Default Rule: Synthetic First

Start every test data need from the assumption that synthetic (fully generated, no real records involved) is the answer, and only reach for anything production-derived when synthetic genuinely can't produce the realism needed (e.g. reproducing a bug that only manifests against a specific, unusual real-world data shape). This ordering matters because it's much easier to accidentally under-protect real data than to accidentally under-realism synthetic data — the failure mode of "our synthetic data was slightly unrealistic" is a missed bug; the failure mode of "we had real customer PII sitting in a staging environment with weaker access controls than prod" is an incident.

```
Can this test need be met with fully synthetic data?
├── Yes (the large majority of cases) → references/synthetic-data-generation.md
└── No — need real-world data shape/distribution/edge cases that are impractical to
    synthesize (e.g. reproducing a production bug tied to specific real data patterns)
    → references/pii-anonymization-and-masking.md, and treat this as the exception
      that needs its own sign-off, not the default workflow
```

---

## Core Principle: Determinism Is What Makes Test Data Debuggable

Test data that's randomly different on every run makes failures hard to reproduce — "it failed once, I can't get it to fail again" is a direct consequence of non-deterministic fixtures. Every synthetic data generator should accept an explicit seed, and that seed should be logged alongside any test failure:

```python
# Deterministic — same seed always produces the same dataset
fake = Faker()
fake.seed_instance(42)
customer = fake.simple_profile()  # identical output every time seed=42 is used

# Non-deterministic — different data every run, harder to reproduce a failure against
fake = Faker()
customer = fake.simple_profile()  # different every run
```

This doesn't mean *all* test data must always be identical across every run forever — property-based/fuzz testing deliberately varies input to find new edge cases, and that's a legitimate, different technique. The rule is narrower: whatever data produced a given test result, the seed/generation parameters that produced it must be captured, so that specific result can be reproduced on demand.

---

## Core Principle: Real PII Has No Safe Default Use in Test Environments

The single most common test-data mistake with real compliance consequences: copying a production database (or a "sanitized" export that turns out to still contain identifiable data) into a staging/test/dev environment because it's the fastest way to get realistic data. Test and staging environments routinely have weaker access controls, weaker monitoring, and broader engineer access than production — which means real PII sitting there is often *less* protected than the same data in the system it was copied from, not equally protected.

If production-derived data is genuinely necessary, it must go through actual anonymization (see `references/pii-anonymization-and-masking.md`) with a defined, defensible process — not an ad hoc "let's just drop the email column" pass that leaves indirect identifiers (a rare combination of zip code + birth date + gender can uniquely identify someone even with no name or email present at all).

---

## Decision Tree

```
What kind of test data is needed?
├── Fresh fixtures for unit/integration tests, no existing data to draw from
│   → references/synthetic-data-generation.md
├── Data needs to reflect real distributions/relationships that are hard to fabricate,
│   and there's a legitimate need for production-derived realism
│   → references/pii-anonymization-and-masking.md (treat as the exception path)
├── The dataset spans multiple related entities and referential integrity matters
│   → references/referential-integrity-and-fixtures.md
└── The question is really about environment/lifecycle ("how do we keep staging data
    fresh," "how do ephemeral test databases get seeded and torn down")
    → references/test-data-lifecycle-and-environments.md
```

---

## Checklist

```
- [ ] Synthetic data is the default; any production-derived data has an explicit, documented reason it's needed
- [ ] Every data generator accepts and logs an explicit seed, so any test result is reproducible on demand
- [ ] No real PII exists in test/staging environments without going through a defined anonymization process — "sanitized" is not the same as "verified anonymized"
- [ ] Indirect/quasi-identifiers (zip + birthdate + gender, etc.) are considered, not just obvious direct identifiers (name, email, SSN)
- [ ] Multi-entity test data maintains referential integrity — no orphaned foreign keys or impossible relationship states
- [ ] Test data has a defined lifecycle — provisioning and teardown are both accounted for, not just creation
```
