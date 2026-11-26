# ADR-0011: Library and CLI invocation for M1

- **Status:** Proposed
- **Date:** 2026-11-27
- **Deciders:** sandbox-runner maintainers (human review required, T2+)
- **Related:** [interfaces.md](../spec/interfaces.md), cog-works ADR-0009 (provider-neutral runtime interface) **[EXT]**

## Context

In M1, humans start runs from the CLI. CogWorks will call the runner through its provider-neutral runtime
interface (cog-works ADR-0009), which we have not reviewed yet. gate-warden also calls the runner.

## Decision

- **M1 transport.** For M1 the runner is a Rust library plus a CLI:
  - `run` takes a request file or stdin and prints the result as JSON on stdout;
  - `tail` shows live or archived telemetry;
  - `reap` sweeps orphaned resources.
- **No network listener.** The runner opens no network listener in L1. Its control and tail interfaces are
  local Unix sockets.
- **Revisit.** We will review cog-works ADR-0009 and decide whether to add a service mode, for example gRPC
  or HTTP over mTLS with OIDC. If we add one, it will reuse the same request and result schemas.

## Alternatives considered

| Option | Pros | Cons |
| --- | --- | --- |
| Library + CLI | Minimal surface; matches M1 | Caller must run on the KVM host or wrap the CLI |
| Network service now | Remote callers | Extra auth surface before it is needed |

## Consequences

- In M1, the callers, or the humans standing in for them, run on, or reach, the KVM host.
- A service mode would need its own ADR.

## Compliance

There is no listening socket other than the proxy's link to each VM. A test enforces this.
