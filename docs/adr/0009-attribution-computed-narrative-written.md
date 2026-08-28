# ADR-0009 — Bottleneck attribution is computed; the model only names and narrates

**Status:** Accepted · **Date:** 2026-08-28 · **Extends** [ADR-0006](0006-deterministic-health-ai-narrative.md)

## Context

Management wants to know where the bottleneck is, and whether it lies in the strategy, in
external conditions, or in a person — and whether it affects one person or everyone.

The tempting implementation hands a month of answers to a model and asks it. It demos
extremely well. It is also the single most dangerous thing this system could do, because the
output is a confident paragraph about a named colleague, produced from evidence nobody can
inspect, at a sample size too small to support it.

## Decision

Attribution is arithmetic over structured data. The model does three narrow jobs.

**Computed:**
- Stage funnel, time-in-stage, stall and regression detection
- Cause-code frequency per player, per team, per window
- Concentration vs. systemic classification, against fixed published thresholds
- All rates, counts and sample sizes
- Cluster formation over answer embeddings

**Model:**
- Proposing a `cause_code` for free-text detail (classification, human-confirmed)
- Naming a cluster (accepted by an author before it enters the taxonomy)
- Writing the narrative — from computed findings only, never from raw answers

A finding cannot be written with a sample size below 5; the constraint is in the schema, not
in the query.

## Consequences

**Good**
- Every claim traces to a count a human can check. "47% of stalled deals cite internal
  approval, n=62" is auditable in a way a generated paragraph is not.
- Findings are reproducible and stable across model versions, so month-over-month trends
  mean something.
- Thresholds are published and tunable, so the team can argue with them — which is the right
  kind of argument to have.
- The dashboard works with AI switched off. Management loses the paragraph, not the analysis.
- Model cost is bounded and tiny: one narrative per module per period over a small
  structured payload, rather than a monthly pass over every answer.

**Bad, accepted**
- Blunt thresholds instead of a statistical test. At n≈10–20 a two-proportion test is
  underpowered anyway; conservative thresholds plus a visible sample size are more honest
  than a p-value that implies precision the data does not have.
- Findings are limited to causes someone put in the taxonomy. Mitigated by the theme-mining
  loop in [doc 17](../17-answer-analysis.md) §17.5, which feeds new causes back in.
- More implementation than one prompt over a table.

**Non-negotiable within this decision:** `capability` findings — the ones about a person —
never trigger automation, never change a score, and are always shown with sample size and
confounders. The order of surfacing puts `external`, `process`, `resource` and `strategy`
first, because when a whole team is blocked those are usually the answer, and a person-shaped
explanation is both the most damaging and the most likely to be wrong.
