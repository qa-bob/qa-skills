---
name: llm-evaluation
description: Design and run systematic evaluations of LLM-backed products — chatbots, agents, and RAG pipelines. Covers behavioral/consistency testing, adversarial and prompt-injection resistance, safety/bias/PII auditing, RAG retrieval-and-generation quality, and LLM-as-judge benchmark design. Use this skill whenever the user wants to test, evaluate, benchmark, red-team, or grade an LLM/AI agent/chatbot/RAG system, mentions hallucination, prompt injection, jailbreaks, groundedness, or comparing model/prompt versions, or asks "how do we know if this AI feature actually works" — even without using formal evaluation terminology.
---

# LLM / Agent / RAG Evaluation

Evaluating an LLM-backed system is a different discipline from evaluating traditional software, for one structural reason: the same input can legitimately produce different outputs on different runs, and "different" doesn't automatically mean "wrong." A test suite built on exact-match assertions — the default reflex for anyone coming from conventional QA — breaks immediately against this kind of target. This skill provides the methodology built for that reality: how to measure behavior that's expected to vary, how to test for misuse rather than just malfunction, and how to build a grading process that doesn't quietly grade itself.

**Boundary:** this skill evaluates AI systems for correctness, safety, and robustness — the same due diligence any product team should apply before shipping an AI feature. It is not a guide to jailbreaking production systems you don't own, nor to building deceptive AI. The adversarial-testing techniques here mirror `security-testing`'s boundary: testing systems you're authorized to test, to make them safer.

---

## Why This Isn't Just "Testing, But For AI"

Three properties of LLM systems break assumptions that hold for almost all other software under test:

1. **Non-determinism is often correct, not a bug.** Two runs of the same prompt can legitimately produce different (but both acceptable) phrasings. Testing needs to target the *property* that must hold (factual accuracy, refusal of a harmful request, retrieval of the right source) rather than the *exact string* produced.
2. **The most common grading tool is itself an LLM, which creates a circularity risk.** Using an LLM to judge an LLM's output is often the only scalable option, but if the judge shares blind spots with the system under test — or worse, if the same model grades itself — passing scores can reflect shared bias rather than actual quality. Ground truth must come from a source independent of the system under test wherever the stakes justify the extra cost of building it.
3. **Failure modes include misuse, not just malfunction.** A traditional bug is the system doing the wrong thing by accident. An LLM system can also be *induced* into doing the wrong thing on purpose — via prompt injection, jailbreak framing, or poisoned retrieved content — which is a security-adjacent failure mode traditional functional testing was never designed to catch.

---

## The Five Evaluation Dimensions

| Dimension | Question it answers | Reference |
|---|---|---|
| Behavioral & consistency | Does the system do the right thing, and does it keep doing it reliably across runs and turns? | `references/behavioral-and-consistency-testing.md` |
| Adversarial & safety | Can the system be manipulated into doing something it shouldn't, and does it avoid producing harmful/biased/leaked-PII content unprompted? | `references/adversarial-and-safety-testing.md` |
| RAG quality | If the system retrieves and grounds answers in external content, is the retrieval relevant and is the generated answer actually faithful to what was retrieved? | `references/rag-evaluation-metrics.md` |
| Benchmark & comparison | How does this version compare to the last one, to a competitor, or to a fixed quality bar — with a number that means something? | `references/llm-as-judge-and-benchmarking.md` |
| (Cross-cutting) Ground truth provenance | Where did the "correct answer" used to grade any of the above actually come from? | Covered inline below — this one isn't a separate reference because it governs all four of the others |

Most real evaluation efforts need at least two of these dimensions — a customer-support RAG chatbot needs RAG quality *and* adversarial/safety (it's both retrieval-grounded and public-facing); a pure benchmark comparison between two prompt versions might need only the benchmark dimension. Scope to what's actually needed rather than running the full matrix by default.

---

## The One Rule That Governs Everything Else: Ground Truth Independence

Every dimension above eventually needs an answer to "what does *correct* look like here?" That answer — the ground truth — must come from a source independent of the system being evaluated, or the evaluation can pass things it shouldn't.

Concretely: if you're testing a RAG chatbot, the "correct" answer to each eval question should be written by a human with access to the source documents (or derived from an authoritative external source), **not** generated by asking the same chatbot and treating its own answer as the target. If you're using an LLM judge to grade output quality, the judge should ideally be a different, typically stronger model than the one under test — using the same model to grade itself is workable for cheap, high-volume triage, but any finding that matters should be spot-checked against an independent source before being trusted.

This single principle is the most common thing skipped under time pressure, and it's the reason evaluation results silently stop meaning anything when skipped: a self-graded, self-generated-ground-truth eval will trend toward "everything looks great" regardless of actual quality, because the same blind spots grade the same blind spots.

---

## Decision Tree — Which Dimension to Reach For

```
What's the actual question being asked?
├── "Does it still work after we changed the prompt/model/RAG corpus?"
│   → Behavioral & consistency (references/behavioral-and-consistency-testing.md)
├── "Can someone get it to say/do something it shouldn't?"
│   → Adversarial & safety (references/adversarial-and-safety-testing.md)
├── "Is it giving accurate, grounded answers from our knowledge base, or making things up?"
│   → RAG quality (references/rag-evaluation-metrics.md)
├── "Is version B actually better than version A, with a number I can put in a doc?"
│   → Benchmark & comparison (references/llm-as-judge-and-benchmarking.md)
└── "We don't know yet — this is a brand-new AI feature about to ship"
    → Start with behavioral & consistency for the core happy paths, add adversarial &
      safety before any public launch, add RAG quality if retrieval is involved,
      and only add formal benchmarking once there's a second version to compare against
```

---

## Reporting

LLM evaluation findings need one extra field beyond a normal test report: what generated the expected answer, since that's the fact a reader needs to trust the finding at all.

```markdown
### [Dimension] Short, specific finding title
**Input:** the exact prompt/query/conversation that was tested
**Expected:** what the system should have done or said
**Ground truth source:** who/what determined "expected" — a human SME, an authoritative
  document, a stronger independent model, or (lower confidence) the same-model self-grade
**Actual:** what the system actually produced
**Severity:** how much this matters in context — a wrong answer to a low-stakes FAQ
  is not the same severity as a wrong answer to a medical or financial question
**Reproducibility:** did this happen once, or N times out of M runs — single-run findings
  on a non-deterministic system need a repeat-rate before they're actionable
```

---

## Checklist

```
- [ ] Ground truth for every eval question traces to a source independent of the system under test
- [ ] Assertions target the property that must hold, not an exact-string match, for anything non-deterministic
- [ ] Adversarial/safety testing included before any public-facing launch, not treated as optional polish
- [ ] If an LLM judge is used, its independence from the system under test is documented (different model, or explicit lower-confidence flag if same model)
- [ ] Findings include a reproducibility rate for non-deterministic behavior, not just a single failing run
- [ ] Benchmark comparisons report a real statistical comparison (see llm-as-judge-and-benchmarking.md), not just "version B scored higher on 10 examples"
```
