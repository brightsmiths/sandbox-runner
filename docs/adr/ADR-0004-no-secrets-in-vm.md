# ADR-0004: Secrets never enter the VM

- **Status:** Proposed
- **Date:** 2026-11-27
- **Deciders:** sandbox-runner maintainers (human review required, T2+)
- **Related:** [security.md](../spec/security.md) T4, T16; [egress.md](../spec/egress.md)

## Context

The agent inside the VM must be assumed hostile, so any secret inside the VM can be exfiltrated.
The brief forbids secrets, tokens and keys of any kind inside the VM.

## Decision

We will keep every credential on the host:

- **What is covered.** GitHub tokens, signing keys, model-gateway and registry credentials, and telemetry
  credentials.
- **Where credentials live.** Credentials exist only in host memory, inside the `TokenIssuer`, `Signer` and
  `SecretSource` adapters and the egress proxy.
- **Secret types.** They are wrapped in types that implement neither `Debug`, `Display` nor `Serialize`,
  and they zeroise on drop.
- **When tokens are issued.** Role tokens are issued only after the VM has stopped.
- **Checkout credentials.** They are stripped from `.git/config` before the workspace is mounted.

## Alternatives considered

| Option | Pros | Cons |
| --- | --- | --- |
| Host-only secrets | No exfiltration path | Needs proxy injection and host-side push |
| Short-lived scoped tokens in VM | Simpler | Still exfiltratable within TTL; agent could push/forge |

## Consequences

- Egress needs credential injection (ADR-0005).
- Pushing must happen on the host (ADR-0006, ADR-0008).
- Telemetry must be relayed by the host (ADR-0013).

## Compliance

- Seeded-canary tests across the VM environment, filesystem, proxy responses and telemetry (RT-SEC, RT-OBS).
- Lints ban secret types in the domain crates.
