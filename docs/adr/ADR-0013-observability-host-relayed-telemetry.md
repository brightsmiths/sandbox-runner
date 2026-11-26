# ADR-0013: Observability via host-relayed in-VM telemetry and host-side OTel

- **Status:** Proposed
- **Date:** 2026-11-27
- **Deciders:** sandbox-runner maintainers (human review required, T2+)
- **Related:** [observability.md](../spec/observability.md), [security.md](../spec/security.md) T13–T16, [ADR-0004](ADR-0004-no-secrets-in-vm.md)

## Context

The user requires good observability of the runner and of the processes inside the
runner. The VM must hold no credentials, including telemetry credentials, and has no network path
except through the egress proxy. Everything produced inside the VM is untrusted.

## Decision

1. **Host-side OTel.** The runner and the proxy emit OTel traces, metrics and logs:
   - one root span per attempt, with a child span per lifecycle phase;
   - the metrics listed in observability.md;
   - structured logs.

   They export over OTLP from the host.
2. **In-VM telemetry is relayed, not exported.** The guest writes framed events to a host-owned channel,
   either a vsock port or a virtio-console:
   - stdout and stderr;
   - process events;
   - task-marshal and agent CLI events;
   - resource samples.

   The host TelemetryCollector receives them. The VM has no OTLP endpoint and no telemetry credentials.
3. **Untrusted handling.** The relay applies:
   - per-frame, per-attempt and rate limits;
   - UTF-8 and control-character sanitising;
   - redaction;
   - host-assigned source, correlation and severity ceiling;
   - stripping of reserved attributes;
   - nesting of guest events under an `in_vm` scope.
4. **Correlation.** `attempt_id`, `run_key`, `stage` and `trace_id` appear in every record, in the proxy's
   upstream headers, and in GitHub output records.
5. **Retention.** A bounded per-attempt log archive is written outside the workspace and survives teardown
   as evidence. It is referenced from the AttemptResult.
6. **Live tail.** `sandbox-runner tail` serves sanitised data over a local Unix socket.
7. **Telemetry is best effort.** It never affects the outcome of an attempt.

## Alternatives considered

| Option | Pros | Cons |
| --- | --- | --- |
| Host-relayed channel (chosen) | No creds/network in VM; live; host controls integrity | Needs guest framing (task-marshal/init) and provider vsock/console support |
| OTel SDK in VM exporting via proxy | Standard tooling | Telemetry endpoint becomes an egress target; guest can forge spans; credentials needed |
| Collect logs from `/output` after exit only | Simple | No live visibility; lost on kill; size unbounded until exit |
| Provider-level console capture only | No guest changes | No structure, no process/task-marshal events |

## Consequences

- task-marshal and the guest init must speak the framing protocol **[EXT: task-marshal]**.
- The frame parser and the sanitiser are fuzzed, and they are covered by 100 % mutation coverage for
  reserved-field handling.
- The archives are evidence, not workflow state. Shipping them to chain-chronicler's store is an L2 concern.

## Compliance

Assertions 36–44 and the RT-OBS tests.
