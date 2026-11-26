# ADR-0010: Budgets enforced by the host

- **Status:** Proposed
- **Date:** 2026-11-27
- **Deciders:** sandbox-runner maintainers (human review required, T2+)
- **Related:** [attempt-lifecycle.md](../spec/attempt-lifecycle.md) §3, [constraints.md](../spec/constraints.md)

## Context

The brief requires that wall-clock, CPU and memory budgets are enforced outside the agent, and that token
and turn budgets are enforced by the model gateway, keyed by the run.

## Decision

- **CPU and memory.** The provider applies the VM's vCPU and memory limits. The host applies cgroup limits
  around the VMM process.
- **Wall clock.** A host watchdog kills the VM at the wall-clock limit, with a precision of 5 s or better.
- **Disk.** Disk and output caps are applied as quotas or as size limits on the tmpfs and the output area.
- **Tokens and turns.** The runner does not enforce these. It passes `run_key` through the proxy so that the
  gateway can enforce them.
- **Host-side phases.** Collection and publishing have their own timeouts, separate from the agent
  budget.

## Alternatives considered

| Option | Pros | Cons |
| --- | --- | --- |
| Host enforcement | Tamper-proof | Provider capability dependency |
| task-marshal self-limits | Easy | Agent-controlled |

## Consequences

Budget enforcement depends on the capabilities of the provider. The microsandbox spike must confirm that it
supports these limits.

## Compliance

Assertions 19 and 20, and the RT-RES tests.
