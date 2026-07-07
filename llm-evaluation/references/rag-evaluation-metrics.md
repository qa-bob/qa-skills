# RAG Evaluation Metrics

Retrieval-Augmented Generation has two failure surfaces that need separate measurement: the **retrieval** step (did we find the right source material?) and the **generation** step (did the model actually use that material correctly?). A system can fail at either independently — good retrieval with poor generation looks the same to an end user as poor retrieval with good generation (both produce a wrong answer), but the fix is completely different, so evaluation has to separate them to be useful.

---

## Retrieval Metrics

Answer: did the system find the right documents/chunks, independent of what the model did with them afterward?

| Metric | What it measures | How to compute |
|---|---|---|
| Context precision | Of the chunks retrieved, what fraction were actually relevant? | (# relevant chunks retrieved) / (# chunks retrieved) |
| Context recall | Of the chunks that *should* have been retrieved (per ground truth), what fraction actually were? | (# relevant chunks retrieved) / (# relevant chunks that exist) |
| Retrieval rank quality | When multiple chunks are relevant, are the most relevant ones ranked highest? | MRR (Mean Reciprocal Rank) or NDCG against a ground-truth relevance ranking |

Computing recall requires knowing, per test question, which chunks in the corpus *should* be retrieved — this has to be built as a ground-truth artifact (a "golden" query-to-relevant-chunk mapping), independent of what the retriever actually returns, or recall becomes circular and meaningless.

---

## Generation Metrics

Answer: given what was actually retrieved, did the model produce a good answer from it?

| Metric | What it measures | How to compute |
|---|---|---|
| Faithfulness / groundedness | Does every claim in the answer trace back to the retrieved content, with nothing invented? | Typically LLM-judge: decompose the answer into individual claims, check each against the retrieved context |
| Answer relevancy | Does the answer actually address the question asked, independent of whether it's grounded? (A perfectly faithful answer to the wrong question still fails the user) | LLM-judge scoring, or embedding similarity between question and answer as a cheaper proxy |
| Answer correctness | Compared to a ground-truth answer, is this substantively correct? | LLM-judge comparison against ground truth, or exact/fuzzy match for short-form factual answers |

**Faithfulness is the metric that catches hallucination specifically** — a RAG system can retrieve perfect source material and still hallucinate on top of it (adding a detail the source didn't contain, misattributing a number, drawing an unsupported inference). Faithfulness scoring exists to catch exactly that gap between "the source was available" and "the model actually stuck to it."

---

## End-to-End Metrics

| Metric | What it measures |
|---|---|
| Answer semantic similarity | Embedding-based similarity between generated and ground-truth answers — a cheap, automatable proxy for correctness, useful for regression detection at scale even though it's less precise than LLM-judge scoring |
| Citation accuracy | If the system cites sources, do the citations actually correspond to where the claim came from (not just any retrieved document)? |

---

## Diagnosing Failures by Combining the Two Surfaces

| Retrieval | Generation | Diagnosis |
|---|---|---|
| Good (high precision/recall) | Good (high faithfulness) | System working correctly |
| Good | Poor (low faithfulness) | Generation problem — model is hallucinating or ignoring good context. Fix: prompt engineering, or a stronger/better-instructed model |
| Poor (low recall) | Good faithfulness to what *was* retrieved | Retrieval problem — the model is being faithful to incomplete information. Fix: embedding model, chunking strategy, indexing, or query rewriting |
| Poor | Poor | Compounding failure — fix retrieval first, since generation quality can't be fairly assessed against bad context |

This is the single most useful diagnostic habit in RAG evaluation: never report "the RAG system got this wrong" without first checking which of the two surfaces actually broke, since the two failure modes point at completely different parts of the system to fix.

---

## Adversarial RAG-Specific Testing

Beyond the general adversarial testing in `adversarial-and-safety-testing.md`, RAG systems have corpus-specific attack surfaces worth testing directly:

- **Cross-tenant retrieval bleed:** in a multi-tenant RAG system, does a query scoped to tenant A ever surface a chunk belonging to tenant B? Test explicitly — this is a data-isolation failure, not just a quality issue.
- **Near-duplicate/contradictory document handling:** when the corpus contains two similar documents with conflicting information (e.g. an old and a new policy version), does the system retrieve and cite the current one, or does it non-deterministically pick either?
- **Ground-truth freshness:** if the corpus is updated, are evaluation ground-truth answers re-validated against the new corpus? A ground-truth answer written against last quarter's policy document will incorrectly fail a system that's now correctly citing the updated policy.

---

## Checklist

```
- [ ] Retrieval and generation quality measured and reported separately, not as one blended score
- [ ] Ground-truth relevant-chunk mapping exists independently of what the retriever returns (for recall)
- [ ] Faithfulness/groundedness specifically measured, not just answer-correctness — this is what catches hallucination-on-top-of-good-retrieval
- [ ] Failure diagnosis follows the retrieval-vs-generation split before recommending a fix
- [ ] Multi-tenant corpora tested for cross-tenant retrieval bleed
- [ ] Ground-truth answers re-validated after any corpus update, not left stale
```
