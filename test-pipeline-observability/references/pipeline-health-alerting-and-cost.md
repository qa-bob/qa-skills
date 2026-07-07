# Pipeline Health Alerting & Cost

Turning trend metrics into alerts that fire at the right time — not so often that the channel gets ignored, not so rarely that real degradation goes unnoticed for months — and tracking what the test pipeline actually costs to run.

---

## Calibrating Alert Thresholds Against Real Variance

The most common alerting mistake is picking a threshold that sounds reasonable ("alert if duration exceeds 10 minutes") without checking what the actual historical distribution looks like. If normal runs already vary between 7 and 9 minutes, a 10-minute threshold will fire on ordinary variance constantly; if normal runs are consistently under 4 minutes, a 10-minute threshold will miss a real 2x slowdown entirely.

```
1. Pull the actual historical distribution for the metric (duration, flake rate, etc.)
   over a representative recent window (e.g. last 30 days)
2. Set the threshold relative to that distribution — e.g. "alert if p95 duration
   exceeds 1.5x the trailing 30-day median," not a fixed absolute number picked
   without looking at the data
3. Re-calibrate periodically — as the suite grows or the team adds new test
   categories, the "normal" baseline shifts, and a threshold that was well-tuned
   six months ago may now be stale
```

---

## What's Actually Worth Alerting On

| Signal | Alert condition | Why this one and not others |
|---|---|---|
| Sustained duration increase | p95 duration trending up over several consecutive runs/days, past a relative threshold | A single slow run is noise; a sustained trend is a real signal worth someone's attention |
| Flake rate spike | Suite-wide or specific-test flake rate crosses a relative threshold within a short window | A sudden spike (vs. a slow drift) often points at a specific recent change worth investigating immediately |
| Infra failure rate spike | Infra-category failures (see `flakiness-and-metrics-dashboards.md`'s failure categorization) exceed a threshold | Distinct from flakiness — this points at CI infrastructure health, a different team/fix than test-code flakiness |
| Quarantine list growing unbounded | Quarantined-test count increases without a corresponding decrease (tests being fixed and un-quarantined) | Signals the team is quarantining faster than fixing — a process health problem, not just a test health problem |
| Cost anomaly | Test-pipeline cloud spend (see below) deviates significantly from its own trailing baseline | Catches a runaway resource-creation bug or an inefficient new test category before it's a large unexpected bill |

**What's usually not worth a dedicated alert:** a single failed run, a single slow run, or noise-level fluctuation within the normal historical range — alerting on any of these trains the team to ignore the alert channel, which defeats the purpose for the signals that do matter.

---

## Avoiding Alert Fatigue

- **Route different alert categories to different channels/severities** — a sustained duration trend is not the same urgency as an infra outage actively blocking every PR; conflating them into one alert stream makes the urgent ones harder to notice.
- **Include the actual data, not just "something's wrong," in the alert** — "p95 duration is 18 min, up from a 30-day median of 11 min" is actionable immediately; "test pipeline health degraded" requires someone to go look before they even know what they're looking at.
- **Review triggered alerts periodically for false-positive rate** — if a specific alert condition fires often and is dismissed as noise every time, that's a signal the threshold (or the metric itself) needs recalibration, not that the team should just get better at ignoring it.

---

## Test Pipeline Cost Tracking

Beyond cloud infrastructure cost (covered from the ephemeral-resource angle in the multi-cloud-testing skill's cost-control guidance), the test pipeline has its own cost dimensions worth tracking directly:

| Cost dimension | What drives it |
|---|---|
| CI compute minutes | Total runner time across all pipeline runs — grows with both suite size and suite duration |
| Parallel runner count | More parallelism reduces wall-clock time but increases concurrent compute cost — there's a real tradeoff here, not a free win |
| Third-party test tool/API costs | LLM-judge API calls (see `llm-evaluation`), device farm minutes (mobile testing), commercial scanning tool licenses — these scale with test volume and are easy to lose track of as usage grows |
| Ephemeral cloud test infrastructure | Covered in `multi-cloud-testing`'s cost-control reference |

Track total pipeline cost as its own trend line on the same dashboard as duration and flake rate (see `flakiness-and-metrics-dashboards.md`) — cost tends to creep upward gradually as a suite grows, and it's much easier to catch and address that creep early than to discover a large, surprising number after months of unattributed growth.

---

## Checklist

```
- [ ] Alert thresholds are set relative to actual historical variance, not picked as a round number with no data behind it
- [ ] Different alert categories (duration, flakiness, infra, cost) route to appropriately different urgency/channels, not one undifferentiated stream
- [ ] Alerts include the actual metric values that triggered them, not just a generic "something degraded" message
- [ ] Alert false-positive rate is periodically reviewed, with thresholds recalibrated when an alert condition is routinely dismissed as noise
- [ ] Test pipeline cost (compute minutes, third-party tool/API spend) is tracked as its own trend line, not discovered only via a surprising bill
```
