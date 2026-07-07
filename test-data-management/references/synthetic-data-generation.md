# Synthetic Data Generation

Generating realistic test data from scratch, with no real records involved — the default, preferred path for the large majority of test data needs.

---

## Persona / Record Generation

Use a data-faking library (Faker for Python/JS, Bogus for .NET, etc.) rather than hand-typing fixtures — hand-typed fixtures tend toward unrealistic uniformity (every test user is "John Doe," every address is "123 Main St"), which under-tests anything sensitive to real-world data variety (name parsing edge cases, international address formats, unicode in text fields).

```python
from faker import Faker

fake = Faker()
fake.seed_instance(42)  # deterministic — see SKILL.md's determinism principle

customer = {
    "id": fake.uuid4(),
    "name": fake.name(),
    "email": fake.email(),
    "address": fake.address(),
    "phone": fake.phone_number(),
    "signup_date": fake.date_between(start_date="-3y", end_date="today"),
}
```

### Locale and internationalization coverage

Real user bases aren't monolingual/mono-format — generate data across locales deliberately if the system needs to handle international users, rather than only ever generating US-formatted names/addresses/phone numbers:

```python
fake_us = Faker("en_US")
fake_jp = Faker("ja_JP")
fake_de = Faker("de_DE")
# Mix locales in a generated dataset to test name-length assumptions,
# non-Latin character handling, address format variance, etc.
```

---

## Boundary-Value and Edge-Case Generation

Realistic-looking data alone under-tests boundaries. Deliberately generate the edge cases a "normal" random sample will rarely produce on its own:

| Category | Edge cases to explicitly include |
|---|---|
| Strings | Empty string, single character, maximum-length string, strings with leading/trailing whitespace, unicode/emoji, strings containing characters meaningful to the storage layer (quotes, SQL metacharacters — for testing injection defenses, not exploiting them) |
| Numbers | Zero, negative (if the domain allows), maximum representable value, values just above/below a business-rule threshold (e.g. exactly at a discount tier boundary) |
| Dates | Leap-year dates (Feb 29), timezone-boundary dates (just before/after midnight UTC vs. local), far-past and far-future dates, the current date at generation time (tests "today" logic explicitly) |
| Collections | Empty collection, single-item collection, a collection at whatever size limit the system enforces, one item over that limit |
| Nullable fields | Explicitly null, explicitly empty (empty string vs. null are different states many systems handle inconsistently) |

A good synthetic dataset mixes realistic random records with a deliberately curated set of these edge cases — relying on randomness alone to eventually generate the boundary case is a slow, unreliable way to get coverage that a few explicit rows can guarantee immediately.

---

## Deterministic Seeding in Practice

```python
# Store the seed alongside the test, not just in a global config —
# each test/fixture set having its own seed makes failures traceable to
# exactly which generation produced the failing data
def make_test_customer(seed: int = 1001):
    fake = Faker()
    fake.seed_instance(seed)
    return fake.simple_profile()

# When a test fails, log the seed used:
# "test_checkout_flow failed with customer fixture seed=1001"
# → anyone can regenerate the exact same customer record to debug
```

For property-based/fuzz testing (deliberately varying input to discover new edge cases), invert this: let the seed vary across runs to explore the input space, but **capture and log the specific seed that produced any failure** so that specific failing case becomes reproducible going forward, even though the overall test run wasn't deterministic.

---

## Structured Fixture Types

Different test layers need different fixture shapes — match the format to where it's consumed:

| Fixture type | Typical use |
|---|---|
| JSON/YAML fixtures | Unit and integration test inputs, config-driven test data |
| Parametrized test tables | Table-driven tests covering many input/expected-output pairs concisely |
| SQL seed scripts | Populating a test database with a known starting state |
| Mock API response payloads | Stubbing external service calls in integration tests |
| Persona records | Representing a specific user archetype consistently across multiple test scenarios (e.g. "the power user," "the first-time user," "the user with an expired subscription") |

Persona records are worth calling out specifically: rather than generating an unrelated random customer for every test, maintaining a small library of named, consistent personas (with realistic, stable attributes) makes test intent much more readable — "test with the `expired_subscription_user` persona" communicates more than "test with `fake.simple_profile()` with `is_active=False` set inline."

---

## Checklist

```
- [ ] Data generated with a faking library, not hand-typed uniform fixtures
- [ ] Locale/internationalization variety included if the system needs to handle it
- [ ] Boundary/edge cases explicitly curated, not left to random chance to eventually appear
- [ ] Every generator call uses an explicit, logged seed
- [ ] Fixture format (JSON/YAML/SQL seed/mock payload/persona) matches what the consuming test layer actually needs
```
