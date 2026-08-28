# ADR-0010 — Question quality is gated at authoring, not policed at answering

**Status:** Accepted · **Date:** 2026-08-28

## Context

The team previously ran daily reports in Telegram and they became ineffective — answers
degraded into filler. Any replacement has to answer: what stops that happening again?

Two families of intervention are available.

**Police the answers.** Score them, flag thin ones, chase people who write *"masih follow
up"*. This is where most reporting tools land, and it fails in a specific way: it corrects
nobody, it teaches people to write for whatever is grading them, and it turns the leader into
the enforcer of a form rather than a reader of information. Worse, it treats a *design*
problem as a *discipline* problem — the answers are thin largely because the questions
allowed thin answers.

**Improve the questions.** A question that cannot be answered without doing the work does not
need policing.

## Decision

Quality is enforced where the question is written.

- Five tests — evidence, specificity, consequence, non-redundancy, thinking
  ([doc 16](../16-question-design.md) §16.2) — are the design standard.
- A deterministic linter checks what it can be certain about, at authoring and at publish.
- An AI reviewer evaluates each draft question against the five tests and proposes a rewrite.
  The author accepts, edits or ignores; `question_reviews.disposition` records which.
- Preview and a 30-day dry run run before anyone is enrolled.
- Only one rule blocks publishing: a module with no fallback cadence
  ([ADR-0008](0008-commitment-driven-cadence.md)). Everything else is a warning.

At answering time the system is nearly silent about quality. One optional nudge — *"this
looks like last week's answer, anything new?"* — which the player can ignore.

## Consequences

**Good**
- The highest-leverage use of AI in the product: it acts once, on one author, before any
  player is asked. A better question improves every future answer from everyone.
- Analytics quality is fixed upstream. [Doc 17](../17-answer-analysis.md) cannot recover
  information the questions never captured, so this is the only place the fix works.
- The leader stays a reader, not an enforcer. That relationship is what the product is
  supposed to protect.
- Cheap: a handful of model calls per module version, versus one per answer forever.

**Bad, accepted**
- Bad questions can still ship — the linter warns, it does not block. Blocking would make
  the console hostile and authors would route around it. The dry run and the staleness metric
  catch what the linter misses, after which the module is revised.
- Quality depends on authors engaging with the review. Measured directly: if authors ignore
  most rewrite suggestions, the suggestions are bad, and `question_reviews.disposition` says
  so.
- A module that lints clean can still be the wrong module. No amount of question review
  substitutes for asking whether this follow-up should exist at all.

**Revisit when:** answer staleness stays high across modules that lint clean. That would mean
the failure is in cadence or in the closing of the loop rather than in the questions, and the
intervention belongs elsewhere.
