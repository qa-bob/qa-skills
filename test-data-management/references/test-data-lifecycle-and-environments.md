# Test Data Lifecycle & Environments

Test data isn't a one-time artifact — it's provisioned, used, sometimes refreshed, and eventually torn down. Environments that skip explicit lifecycle management tend to accumulate stale, inconsistent, or (worse) sensitive leftover data over time.

---

## The Lifecycle

```
1. Provision   — create the environment's initial data state (seed data, see
                  referential-integrity-and-fixtures.md)
2. Use         — tests run against it, potentially mutating state
3. Reset/Refresh — return to a known-good state, either per-test or on a schedule
4. Teardown    — fully remove the environment and its data when no longer needed
```

Skipping step 3 or 4 is where most of the pain accumulates: a test environment that's never reset drifts further from a known state with every test run, until failures become impossible to diagnose because nobody can say what state the data was actually in when a given test ran.

---

## Isolation Strategies

| Strategy | How it works | Tradeoff |
|---|---|---|
| Fresh ephemeral environment per test run | Spin up a new database/container per CI run, seed it, tear it down after | Slowest to set up per run, but eliminates any cross-run state leakage entirely |
| Transactional rollback per test | Wrap each test in a DB transaction, roll back at the end | Fast, but doesn't work for anything that isn't transactional (external API calls, file writes, some NoSQL stores) |
| Namespaced/prefixed data per test run | Generate data with a unique run-ID prefix, filter/query scoped to that prefix, clean up by prefix afterward | Works for shared environments where spinning up a fresh instance per run isn't practical, but requires discipline — every test must respect the namespace boundary |
| Dedicated per-developer/per-branch environment | Each developer or feature branch gets its own persistent environment | Good for exploratory/manual testing; still needs its own periodic refresh cycle to avoid drifting into an unrepresentative state |

For CI-driven automated test suites, ephemeral-per-run (via Docker/Testcontainers or a cloud ephemeral-database service) is usually worth the setup cost — it removes an entire category of "flaky because of leftover state from a previous run" failures.

---

## Ephemeral Test Infrastructure Patterns

```yaml
# docker-compose.test.yml — ephemeral, torn down after the run, never a persistent resource
services:
  test-db:
    image: postgres:16
    environment:
      POSTGRES_DB: test
      POSTGRES_PASSWORD: test  # test-only credential, never reused for a real environment
    tmpfs:
      - /var/lib/postgresql/data  # in-memory, disappears entirely on container stop
```

```python
# Testcontainers — provision, seed, use, and tear down within the test process itself
from testcontainers.postgres import PostgresContainer

with PostgresContainer("postgres:16") as postgres:
    conn = get_connection(postgres.get_connection_url())
    seed_test_data(conn)  # from referential-integrity-and-fixtures.md
    run_tests(conn)
    # container auto-removed on context exit — nothing left behind
```

---

## Refresh Cadence for Longer-Lived Environments

Environments that persist across many test runs (shared staging, per-branch environments) need a defined refresh cadence rather than running indefinitely on whatever state accumulated:

- **Scheduled full refresh** (e.g. nightly rebuild from seed data) resets known-drift back to a clean baseline — good default for shared staging environments.
- **On-demand refresh** triggered by a specific event (a new release candidate, a reported "staging is broken" issue) for environments where a fixed schedule doesn't fit the usage pattern.
- **Never let "someone might be using it" become a permanent excuse to skip refresh** — define an actual maximum staleness window (e.g. "staging is rebuilt at least weekly regardless") so environments don't silently drift for months.

---

## Teardown Discipline

Teardown matters for two distinct reasons, not just "cleaning up":

1. **Cost** — ephemeral cloud resources (databases, compute, storage) left running because a teardown step was skipped or failed silently accumulate real cost over time.
2. **Data exposure window** — any test/staging environment that contains even anonymized-but-still-sensitive data has a smaller exposure risk the shorter it exists. An environment torn down promptly after use has a much smaller window during which a misconfiguration could expose it than one left running indefinitely "just in case."

```
- [ ] Every provisioned test environment has a corresponding, verified teardown step
      (not just a script that exists — confirm it actually runs and succeeds)
- [ ] Teardown failures are alerted on, not silently ignored (a failed teardown that
      nobody notices is exactly how orphaned resources and stale environments accumulate)
- [ ] CSP-provisioned ephemeral resources are tagged clearly (e.g. "ephemeral-test") so
      automated cost/cleanup tooling can identify and reap anything that outlived its
      intended lifecycle
```

---

## Checklist

```
- [ ] Isolation strategy (ephemeral-per-run, transactional rollback, namespacing, or dedicated environment) is a deliberate choice, not an accident of whatever was easiest to set up first
- [ ] CI-driven automated suites use ephemeral, torn-down-after-run infrastructure by default
- [ ] Longer-lived shared environments have a defined, enforced refresh cadence
- [ ] Teardown is verified to actually succeed, with alerting on failure — not just assumed to work because a script exists
- [ ] Ephemeral cloud resources are tagged for automated cost/cleanup tracking
```
