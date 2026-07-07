# PII Anonymization & Masking

For the exception case where synthetic data genuinely can't provide the realism needed and production-derived data is the only practical option. Treat everything in this file as a process that needs sign-off, not a routine step — the goal is to make the exception path safe, not to make it easy to reach for by default.

---

## Direct vs. Indirect Identifiers

The most common anonymization mistake is stopping after removing direct identifiers and treating the result as safe.

| Type | Examples | Why removal alone isn't enough |
|---|---|---|
| Direct identifiers | Name, email, SSN, phone number, exact street address | Removing these is necessary but not sufficient |
| Indirect / quasi-identifiers | ZIP code, birth date, gender, employer, rare medical condition, exact timestamp of an action | A *combination* of quasi-identifiers can uniquely re-identify a person even with zero direct identifiers present — the classic finding is that ZIP code + birth date + gender alone uniquely identifies a large majority of the US population |

Any anonymization process needs to explicitly account for quasi-identifiers, either by generalizing them (exact birth date → birth year, exact ZIP → first 3 digits or region) or by applying a formal privacy technique (below) that accounts for combinations, not just individual columns.

---

## Masking vs. Tokenization vs. Generalization

| Technique | What it does | When to use it |
|---|---|---|
| Masking | Replace part of a value with a fixed pattern (e.g. `***-**-1234` for an SSN) | Good for display/UI testing where the *shape* of the data matters but the value doesn't need to be usable for joins |
| Deterministic tokenization | Replace a value with a consistent token derived from it (same input → same token every time) | Needed when test data must preserve relationships — e.g. the same customer's orders across multiple tables must still resolve to "the same customer" post-anonymization, just not the real one |
| Generalization | Replace a precise value with a less precise category (exact age → age bracket, exact location → region) | Reduces re-identification risk from quasi-identifiers while preserving enough signal for realistic-distribution testing |
| Full replacement with synthetic value | Replace with an entirely fabricated but plausible value (real name → a Faker-generated name) | Best when the specific value doesn't need to relate back to the original at all |

**Deterministic tokenization is the one that needs the most care**: it must be irreversible (one-way, e.g. a keyed hash) so the token can't be reversed back to the original value, but consistent (same input always produces the same output within a given dataset/run) so relationships in the data are preserved. A naive "just hash it" approach is close to right but needs a secret key/salt controlled separately from the data — an unsalted hash of a predictable value space (like SSNs) can be reversed by brute-force pre-computation.

---

## Regulatory Context

Different regulatory frameworks define "anonymized" differently, and this affects what's actually defensible, not just technically achievable:

| Framework | Relevant concept | Practical implication |
|---|---|---|
| HIPAA (US healthcare) | Safe Harbor method defines 18 specific identifier categories that must be removed/generalized | Provides a concrete checklist, but Safe Harbor compliance doesn't automatically mean zero re-identification risk for small/unusual populations |
| GDPR (EU) | "Anonymization" must be irreversible; if it's reversible even in principle by *anyone* (including the org itself), it's "pseudonymization," which is still regulated personal data | Tokenization/masking done reversibly (org retains the mapping) is pseudonymization, not anonymization — different, weaker compliance posture, and test/staging environments holding pseudonymized data are still in regulatory scope |
| PCI-DSS (payment cards) | Full PAN (card number) must never exist in test/dev environments in usable form | Use test-mode tokens from the actual payment processor's sandbox, never masked/anonymized versions of real card numbers |

**This isn't legal advice — the underlying regulatory determination should involve whoever handles compliance for the organization.** What this skill provides is the testing-side discipline: know which category your anonymization actually falls into (true anonymization vs. pseudonymization) before assuming a given technique clears the compliance bar, and default to the payment-processor's own sandbox/test-token systems for anything card-related rather than attempting to anonymize real PANs at all.

---

## A Practical Anonymization Workflow

```
1. Inventory every field in the source data — classify each as: direct identifier,
   indirect/quasi-identifier, or non-identifying
2. Apply the appropriate technique per field:
   - Direct identifiers → full synthetic replacement (not masking — full replacement)
   - Quasi-identifiers → generalization, or deterministic tokenization if
     relationships must be preserved
   - Non-identifying fields → can often be left as-is, but check for accidental
     free-text PII (support ticket notes, comment fields) — these are commonly missed
3. Verify no small/rare subgroup remains uniquely identifiable by the remaining
   quasi-identifier combination (k-anonymity check: does every combination of
   retained quasi-identifiers apply to at least k records, not just 1?)
4. Document the technique applied per field — this record is itself an artifact
   an auditor or compliance reviewer will want to see
5. Restrict access to the anonymized dataset with the same rigor as the source data
   until step 3 is actually verified, not assumed
```

---

## Checklist

```
- [ ] Quasi-identifiers explicitly identified and handled, not just direct identifiers
- [ ] Tokenization (if used) is keyed/salted and the key is managed separately from the data
- [ ] The distinction between true anonymization and pseudonymization is understood for this specific dataset, and treated accordingly for compliance scope
- [ ] Payment card data uses the processor's sandbox/test tokens, never an anonymized version of a real PAN
- [ ] A k-anonymity (or equivalent) check confirms no small subgroup is still uniquely identifiable after processing
- [ ] The anonymization process itself is documented per field, as an auditable artifact
```
