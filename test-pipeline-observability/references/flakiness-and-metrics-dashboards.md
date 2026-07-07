# Flakiness & Metrics Dashboards

Turning individual traced runs (from `otel-test-pipeline-instrumentation.md`) into trend metrics that answer "is this getting better or worse over time" — the question a single run's trace can't answer on its own.

---

## Defining Flakiness Precisely

A test is flaky if it produces different results (pass/fail) across multiple runs against the *same* code — the defining characteristic is that the code didn't change, but the outcome did. This needs to be measured, not guessed at from a gut feeling of "that test seems flaky":

```
flake_rate(test) = (# runs where this test's result differs from the majority
                     result for this exact test, at this exact commit)
                   / (total runs of this test at this commit)
```

Computing this requires either deliberately running a suspect test multiple times against the same commit (a dedicated flake-detection job, run periodically or on-demand) or accumulating enough natural retry data over time (if tests are automatically retried on failure, a test that fails then passes on retry, with no code change in between, is direct flakiness evidence).

---

## Per-Test Flakiness Tracking

```sql
-- Example query against a trace/metrics store — flake rate per test over the last 30 days
SELECT
  test_name,
  COUNT(*) AS total_runs,
  SUM(CASE WHEN outcome = 'failed' THEN 1 ELSE 0 END) AS failures,
  SUM(CASE WHEN retry_count > 0 AND outcome = 'passed' THEN 1 ELSE 0 END) AS flaky_passes,
  ROUND(100.0 * SUM(CASE WHEN retry_count > 0 AND outcome = 'passed' THEN 1 ELSE 0 END) / COUNT(*), 2) AS flake_rate_pct
FROM test_runs
WHERE run_timestamp > NOW() - INTERVAL '30 days'
GROUP BY test_name
HAVING flake_rate_pct > 0
ORDER BY flake_rate_pct DESC;
```

Surface this as a ranked list, not just an aggregate suite-wide flake percentage — a suite-wide "2% flaky" number hides whether that's spread evenly (lower individual concern) or concentrated in 3 specific tests (worth fixing or quarantining those 3 directly). The ranked, per-test view is what's actually actionable.

---

## Quarantine as a Deliberate, Tracked Process

Once a test is confirmed flaky, quarantining it (excluding it from the blocking test run while keeping it visible and tracked) is often the right immediate move — but only as a tracked, time-boxed process, not a silent, permanent removal:

```yaml
# Quarantined tests still run and report, but don't block the pipeline
- name: Run quarantined tests (non-blocking)
  run: pytest tests/quarantine/ --continue-on-collection-errors || true
  continue-on-error: true
```

```
quarantine_record:
  test: test_checkout_flow_with_discount
  quarantined_date: 2026-06-01
  reason: "intermittent timeout, root cause not yet identified"
  owner: <team/person responsible for fixing it>
  review_date: 2026-06-15  # explicit re-check date, not indefinite
```

A quarantine list with no owner and no review date silently becomes a permanent graveyard of untrusted tests — track both, and treat a quarantined test that's still quarantined past its review date as its own alertable condition.

---

## Dashboard Design

A useful test-pipeline health dashboard needs a small number of trend lines, each answering one of the questions from `SKILL.md`, rather than one dense screen trying to show everything:

| Panel | Metric | Why it's on the dashboard |
|---|---|---|
| Pipeline duration trend | p50/p95 duration over time, by stage | Catches gradual slowdown before it's an acute problem |
| Suite-wide flake rate trend | % of runs with at least one flaky-pass, over time | Catches overall reliability degradation |
| Top flaky tests (ranked) | Per-test flake rate, last 30 days | Points directly at what to fix next |
| Quarantine list status | Count quarantined, count past review date | Prevents the quarantine list from becoming a silent graveyard |
| Infra failure rate | % of runs failing due to infrastructure (not test/code), over time | Distinguishes "our tests are unreliable" from "our CI runners/network are unreliable" — different owners, different fixes |

Grafana (backed by the OTel-exported trace/metrics store) is a common implementation choice, but the panel design above is the actual transferable knowledge — the same five panels are worth building regardless of which dashboarding tool is in use.

---

## Checklist

```
- [ ] Flakiness is measured directly (same commit, different outcome), not inferred from a vague impression
- [ ] Flaky-test tracking is per-test and ranked, not collapsed into one suite-wide percentage
- [ ] Quarantine is a tracked process with an owner and a review date, not a silent, indefinite removal
- [ ] The dashboard separates duration, flake rate, and infra-failure rate as distinct trend lines, not one blended health score
- [ ] Quarantined tests still run and report (non-blocking) so their status is visible, rather than being fully removed from the suite
```
