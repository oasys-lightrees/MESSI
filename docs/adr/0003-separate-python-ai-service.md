# ADR-0003 — AI in a separate Python service with no data access

**Status:** Accepted · **Date:** 2026-08-18

## Context

The proposal specifies Python / FastAPI for the AI layer, and Rust for the backend. Two
questions remained: is a separate service warranted, and what access should it have?

The pull toward giving it database access is strong — the assistant needs context, and
letting the AI service query directly seems simpler than passing bundles over HTTP.

## Decision

A separate stateless Python service, with **no database credentials, no object storage
credentials and no tool access**.

- All inputs arrive in the request payload.
- All retrieval and authorization filtering happens in the Core API, which then passes a
  pre-authorized context bundle.
- The service is reachable only from the private subnet, authenticated by a signed service
  token and restricted by security group.
- It is idempotent and side-effect-free, so retries are unconditionally safe.

## Consequences

**Good**
- Authorization exists in exactly one component. "Ask questions against authorized
  operational data" is true by construction rather than by careful prompt engineering.
- Prompt injection is contained: an injected instruction reaches a component that has
  nothing to give it. The worst case is a bad string in a summary, not data exfiltration.
- The Python AI ecosystem (evaluation harnesses, tokenizers, provider SDKs) is available
  without contorting the Rust build.
- Prompt and model iteration deploys independently of the transactional core.
- Statelessness makes the service trivially horizontally scalable and trivially testable.

**Bad, accepted**
- An extra network hop and an extra deployable to operate.
- Context assembly logic lives in Rust, where the AI work is otherwise in Python. This is a
  real ergonomic cost and it is the price of the security property; it is not negotiable.
- Large context bundles cross the wire on every call. Bounded by the token ceilings in
  [doc 06](../06-ai-layer.md) §6.4, which we want anyway for cost reasons.
- Two language toolchains in CI.

**Rejected: AI logic inside the Rust backend.** Would remove the hop, but puts prompt
iteration on the critical deploy path of the transactional system and gives the model-facing
code the database credentials — losing the property that makes this design defensible in a
security review.
