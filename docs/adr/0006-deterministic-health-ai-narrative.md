# ADR-0006 — Deterministic health metrics; AI writes only the narrative

**Status:** Accepted · **Date:** 2026-08-18

## Context

Slide 6 of the proposal shows the flagship view: a project card reading
`Status: AT RISK · Progress: 68% · Open issues: 4 · Critical blockers: 2`, above an
"AI Management Summary" paragraph with three recommended actions.

It would be simpler to ask a model to produce the entire card.

## Decision

Split the card at the boundary between fact and interpretation.

**Computed by SQL and rules:** health state, progress %, open issue count, critical blocker
count, overdue count, days since activity, the identified risk issues, and
`triggered_rule` — the named rule that produced the health state.

**Produced by the model:** the narrative paragraph and the ranked recommended actions,
each linked to a specific issue where one applies.

Rule thresholds live in `organizations.settings`, not in code. The health endpoint returns
`ai_summary` as a nullable sub-object.

## Consequences

**Good**
- The numbers are reproducible, explainable and auditable. A manager who asks "why is this
  at risk" gets `triggered_rule: milestone_at_risk`, not a paraphrase.
- The card renders correctly and completely when the AI layer is unavailable, over budget,
  or disabled. The most-used management view has no hard dependency on a third party.
- Health can be recomputed cheaply on every relevant event; only the narrative is
  debounced and cached, which is where the cost is.
- Historical snapshots are comparable over time, because the rules that produced them are
  versioned and deterministic. Model-generated statuses would drift silently across model
  versions and make trend charts meaningless.
- The model works on a small, structured input rather than a raw event dump — better output,
  fewer tokens.

**Bad, accepted**
- Health rules must be maintained and tuned by hand as Lightrees learns what "at risk"
  means for their work. This is a feature: the definition becomes explicit and arguable
  instead of hidden inside a prompt.
- Rules will occasionally disagree with the narrative if the model reads the situation
  differently. The UI shows the rule as authoritative and the narrative as commentary.
- Slightly more implementation than one prompt.

**Revisit:** not the split — it is load-bearing. What may change is the rule set, which is
configuration and expected to be tuned during the pilot.
