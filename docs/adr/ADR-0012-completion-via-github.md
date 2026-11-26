# ADR-0012: Completion signalled via GitHub; no queue events in L1

- **Status:** Proposed
- **Date:** 2026-11-27
- **Deciders:** sandbox-runner maintainers (human review required, T2+)
- **Related:** [attempt-lifecycle.md](../spec/attempt-lifecycle.md) §6, handbook principles **[EXT]**

## Context

GitHub is the only system of record, and services never hand work to each other directly. The brief says a
queue event *may* also be emitted.

## Decision

- **How completion is signalled.** Completion is observable as the pushed stage branch plus an output record.
  The output record is a comment or a committed file, as the stage contract chooses. It is idempotent and
  carries the `attempt_id` and `trace_id`.
- **No queue events in L1.**
- **gate-warden mode.** The runner writes nothing to GitHub. gate-warden publishes its own verdict.

## Alternatives considered

| Option | Pros | Cons |
| --- | --- | --- |
| GitHub-only | Single source of truth; restart-safe | Relies on webhooks for latency |
| GitHub + advisory queue event | Lower latency | Extra dependency; dual signals |
| Queue event as primary | Fast | Violates system-of-record principle |

## Consequences

CogWorks reacts to webhooks that queue-keeper routes.

## Compliance

Assertions 30 and 31.
