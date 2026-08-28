# ADR-0007 — Follow-up cycles generalize screening

**Status:** Accepted · **Date:** 2026-08-28 · **Supersedes part of** [ADR-0005](0005-ai-proposes-humans-dispose.md) context

## Context

The original design read "Screening" in the proposal deck as customer or candidate intake:
a form, filled once, reviewed, converted to a project. Documents 03 and 04 modelled it that
way.

Clarification from the product owner: **MESSI is Messenger Screening** — a *recurring*
procedural checklist a team member completes every day, confirming they have swept their
chat channels and reporting what came in and what was answered. Alongside it sit two more
follow-up programs with the same shape but different subjects: lead status reports, and
project status reports.

So the deck's central workflow is not intake. It is *recurring follow-up*, and screening is
one instance of it.

Three near-identical subsystems were therefore in prospect: screening templates and
instances; a lead follow-up scheduler; a project follow-up scheduler. Each with its own
versioned question sets, answers, reminders, and conversion-to-project.

## Decision

One engine. A **follow-up module** carries a versioned question set, a cadence, a subject
type and a set of allowed outcomes. An **enrolment** binds a player to a subject under that
module. A **cycle** is one occurrence.

Screening becomes a module with `cadence.kind = "once"`:

| Screening concept | Becomes |
|---|---|
| `screening_templates` | `followup_modules` + `followup_module_versions` |
| `screenings` | `followup_cycles` with `period_key = 'once'` |
| `screening_answers` | `followup_answers` |
| Screening → project conversion | `followup_outcomes` |

Intake-specific review states (`in_review`, `needs_info`, `accepted`, `rejected`) survive as
an optional module capability rather than a property of every module.

## Consequences

**Good**
- MESSI, LEADS and PRISTA are configuration, not three codebases. Adding PRISTA after LEADS
  cost a `subject_kind` value and a pre-fill rule.
- One inbox, one reminder path, one scoring model, one set of events. A player is not asked
  to learn three tools that ask them questions.
- The Phase 3 promise of "custom workflows" is now nearly free: a tenant creating a new
  follow-up program is creating a row, not requesting a feature.
- Question-set versioning, `answers.origin` and outcome provenance — already designed for
  screening — apply unchanged to every module.

**Bad, accepted**
- Documents 03, 04, 05 and 11 describe screening as a standalone subsystem and are now
  partly superseded. They are corrected rather than left to drift, but the design history
  is messier than if this had been understood at the start.
- The generalized model is more abstract. "Screening" was concrete and easy to picture;
  "a module with a subject kind and a cadence" needs the vocabulary table in
  [doc 14](../14-followup-engine.md) §14.2 to stay legible.
- A one-shot intake form now carries fields it does not use (`period_key`, `cadence`). The
  cost is a few always-constant columns, which is cheaper than a parallel subsystem.

**Revisit when:** a module needs a question type or lifecycle that cannot be expressed as
configuration and would distort the shared model to accommodate. At that point it earns its
own subsystem — but the burden of proof sits with the exception.
