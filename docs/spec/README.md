# sandbox-runner specification

sandbox-runner runs **one stage attempt** of the Brightsmiths factory inside a fresh,
strongly isolated microVM. It then collects the commits and records, checks their scope,
pushes them under a minutes-lived role token, and destroys everything. CogWorks uses it
for agent stages. gate-warden uses it to run checks in a clean environment. It exists as
a separate service so that the orchestrator never holds host, virtualisation or push
privileges.

The wider system architecture is described in the handbook repository
[brightsmiths/handbook](https://github.com/brightsmiths/handbook) (`docs/`, currently on
the `feat/initial` branch). The pages most relevant here are
`docs/architecture/services.md`, `docs/architecture/isolation-and-security.md`,
`docs/architecture/workflow.md` and `docs/roadmap.md`.

## Scope

- **L1 (milestone M1, human-led):** in scope for this spec.
- **L2 (milestone M6, factory-built):** described as future work in [roadmap.md](roadmap.md).
- Anything beyond L2 is "after factory role complete".

## Reading order

| # | File | What it covers |
| --- | --- | --- |
| 1 | [overview.md](overview.md) | System context, goals, non-goals |
| 2 | [vocabulary.md](vocabulary.md) | Domain concepts, identifiers, error concepts |
| 3 | [responsibilities.md](responsibilities.md) | Component CRC cards (knows / does / collaborators) |
| 4 | [architecture.md](architecture.md) | Business logic vs interfaces vs infrastructure; state machine |
| 5 | [interfaces.md](interfaces.md) | Request/result contracts, outbound dependencies, VM boundary contract |
| 6 | [attempt-lifecycle.md](attempt-lifecycle.md) | Step-by-step attempt behaviour, push, teardown |
| 7 | [egress.md](egress.md) | Deny-by-default egress proxy and credential injection |
| 8 | [observability.md](observability.md) | Host-side and in-VM telemetry, correlation, retention |
| 9 | [security.md](security.md) | Threat model and mitigations |
| 10 | [edge-cases.md](edge-cases.md) | Non-standard flows and failure modes |
| 11 | [assertions.md](assertions.md) | Testable Given/When/Then behaviour |
| 12 | [testing.md](testing.md) | Test strategy including red-team, property, fuzz and mutation testing |
| 13 | [constraints.md](constraints.md) | Implementation rules (types, errors, boundaries, performance) |
| 14 | [operations.md](operations.md) | KVM hosts, configuration, deployment, runbooks |
| 15 | [assumptions.md](assumptions.md) | Challenged assumptions and resolutions |
| 16 | [tradeoffs.md](tradeoffs.md) | Alternatives considered (summary; detail in ADRs) |
| 17 | [roadmap.md](roadmap.md) | Increments, proposed work items, dependency order |

Machine-readable contracts live in [`../schemas/`](../schemas/). Decisions live in
[`../adr/`](../adr/).

## Markers used in this spec

- **[EXT]**: depends on a document outside this repository that still needs to be
  reconciled. These are the handbook pages and the provider-neutral runtime interface
  in cog-works (its ADR-0009).
- **[EXTRAP]**: an extrapolation beyond the service brief. Treat it as a proposal that
  reviewers should confirm.

## Workflow from here

1. **Human review** of this spec set and the ADRs. This is a T2+ repository, so a human approves every change.
2. **Interface design.** Turn [vocabulary.md](vocabulary.md),
   [responsibilities.md](responsibilities.md) and [interfaces.md](interfaces.md) into Rust
   crates, traits and types, and refine the JSON Schemas.
3. **Test design.** Turn [assertions.md](assertions.md) into tests, following the tiers in
   [testing.md](testing.md). The red-team suite is the M1 exit criterion.
4. **Planning.** Break the work items listed in [roadmap.md](roadmap.md) into GitHub issues
   once a human confirms them.
5. **Feedback loop.** Record gaps found during design or testing in
   [edge-cases.md](edge-cases.md), [assertions.md](assertions.md) or
   [vocabulary.md](vocabulary.md).
