# ADR-0005 — AI proposes, humans dispose

**Status:** Accepted · **Date:** 2026-08-18

## Context

The proposal lists classification, extraction and automation as AI capabilities. The
tempting implementation writes model output straight into the business record: the
screening arrives, the fields fill themselves, the category is set.

The proposal also states that success is measured by operational improvement, and names
adoption as a risk.

## Decision

Model output is never a business record until a human or an explicitly configured,
confidence-gated rule promotes it.

- AI results are written to `ai_outputs`, a separate table, with `applied_at` null.
- A distinct endpoint (`apply-extraction`, `apply-classification`) moves values into the
  record, per field, and writes `ai_feedback` in the same call.
- Auto-application exists for classification only, requires a `min_confidence` threshold on
  the rule, is recorded as an automation-actor acceptance, and is reversible.
- Deterministic scores, statuses, permissions and metrics are never model-written at all.

## Consequences

**Good**
- A confidently wrong model produces a rejected suggestion, not a wrong customer outcome.
- `ai_feedback` accumulates a real, in-situ quality dataset from day one. Six weeks in, the
  question "is the classifier good enough to trust" has a number, not an argument.
- Users meet AI as an assistant that is sometimes wrong and always correctable — which is
  what it is. Trust is calibrated rather than broken.
- Turning AI off degrades the product to Phase 1 rather than breaking it.
- Provenance (`origin`, `confidence`) is visible on every field, so any record can be
  audited.

**Bad, accepted**
- A review click remains in the loop, so the theoretical maximum time saving is not
  reached. Measured against a baseline of manual re-entry, accepting four pre-filled fields
  is still dramatically faster than typing them — and the KPI will show exactly how much.
- More UI to build: review panels, per-field accept/edit/reject, confidence display.
- Two writes (output, then application) rather than one.

**Revisit when:** a task's edit-free acceptance rate exceeds 90% over ≥ 200 samples. That
is the evidence threshold for widening auto-application — and by then the data to justify
it already exists, which is the point.
