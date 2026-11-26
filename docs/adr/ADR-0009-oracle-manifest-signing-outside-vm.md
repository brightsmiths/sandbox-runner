# ADR-0009: Oracle manifest signing outside the VM after recomputation

- **Status:** Proposed
- **Date:** 2026-11-27
- **Deciders:** sandbox-runner maintainers (human review required, T2+)
- **Related:** [attempt-lifecycle.md](../spec/attempt-lifecycle.md) §4, oracle-guard **[EXT]**

## Context

oracle-guard computes the oracle manifest inside the VM. The brief says signing, with the oracle role key,
happens outside the VM. If the runner signed whatever the VM handed over, the agent could get
arbitrary content signed.

## Decision

When `sign_oracle_manifest = true` and `identity_role = oracle`:

1. **Recompute.** The runner recomputes the manifest's file digests from the verified bundle head tree.
2. **Compare.** It compares them with `/output/oracle-manifest.json`.
3. **Sign only on a match.** It asks the KMS to sign the manifest digest with the oracle role key only if the
   two match. Otherwise the outcome is `rejected`/`manifest_mismatch`.

The signature format follows the Ed25519 signing used today, with a move to in-toto/DSSE through
chain-chronicler **[EXT]**.

## Alternatives considered

| Option | Pros | Cons |
| --- | --- | --- |
| Recompute then sign on host | Agent cannot launder content | Runner must know manifest format |
| Sign as delivered | Simple | Signs agent-controlled data |
| Sign inside VM | — | Violates ADR-0004 |

## Consequences

The runner depends on the manifest format that oracle-guard defines. That format must be versioned.

## Compliance

Assertion 26, the RT-ORC tests, and the `fuzz_oracle_manifest` fuzz target.
