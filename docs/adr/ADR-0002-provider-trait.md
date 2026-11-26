# ADR-0002: Provider trait over microVM backends

- **Status:** Proposed
- **Date:** 2026-11-27
- **Deciders:** sandbox-runner maintainers (human review required, T2+)
- **Related:** [architecture.md](../spec/architecture.md), [responsibilities.md](../spec/responsibilities.md)

## Context

L1 uses microsandbox (a libkrun microVM). L2 needs a second provider, Firecracker or Kata
Containers. The service must not depend on one cloud or one hypervisor stack.

## Decision

We will define a `SandboxProvider` port with these operations:

- create from a pinned image, with the limits, the mounts, a network attachment to the proxy only, and a
  telemetry channel;
- start;
- wait;
- kill;
- destroy and confirm;
- report usage;
- list owned sandboxes (used by the orphan sweep);
- a capabilities descriptor.

microsandbox is the reference adapter. Domain code depends only on the trait.

## Alternatives considered

| Option | Pros | Cons |
| --- | --- | --- |
| Trait + adapters | L2 provider without domain changes; fakes for tests | Lowest-common-denominator risk |
| Hard-code microsandbox | Faster M1 | Rewrite at L2; no fakes |
| Shell out to provider CLIs only | Simple | Weak error typing; harder to verify teardown |

## Consequences

- Domain logic can be tested with a fake provider.
- The capabilities descriptor lets the runner refuse to run when a provider lacks a required feature, such
  as a read-only root filesystem, vsock or memory limits.
- The microsandbox capability spike is the first provider work item.

## Compliance

- The domain crates do not depend on any provider crate (checked in CI).
- The provider contract test suite runs against every adapter.
