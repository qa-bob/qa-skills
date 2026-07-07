# LLM-as-Judge & Benchmarking

Covers how to grade LLM output at scale using another LLM, and how to turn individual graded examples into a benchmark comparison that supports an actual decision ("is version B better than version A?").

---

## Designing an LLM-as-Judge Rubric

A vague judge prompt ("rate this response's quality from 1-10") produces noisy, low-agreement scores. A good rubric decomposes quality into specific, independently-gradable criteria:

```markdown
## Judge Rubric — Customer Support Response Quality

Score each criterion 0 or 1 (binary is more reliable than a 1-10 scale for LLM judges —
finer-grained scales show worse inter-rater agreement in practice):

1. **Factual accuracy**: Does the response avoid stating anything that contradicts
   the provided ground truth or source material?
2. **Completeness**: Does the response address all parts of the user's question?
3. **Appropriate tone**: Is the tone professional and appropriately empathetic for
   the situation described?
4. **Actionability**: If the user needs to do something next, is that clearly stated?
5. **No unsupported claims**: Does the response avoid asserting anything not
   grounded in the provided context? (Only applicable for RAG-grounded responses —
   see rag-evaluation-metrics.md for the dedicated faithfulness metric)

Final score = sum of criteria met. Report per-criterion breakdown, not just the total —
a response that fails criterion 5 (hallucination) needs a different fix than one that
fails criterion 3 (tone), even if they land on the same total score.
```

### Judge model selection

- Prefer a **different, typically stronger** model as judge than the model under test — using the same model to judge itself is workable for fast, low-stakes triage, but creates the circularity risk described in `SKILL.md`.
- **Pin the judge model version explicitly** and re-validate the rubric when it changes — judge models drift in scoring behavior across versions just like any other model output, and an unpinned judge silently changes the meaning of "passing" over time.
- **Validate the judge against human judgment** on a sample before trusting it at scale — have a human grade the same examples using the same rubric, and check agreement. Low agreement means the rubric needs to be more specific, not that the judge model needs to be swapped.

---

## Benchmark Design

A benchmark is a fixed, versioned set of test cases run consistently across comparisons — its value comes entirely from that consistency, so treat the benchmark set itself as an asset that changes deliberately and rarely, not something regenerated fresh for every comparison.

```
1. Fixed test set — same inputs used every time this benchmark runs, versioned
   (e.g. "support-benchmark-v3.jsonl") so results across time are comparable
2. Fixed rubric — same judge rubric applied every time; a rubric change means the
   benchmark version number changes too, since old and new results aren't comparable
3. Multiple runs per input if the system is non-deterministic — report a distribution
   (mean, or pass rate), not a single-run score
4. A held-out slice that's never used to tune prompts/few-shot examples — otherwise
   the benchmark measures how well the system was tuned to the benchmark, not
   real-world quality
```

---

## Comparing Two Versions Properly

"Version B scored 82%, version A scored 78%, so B is better" is not a sound comparison on a small sample — a 4-point difference on, say, 50 examples can easily be noise. Use a real statistical comparison:

- **Paired comparison when possible**: run both versions on the *same* input set (not different random samples) — this controls for input difficulty differing between the two evaluation runs.
- **Report a confidence interval or significance test**, not just a point estimate. For a pass/fail rate, a two-proportion test (or a bootstrap confidence interval) tells you whether an observed difference is likely real or within noise.
- **Report effect size alongside significance**: a statistically significant 1% improvement may not be worth the cost of shipping a change, even if it's "real." Both the size and the confidence matter for the actual ship/no-ship decision.
- **Watch sample size**: small benchmarks (under ~30-50 examples) will struggle to detect anything but large effect sizes — if the benchmark is meant to catch small regressions, it needs to be large enough to have the statistical power to do so.

```
Example writeup:
"Version B showed a 4.2 percentage-point improvement in pass rate (82.1% vs 77.9%,
n=200 paired examples, 95% CI for the difference: [0.8, 7.6], p=0.02). This is a
real but modest improvement — worth shipping, not a dramatic win."
```

---

## Avoiding Common Benchmark Mistakes

| Mistake | Why it's a problem | Fix |
|---|---|---|
| Grading with the same model being tested, with no independent validation | Circular — shared blind spots pass silently | Use a different/stronger judge, or validate same-model grading against a human-graded sample |
| Reusing benchmark examples that were also used to write/tune the prompt | The benchmark measures memorization of its own test set, not generalization | Maintain a held-out set never used for tuning |
| Reporting only an aggregate score | Hides which specific failure categories are driving the number | Report per-criterion/per-category breakdowns alongside the aggregate |
| Comparing unpaired samples across versions | Input-difficulty differences between samples get mistaken for version differences | Run both versions on the identical input set |
| Treating a benchmark result as permanent | Model providers update models behind the same API endpoint; corpora and prompts drift | Re-run the benchmark on a defined cadence, not just once at launch |

---

## Checklist

```
- [ ] Judge rubric decomposes quality into specific, independently-scored criteria — not one vague overall score
- [ ] Judge model is different from (or explicitly validated against) the model under test
- [ ] Judge model version is pinned; rubric re-validated when it changes
- [ ] Benchmark test set is fixed/versioned, with a held-out slice never used for tuning
- [ ] Version comparisons are paired (same inputs) and report significance/confidence, not just a point estimate
- [ ] Benchmarks are re-run on a defined cadence, not treated as a one-time launch gate
```
