# ADR-0008: Per-role, minutes-lived GitHub App tokens from the KMS

- **Status:** Proposed
- **Date:** 2026-11-27
- **Deciders:** sandbox-runner maintainers (human review required, T2+)
- **Related:** [attempt-lifecycle.md](../spec/attempt-lifecycle.md) §6, [security.md](../spec/security.md) T4, T6, T17

## Context

The runner pushes on the agent's behalf. Identities are per role, for example oracle or builder, so that
the oracle is never writable by its author. The agent must never see the token.

## Decision

- **Who issues the token.** The `TokenIssuer`, backed by the KMS and a GitHub App per role, issues a
  minutes-lived installation token for the stage's role.
- **Token scope.** The token is limited to the single repository, with `contents:write` plus
  `issues:write`/`pull_requests:write` only when `comment` mode is used. It never has `checks`, `statuses`
  or `administration` permissions.
- **When it is fetched.** The token is fetched after the VM has stopped and after the scope check has
  passed, and it is zeroised right after publishing.
- **Retry on expiry.** If the token expires during a push, the runner fetches a fresh one once.

## Alternatives considered

| Option | Pros | Cons |
| --- | --- | --- |
| Per-role App, minutes-lived | Least privilege, per-role attribution | More App installations to manage **[EXT]** |
| Single App for all roles | Simpler | Breaks role separation |
| Long-lived PAT | Simplest | Unacceptable blast radius |

## Consequences

The role mapping lives in host configuration. The App layout must match the handbook's
identity model **[EXT]**.

## Compliance

Assertions 27 and 28, and the RT-GH tests.
