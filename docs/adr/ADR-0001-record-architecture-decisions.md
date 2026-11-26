# ADR-0001: Record architecture decisions

- **Status:** Proposed
- **Date:** 2026-11-27
- **Deciders:** sandbox-runner maintainers (human review required, T2+)
- **Related:** [ADR_TEMPLATE.md](ADR_TEMPLATE.md), handbook `docs/adr/` **[EXT]**

## Context

sandbox-runner is a security-boundary service. Its decisions have to be reviewable and
traceable, and contributors must be able to find them, both humans and agents in later
milestones.

## Decision

We will record significant decisions as ADRs in `docs/adr/`, named
`ADR-NNNN-kebab-case.md` and following [ADR_TEMPLATE.md](ADR_TEMPLATE.md):

- A new ADR starts as **Proposed** and becomes **Accepted** after human review.
- An accepted ADR is never edited in substance. It is superseded by a new one instead.
- If the handbook has a template, this repository's template may be aligned with it **[EXT]**.

## Alternatives considered

| Option | Pros | Cons |
| --- | --- | --- |
| ADRs in repo | Versioned with code; reviewable in PRs | Needs discipline |
| Decisions only in handbook | Single place | Service-level detail clutters system docs |

## Consequences

- Decisions are discoverable next to the spec.
- Superseding requires a new file, never a silent edit.

## Compliance

PR review checks that significant changes reference or add an ADR.
