# ADR-0008 — Commitment-driven cadence

**Status:** Accepted · **Date:** 2026-08-28

## Context

A recurring follow-up system has one dominant failure mode, and it is not technical.

Ask 8 players about 20 leads every day and the system generates 160 questions daily. Within
two weeks the answers are *"masih follow up"* pasted into every box. The data becomes
worthless, the players resent the ritual, and the leader goes back to asking people directly
— which is exactly the manual work the system was built to remove. The product does not fail
loudly; it quietly becomes a form nobody reads.

The obvious mitigations — reminder throttling, digests, snooze — treat the symptom. The
cause is that a fixed calendar cadence asks on days when there is nothing to report.

## Decision

The follow-up interval is derived from what the player promised, not from a fixed calendar.

The question set already asks *"Next action apa?"* and *"Kapan?"*. Those two answers are
recorded as a **commitment**: an action, a date, an owner. The next cycle for that subject is
scheduled at the commitment's due date.

- Player says "call PT Anu on Friday" → nothing is asked Tuesday through Thursday. Friday
  the system asks about that specific promise.
- Commitment met by its date → `kept`, and the answer creates the next commitment.
- Date passes with no answer → `broken`, which raises `commitment.broken` and escalates to
  the leader through the existing rules engine.

A fallback cadence (typically weekly or monthly) still applies so a subject with no live
commitment drops to a slow heartbeat rather than disappearing.

## Consequences

**Good**
- Question volume falls by roughly the ratio of commitment interval to daily polling, with
  no loss of coverage. Fewer asks, each one about something specific that was actually
  promised.
- Players set their own rhythm and are held to it. "You said Friday" is a materially
  different conversation from "you didn't fill the form" — the first is about the work, the
  second about the tool.
- The leader dashboard becomes exception-based. Kept commitments need no attention; only
  broken ones surface. An all-green day requires zero reading, which is the actual
  definition of having replaced manual follow-up.
- It produces **commitment-kept rate**, a measure of follow-through rather than of
  form-filling. Submission rate can be gamed in thirty seconds a day; commitment-kept rate
  cannot.

**Bad, accepted**
- A player can set distant dates to reduce how often they are asked. Visible as a rising
  mean commitment interval on the leader dashboard — surfaced as a metric rather than
  blocked by a rule, since a legitimate long interval is common in enterprise sales.
- Cycle generation is more complex than a cron expression: two generators (calendar
  fallback and commitment-triggered) writing into one table, made safe by
  `UNIQUE (enrolment_id, period_key)` with `period_key = 'c:<commitment_id>'` for the
  commitment-triggered case.
- Subjects with no commitment could be neglected if the fallback were omitted. It is not
  optional — a module without a fallback cadence fails validation on publish.
- Players must be taught that the date they enter is a promise the system will hold them to.
  This is a change in working culture, not only in tooling, and should be said out loud when
  the module is rolled out.

**Revisit:** not the mechanism. The fallback intervals and the escalation grace period are
configuration and are expected to be tuned during the pilot.
