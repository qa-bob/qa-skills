# Closure Reports & Audience-Aware Writing

The end-of-cycle summary — the document most likely to actually be read by people outside the immediate test team, and therefore the one where audience-aware writing matters most.

---

## Closure Report Structure

```markdown
# Test Closure Report — [Release/Feature/Cycle Name]

## Executive Summary
[2-4 sentences, non-technical: what was tested, overall risk posture, and the
ship/no-ship recommendation if one is being made. This is the section most
readers will read and nothing else — it needs to stand completely on its own.]

## Scope Summary
[Brief recap of what was in/out of scope — link to the full test plan for detail,
don't repeat it verbatim here.]

## Results Summary
| Metric | Value |
|---|---|
| Test cases executed | 142 / 145 (98%) |
| Pass rate | 96% |
| Defects found | 8 (0 Critical, 1 High, 4 Medium, 3 Low) |
| Code coverage | 84% |
| Open defects at closure | 1 High (see Risk Acceptance below), 2 Medium (scheduled) |

## Key Findings
[The 2-4 findings that actually matter — not an exhaustive defect list (link to
the tracker for that). What would change someone's ship decision if they only
read this section?]

## Risk Acceptance / Deferred Items
[Anything not fully resolved, with an explicit disposition and owner — "we know
about this, here's who owns it and when it's addressed" rather than a silent gap]

## Lessons Learned
[What would make the NEXT cycle better — process issues, environment issues,
gaps in the original plan discovered during execution]

## Sign-off
[Who reviewed and approved this closure, and when]
```

---

## Writing the Executive Summary So It Actually Stands Alone

The executive summary needs to work as a complete, decision-supporting unit even if literally nothing else in the report gets read — this is a much higher bar than "a summary of the rest of the document," and it's worth drafting last, after the detailed sections are written, specifically distilling down to what a time-pressured reader needs.

```markdown
BEFORE (buries the actual signal in hedge-y, non-committal language):
"Testing was conducted across multiple areas of the checkout flow. Several
issues were identified and most have been addressed. Some items remain open
and are being tracked. Overall testing went reasonably well."

AFTER (states the actual signal plainly):
"Checkout flow testing found one High-severity defect (payment retry logic
double-charges under specific network-timeout conditions) that is NOT yet
fixed — recommend holding this release until it's resolved, or explicitly
accepting this risk with a documented mitigation (e.g. feature-flag the retry
path off) if the release date can't move. All other testing passed with no
blocking issues."
```

The "after" version takes a real editorial stance — it says what matters and what it recommends — rather than staying neutral to avoid being wrong. A neutral, hedge-everything summary protects the author from ever being blamed for a bad call, but it also fails at the summary's actual job, which is to give the reader enough signal to make a decision without reading further.

---

## Reporting Metrics Honestly

Closure reports are where the temptation to present metrics more favorably than the underlying reality is strongest, since this is often the document that determines whether a release is perceived as ready. Resist reporting metrics in a way that's technically true but misleading:

| Misleading framing | Honest framing |
|---|---|
| "96% pass rate" with no mention that the 4% includes an unresolved High-severity defect | State the pass rate AND call out any unresolved high-severity items explicitly in the same breath — a high pass rate doesn't make a Critical defect not-a-blocker |
| "Coverage increased to 84%" with no mention that coverage decreased in the specific module with the most defects historically | Report coverage trends per-module for anything with a known defect-density history, not just an aggregate number that can hide a regression in the riskiest area |
| Silently dropping a test category from this cycle's report because it wasn't executed | Explicitly state what wasn't executed and why, in the Scope Summary — an absence that isn't mentioned reads as "everything was covered," which is a misleading impression even if never stated directly |

---

## Tying Back to Test Pipeline Metrics

If the broader practice includes ongoing pipeline observability (see `test-pipeline-observability`), the closure report is a good place to surface a trend view, not just this cycle's snapshot — "pass rate has been stable at 94-96% over the last 4 releases" carries different weight than a single cycle's number in isolation, and it's a cheap addition if that trend data already exists from ongoing metrics tracking.

---

## Checklist

```
- [ ] Executive summary states a clear recommendation/signal, not a hedge-everything neutral summary
- [ ] Metrics are reported alongside the context needed to interpret them honestly (unresolved severity, per-module trends, what wasn't executed)
- [ ] Deferred/open items have an explicit disposition (owner + date, or documented risk acceptance) — no silent gaps
- [ ] Lessons learned are captured for the next cycle, not just a status report on this one
- [ ] Trend context (if available from ongoing pipeline metrics) is included alongside this cycle's snapshot numbers
```
