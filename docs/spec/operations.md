# Operations

## Host requirements (L1)

| Requirement | Detail |
| --- | --- |
| Virtualisation | x86_64 or aarch64 with KVM (`/dev/kvm`); bare metal or a cloud/on-prem instance with nested virtualisation |
| Kernel | Recent LTS kernel supported by libkrun/microsandbox **[EXTRAP: pin exact minimum after evaluation]**; cgroup v2 |
| Provider | microsandbox at a pinned version |
| Reference size | 16 cores, ≥ 64 GiB RAM, fast local SSD for workspaces; 4 concurrent attempts (see [constraints.md](constraints.md)) |
| Storage | Separate volumes or directories for: workspaces (wiped per attempt), images (read-only), telemetry archives (retained) |
| Network | Outbound TLS from host to GitHub, KMS, agent-gate, model gateway, dependency proxy, OTLP endpoint; VM interfaces firewalled to proxy only |
| Identity | Host workload identity via any OIDC provider, used to authenticate to Vault/KMS |

There are no cloud-specific dependencies. The same host profile runs on-prem or on cloud
bare-metal or nested-virtualisation instances.

## Processes

- **`sandbox-runner`**: the runner. It runs as an unprivileged service user. It needs access to
  `/dev/kvm` and to the provider, but not root, where the provider allows it.
- **`sandbox-egress-proxy`**: a separate process with its own network namespace. It is the only process
  that resolves egress credentials.
- **Reaper**: runs inside the runner, at startup and on a timer.

## Configuration

Configuration lives in files. Environment variables never carry secrets.

- **Host limits:** maximum vCPUs, memory and wall-clock per attempt, and maximum concurrency.
- **Branch rules:** the stage-branch pattern and the list of protected-branch patterns.
- **Role mapping:** role → GitHub App installation (a reference), uid, and permitted signing key (a reference).
- **Network profiles:** egress rules (see [egress.md](egress.md)). Each credential is a `credential_ref`.
- **Endpoints:** agent-gate, OTLP, the model gateway upstream and the dependency proxy upstream.
- **Telemetry limits:** caps, rate limits and the archive retention period.

All credential material is resolved at runtime from Vault or a KMS through the host workload
identity.

## Deployment

- **Install.** One runner and one proxy per KVM host. In L1, a single host is enough for M1.
- **Upgrades.** Drain the host (stop accepting new attempts and let running ones finish or time out), then
  upgrade, run the orphan sweep and resume.
- **Changes need review.** Because this is a T2+ repository, every change to the runner, the proxy or a
  network profile is reviewed by a human.

## Observability

All telemetry is specified in [observability.md](observability.md), which covers traces, metrics, logs, in-VM
relay, correlation, retention and live tailing. Operational alerts built on it:

| Alert | Condition |
| --- | --- |
| Teardown failure | Any `sandbox_runner.teardown.failures` increment (host quarantined) |
| Orphans found | `sandbox_runner.orphans.swept` > 0 outside startup |
| Egress deny spike | Deny rate above baseline for a stage |
| Scope rejections | Any `scope.rejections` (informational, security-relevant) |
| Token issuance failures | `token.issued{result=error}` > 0 |
| Telemetry drops | `telemetry.dropped{reason=export_backpressure}` sustained |
| Capacity | `attempts.active` at max for > N minutes |

## Runbooks

- **Orphaned VM or workspace.** Run `sandbox-runner reap --dry-run`, then `sandbox-runner reap`. Check
  that the result appears in `orphans.swept`.
- **Host quarantined after a teardown failure:**
  1. Inspect the provider state.
  2. Destroy any remaining resources by hand.
  3. Wipe the workspace volume.
  4. Clear the quarantine flag.
  5. Record the cause.
- **KMS or Vault outage.** Attempts fail with `token_unavailable` or `scope_check_unavailable`. The proxy's
  cached credentials expire with their TTL. No action is needed beyond restoring the KMS. Callers retry.
- **agent-gate outage.** Every attempt fails closed. Restore agent-gate. Do not bypass it.
- **Investigating an attempt.** Use `sandbox-runner tail <attempt_id>`, or query by `trace_id` or
  `attempt_id` in the OTel backend (see [observability.md](observability.md)).

## Scaling

- **L1:** a single host, with concurrency capped per host. A host at capacity returns
  `provider_unavailable` and the caller retries. The runner has no internal queue.
- **L2 (M6):** a pool of KVM-capable hosts, pre-warmed images and read-only dependency caches.
  Placement and pool management will be specified at L2 (see [roadmap.md](roadmap.md)).
