---
name: test-documentation
description: Write test plans, individual test case documentation, and end-of-cycle closure reports for QA and testing efforts. Use this skill whenever the user needs a test plan, test strategy document, test case template, traceability matrix, closure report, or any stakeholder-facing summary of testing work — especially when the audience includes non-technical stakeholders (executives, auditors, product leadership) who won't read test code directly.
---

# Test Documentation & Closure Reporting

Test documentation exists to let someone who didn't personally run the tests trust that testing happened, understand what it covered, and act on what it found — an executive deciding whether to ship, an auditor confirming compliance evidence exists, a new team member trying to understand what's already covered before writing more tests. The most common failure in test documentation is writing one document that tries to serve all of these readers equally, which tends to serve none of them well: a document dense enough to satisfy an auditor's evidence requirements is usually too dense for an executive skimming for a ship/no-ship signal, and a document light enough for quick executive reading usually lacks the traceability an auditor needs.

This skill treats test documentation as a set of distinct artifacts for distinct audiences and distinct moments in the test lifecycle — a plan (before testing, defining scope and intent), test case records (during testing, the granular evidence), and a closure report (after testing, the summary and the decision-support document) — rather than one document trying to do all three jobs.

---

## When to reach for each part of this skill

| Situation | Do this |
|---|---|
| Starting a test effort — need to define scope, approach, entry/exit criteria | Read `references/test-plan-authoring.md` |
| Documenting individual test cases with enough structure to be traceable and auditable | Read `references/test-case-documentation-standards.md` |
| Wrapping up a test cycle — need a report stakeholders will actually read and act on | Read `references/closure-reports-and-audience-aware-writing.md` |
| None of the above — just getting oriented | Keep reading this file |

---

## The Three Artifacts and When Each Gets Written

```
BEFORE testing        DURING testing              AFTER testing
─────────────────    ──────────────────           ─────────────────
Test Plan             Test Case Records            Closure Report
(scope, approach,      (per-test-case evidence:     (summary, metrics,
 entry/exit criteria,   preconditions, steps,        defects, decision-
 risk prioritization)   expected/actual, status)     support recommendation)

references/            references/                 references/
test-plan-authoring.md test-case-documentation-      closure-reports-and-
                        standards.md                 audience-aware-writing.md
```

Skipping the plan and going straight to ad hoc testing tends to produce coverage that reflects whatever was top-of-mind rather than an actual risk-based strategy. Skipping test case records and going straight from "we tested it" to a closure report leaves the closure report's claims unverifiable — anyone asking "how do we know this was actually tested" has nothing to point to. Skipping the closure report leaves the results locked in raw test-case data that no stakeholder outside the immediate test team will ever actually read.

---

## Core Principle: Write for the Reader Who Will Act On It

Before writing any test documentation artifact, identify who's actually going to read it and what decision or action it needs to support — this determines structure and depth far more than any generic template does.

| Reader | What they need | What they don't need |
|---|---|---|
| Executive / product leadership deciding whether to ship | A short, decisive summary: what was tested, overall risk posture, the 2-3 findings that actually matter | Full traceability matrices, every individual test case, methodology detail |
| Engineer picking up where testing left off | Detailed coverage information: what's tested, what's explicitly out of scope, known gaps | A polished narrative summary — they need the working detail, not the pitch |
| Auditor / compliance reviewer | Traceable evidence: which requirement maps to which test case, with a clear pass/fail record and timestamp | Persuasive framing — they need verifiable evidence, not a sales pitch for the quality of the work |
| Future team member (6+ months later) | Enough context to understand *why* a decision was made (why this scope, why this risk was accepted), not just what the scope was | Day-to-day operational detail that's since become stale |

A single document can serve more than one of these audiences with clear internal structure (e.g. an executive summary section up front, detailed appendices after) — the failure mode is writing without deciding this up front and producing something that drifts toward serving whichever audience the author had most in mind, usually at the expense of the others.

---

## Checklist

```
- [ ] The intended primary reader and the decision/action the document supports are identified before writing starts
- [ ] Plan, test case records, and closure report are treated as distinct artifacts for distinct moments, not merged into one document trying to do all three jobs
- [ ] Executive-facing content leads with a short, decisive summary — not buried after pages of methodology
- [ ] Auditor-facing content includes actual traceability (requirement → test case → result), not just a narrative claim that coverage exists
- [ ] Documentation explains *why* key decisions were made (scope choices, accepted risks), not just *what* was decided, so it's still useful to someone reading it much later
```
