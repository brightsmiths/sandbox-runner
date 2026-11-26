# ADR-0007: Scope pre-check before push, fail closed

- **Status:** Proposed
- **Date:** 2026-11-27
- **Deciders:** sandbox-runner maintainers (human review required, T2+)
- **Related:** [attempt-lifecycle.md](../spec/attempt-lifecycle.md) §5, agent-gate **[EXT]**

## Context

The brief requires agent-gate's scope check on the produced commits before pushing, and a structured
rejection when the check fails. agent-gate may be unavailable.

## Decision

- **When the check runs.** The runner calls agent-gate's scope check for `checkout_sha..head`, with the role
  and the effective mount plan, after verifying the bundle and before obtaining any token.
- **Out of scope.** The outcome is `rejected`/`scope_violation` with the violating paths. Nothing is pushed:
  either all commits are pushed or none.
- **agent-gate unavailable.** The outcome is `failed`/`scope_check_unavailable`, which is fail-closed. The
  runner never pushes without an `in_scope` verdict, and the type system enforces this, because a
  `PublishPlan` requires an `InScope` value.

## Alternatives considered

| Option | Pros | Cons |
| --- | --- | --- |
| Fail closed | Boundary holds | Availability coupled to agent-gate |
| Fail open + warn | Availability | Unscoped writes reach GitHub |
| Partial push of in-scope commits | Salvages work | Rewrites history; confusing |

## Consequences

agent-gate's availability is on the critical path. Callers retry `scope_check_unavailable`.

## Compliance

Assertions 24 and 25, and mutation testing at 100 % for the scope-handling module.
