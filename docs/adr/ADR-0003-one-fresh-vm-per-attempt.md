# ADR-0003: One fresh VM per attempt; nothing persists

- **Status:** Proposed
- **Date:** 2026-11-27
- **Deciders:** sandbox-runner maintainers (human review required, T2+)
- **Related:** [attempt-lifecycle.md](../spec/attempt-lifecycle.md), [security.md](../spec/security.md) T9

## Context

The brief says the runner must not keep state beyond the attempt, must not share writable caches across
attempts, and must destroy the VM, its workspace and its caches.

## Decision

We will set up each attempt as follows:

- **Fresh resources.** Every attempt gets a new VM, a new workspace and a new per-attempt cache.
- **Teardown on every path.** Teardown runs on every outcome, guarded against panics. An orphan sweep runs
  at startup and on a timer.
- **Quarantine.** If teardown cannot be confirmed, the host is quarantined.
- **Telemetry archives.** The per-attempt telemetry log archive is kept as *evidence* outside the workspace.
  No later attempt can read it, and it is never mounted into a VM (see ADR-0013).
- **L2 caches.** Dependency caches at L2 are read-only to VMs.

## Alternatives considered

| Option | Pros | Cons |
| --- | --- | --- |
| Fresh VM per attempt | Strong isolation; simple reasoning | Start latency (mitigated by L2 pre-warm) |
| VM reuse across attempts | Faster | Contamination; violates brief |
| Shared writable cargo cache | Faster builds | Cache poisoning across attempts |

## Consequences

- Cold builds are slower in L1. This is accepted.
- An idempotency journal holds only a mapping from attempt to result, until the result has been delivered.

## Compliance

Assertions 32–35 and the RT-ISO red-team tests.
