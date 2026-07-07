---
name: test-pipeline-observability
description: Instrument the test pipeline itself — not the application under test — with tracing, flaky-test tracking, and health/cost alerting. Use this skill whenever the user wants to understand why CI is slow, track flaky tests over time, set up dashboards for test suite health, get alerted when the pipeline degrades, or asks "why do our tests keep getting slower/flakier" — even if they frame it as a CI or DevOps problem rather than a testing one.
---

# Test Pipeline Observability

Teams instrument their production applications by default now — logs, metrics, traces are assumed. The test pipeline that gates every deploy to that production application usually gets none of that. The result is a common, avoidable failure mode: nobody notices the test suite has gotten 3x slower over six months, or that a specific test has been flaky for weeks, until it becomes an acute problem (a release blocked, a developer losing an hour to a flaky retry loop) rather than something caught early as a visible trend.

This skill treats the test pipeline as a system worth observing in its own right — distinct from application observability (which tests whether the *product* behaves correctly) and distinct from the test metrics covered elsewhere (coverage, defect density) in that this is specifically about the health and performance of the *pipeline process* itself: how long it takes, how often it flakes, and what it costs.

---

## When to reach for each part of this skill

| Situation | Do this |
|---|---|
| Adding tracing to the test pipeline so failures/slowness are diagnosable | Read `references/otel-test-pipeline-instrumentation.md` |
| Tracking flaky tests over time and building a health dashboard | Read `references/flakiness-and-metrics-dashboards.md` |
| Deciding what to alert on, and controlling CI cost/resource usage | Read `references/pipeline-health-alerting-and-cost.md` |
| None of the above — just getting oriented | Keep reading this file |

---

## Three Questions This Skill Answers

```
1. "Why did this specific test run fail/take so long?"
   → Needs per-run TRACING — a structured record of what happened during
     this specific execution, so a failure or slowdown can be diagnosed
     without re-running and hoping to reproduce it
2. "Is our test suite getting slower/flakier over time, and where specifically?"
   → Needs TREND METRICS aggregated across many runs — a single run's trace
     doesn't answer a trend question; that needs a time series
3. "Should someone be paged/notified right now?"
   → Needs ALERTING with thresholds calibrated to avoid both silence
     (real degradation goes unnoticed) and noise (alert fatigue causes the
     alert channel itself to get ignored)
```

Each of the three reference files answers one of these — they build on each other (trend metrics are computed from many individual traced runs; alerts fire based on trend metrics crossing a threshold), but they're distinct concerns worth separating rather than solving all at once with one dashboard.

---

## Core Principle: Test Pipeline Failures Are Not All The Same Failure

A red CI run can mean genuinely different things, and treating them identically (just "rerun until green") hides real signal:

| What actually happened | What it means | How observability should surface it differently |
|---|---|---|
| A real bug in the code under test | The test is doing its job correctly | Should show up clearly as a legitimate blocker, not get lost in noise |
| A flaky test (same code, inconsistent result) | The test itself is unreliable | Needs to be tracked as its OWN category with its own rate, separate from real failures — see `flakiness-and-metrics-dashboards.md` |
| Test infrastructure failure (a runner died, a network blip, a rate-limited dependency) | Neither the code nor the test is at fault | Should be visible as an infrastructure-health signal, distinct from both of the above |
| The test suite is just slow (passed, but took much longer than usual) | No functional problem, but a trend worth watching | Needs duration tracked as a first-class metric, not just pass/fail |

A test pipeline dashboard that only shows "pass rate" collapses all four of these into one number, which is close to useless for actually deciding what to fix. Observability here means being able to distinguish these categories, not just knowing something was red.

---

## Checklist

```
- [ ] Failures are categorized (real bug / flaky / infra / slow-but-passed), not collapsed into one pass/fail signal
- [ ] Individual test runs are traced with enough structure to diagnose a specific failure without needing to reproduce it live
- [ ] Trend metrics (duration, flake rate, failure rate) are tracked over time, not just visible per-run
- [ ] Alert thresholds are calibrated against real historical variance, not picked arbitrarily
- [ ] Test pipeline cost is tracked as a monitored trend, not discovered only when a bill is surprising
```
