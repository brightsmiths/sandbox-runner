# Roadmap

The system milestones are defined in the handbook (`docs/roadmap.md`) **[EXT]**.

## L1 (milestone M1, built human-led)

L1 is everything in this spec unless it is marked L2.

### Proposed work items

These have not been created as GitHub issues yet. Each needs human confirmation first.

| # | Work item | Depends on | Key assertions |
| --- | --- | --- | --- |
| 1 | **Spec set** (this folder, schemas, ADRs, AGENTS.md) | — | — |
| 2 | **Provider trait + microsandbox provider**, including capability spike (read-only rootfs, mounts, limits, vsock/console, network attachment) | 1 | 8, 9, 19, 20, 32 |
| 3 | **Domain core**: request validation, state machine, ReasonCodes, mount-plan resolution, path normalisation (with property, fuzz and mutation tests) | 1 | 1–7, 32 |
| 4 | **Mount-plan application** (consumes agent-gate output) and workspace preparation | 2, 3 | 11, 12 |
| 5 | **Egress proxy** with extensible allowlist (crates + model gateway) and credential injection | 1 | 13–18 |
| 6 | **Observability**: host OTel traces/metrics/logs, in-VM telemetry relay (vsock/console), sanitiser, log archive, `tail` | 2, 3 | 36–44 |
| 7 | **Result collection + scope pre-check + role-token push** (bundle verification, output record modes, manifest signing) | 3, 4 | 21–31 |
| 8 | **Teardown + Reaper + idempotency journal** | 2, 3 | 32–35 |
| 9 | **Red-team test suite** (M1 exit criterion): agent cannot write outside paths, reach internet, read secrets, forge a check; cannot spoof telemetry | 4–8 | All [RT] |

Dependency order within L1: **1 → (2, 3, 5 in parallel) → (4, 6, 8) → 7 → 9**.
The self-hosted KVM CI runner is a prerequisite for the integration parts of 2 and for 9.

## L2 (milestone M6, factory-built)

- A second provider: Firecracker or Kata Containers.
- Pre-warmed images, to reduce start latency.
- Read-only dependency caches. They are never write-shared across attempts.
- Capacity: a pool of KVM-capable hosts, with placement and pool management.
- Additional dependency ecosystems if discipline packs need them (npm, PyPI, OCI). Adding one is a
  configuration change ([egress.md](egress.md)).
- Telemetry archives shipped to the chain-chronicler L2 store **[EXT]**.

## Revisit points

- **Service mode.** Decide whether to add a long-running service mode after reading cog-works ADR-0009
  **[EXT]** ([ADR-0011](../adr/ADR-0011-library-and-cli-invocation.md)).
- **Queue events.** Decide whether to emit queue events as an advisory completion signal
  ([ADR-0012](../adr/ADR-0012-completion-via-github.md)).
- **Patch handoff.** Only if a concrete stage requires it ([ADR-0006](../adr/ADR-0006-commit-bundle-handoff.md)).

## After factory role complete

Anything beyond L2 belongs to this service's own roadmap.
