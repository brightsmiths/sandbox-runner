# Observability

Good observability is a first-class requirement for both **the runner** and **the
processes inside the VM**. Operators and reviewers must be able to answer three questions
for any attempt: what happened, where the time went, and why it ended. They must also be
able to watch a live attempt during M1. Throughout, the VM never holds telemetry credentials
and can never spoof host-side events. The decision is recorded in
[ADR-0013](../adr/ADR-0013-observability-host-relayed-telemetry.md).

Telemetry is **evidence, never workflow state**. GitHub remains the system of record.
Losing telemetry never changes an attempt's outcome.

## 1. Correlation

Every span, metric data point (as attributes, kept within cardinality limits), log line,
proxy request and GitHub record carries the CorrelationContext:

| Attribute | Source | Notes |
| --- | --- | --- |
| `brightsmiths.attempt_id` | runner | ULID; high cardinality: spans/logs only, not metric labels |
| `brightsmiths.run_key` | request | Also sent by the proxy to the model gateway |
| `brightsmiths.stage` | request | Metric label |
| `brightsmiths.caller` | request | Metric label |
| `brightsmiths.identity_role` | request | Metric label |
| `vcs.repository`, `vcs.sha` | request | Spans/logs |
| `trace_id` | runner | W3C trace context. If the caller passes a `traceparent`, the attempt span is its child |

- **Proxy.** The proxy adds `run_key`, `attempt_id` and `traceparent` to upstream requests, so that the
  model gateway's records can be joined with the runner's.
- **GitHub.** The output record (comment marker or committed file) includes `attempt_id` and
  `trace_id`.

## 2. Host-side runner telemetry

### Traces

There is one root span per attempt, `sandbox_runner.attempt`, with a child span for each
lifecycle phase:

```
sandbox_runner.attempt
├── validate          (schema, semantics, mount plan fetch, output schema fetch)
├── provision         (workspace, checkout, mounts, allowlist install, vm create)
├── run               (vm boot → exit; events: budget warnings, kill)
│   └── vm.boot
├── collect           (bundle verify, record validate, manifest compare/sign)
├── scope_check
├── publish           (token issue, push, record write)  [cogworks mode]
└── teardown          (kill, destroy, delete, allowlist remove, confirm)
```

- **Span status and outcome.** These are set only by the host, from domain outcomes. The root span records
  `outcome` and `reason.code`.
- **Calls to external dependencies.** Calls to agent-gate, the KMS, GitHub and the OTLP exporter are
  client spans. Span attributes never contain token values.

### Metrics (OTel, exported via OTLP)

| Metric | Type | Labels |
| --- | --- | --- |
| `sandbox_runner.attempts` | counter | outcome, reason, caller, stage, role, provider |
| `sandbox_runner.attempt.duration` | histogram | outcome, caller |
| `sandbox_runner.phase.duration` | histogram | phase, outcome |
| `sandbox_runner.vm.boot.duration` | histogram | provider |
| `sandbox_runner.vm.teardown.duration` | histogram | provider, confirmed |
| `sandbox_runner.resource.usage_ratio` | histogram | resource (cpu, memory, wall_clock, disk, output) |
| `sandbox_runner.budget.exceeded` | counter | resource |
| `sandbox_runner.egress.requests` | counter | decision (allow/deny), destination_kind, ecosystem |
| `sandbox_runner.egress.bytes` | counter | direction, destination_kind |
| `sandbox_runner.scope.rejections` | counter | role, stage |
| `sandbox_runner.token.issued` | counter | role, result |
| `sandbox_runner.manifest.signatures` | counter | result (signed, mismatch) |
| `sandbox_runner.orphans.swept` | counter | kind (vm, workspace, allowlist) |
| `sandbox_runner.teardown.failures` | counter | provider |
| `sandbox_runner.attempts.active` | gauge | provider |
| `sandbox_runner.host.capacity` | gauge | resource |
| `sandbox_runner.telemetry.dropped` | counter | source, reason (rate, size, export_backpressure) |

### Structured logs

- **Format.** Logs are JSON lines and use the OTel log data model. Every entry carries the
  CorrelationContext.
- **No secrets.** Secret-bearing types cannot be formatted, because they implement neither `Debug` nor
  `Display`. A redaction layer is the last line of defence: it masks known token shapes such as `ghs_…`,
  `Bearer …` and PEM blocks, plus values that the `SecretSource` registers at runtime. A test
  asserts that neither the logs nor the exported telemetry contain any secret seeded during a test run.
- **Untrusted content in host logs.** Content from the VM never appears in host log *messages*. It appears
  only in clearly typed fields such as `vm.stdout`, after escaping.

## 3. In-VM process observability

### Channel

In-VM telemetry leaves the VM **only** over a host-owned, data-only channel. The provider
exposes it as a vsock port or a virtio-console (for microsandbox: vsock when available,
otherwise the console stream **[EXTRAP: confirm the microsandbox capability]**).

- **No network path.** The VM has no OTLP endpoint and no telemetry credentials. Telemetry does not go
  through the egress proxy.
- **Frames.** The guest writes length-prefixed frames, each with these fields: `seq`, `ts_guest`, `kind`
  and `payload`. The host assigns `ts_host`, the correlation and `source = in_vm`.

### What is captured

| Kind | Producer | Content |
| --- | --- | --- |
| `stdout` / `stderr` | guest init / task-marshal | Agent CLI and child process output, line-framed |
| `process` | guest init (PID 1 supervisor) | Process start/exit: pid, ppid, argv[0] (truncated), exit code/signal |
| `task_marshal` | task-marshal | Stage events: started, tool call summaries, `done`, `dispute`, errors |
| `agent` | agent CLI (via task-marshal) | Turn boundaries, model call latency, tool invocations (names, not raw prompts) |
| `resource` | guest sampler (e.g. every 5 s) | CPU, RSS, disk usage of `/workspace`, `/tmp`, `/output` |

- **Host-side measurements.** The host also samples VM resource usage from the provider. Host samples are
  authoritative for budgets. Guest samples are only for diagnosis.
- **Exit status.** The exit code of the VM entrypoint comes from the provider (host side), not from
  the guest's frames.

### Untrusted-telemetry rules

Everything that arrives on the channel is untrusted data:

1. **Size limits.** A frame may be at most 64 KiB, and a single attempt may produce at most 256 MiB
   in total **[EXTRAP: defaults, configurable]**. Above the limits, frames are dropped and counted, and
   the attempt is marked `telemetry.truncated = true`.
2. **Rate limits.** A token bucket per attempt and per kind controls how much can be relayed.
   Back-pressure never blocks the VM indefinitely: excess frames are dropped.
3. **No spoofing.** The guest cannot set `source`, `severity` above `info`, `trace_id`, `span_id`,
   `outcome`, `attempt_id` or any `sandbox_runner.*` attribute. The host overwrites or strips these.
   In-VM events become span events or logs under a dedicated `in_vm` child scope, never host spans.
4. **No log injection.** Payloads are UTF-8-sanitised (invalid UTF-8 is replaced), control and ANSI
   escape characters are escaped, and newlines are framed rather than interpreted. Payloads are never
   used as format strings or as template input.
5. **Redaction.** The same redaction layer that protects host logs runs over in-VM content. This
   catches anything the agent echoes from upstream responses.
6. **Parsing.** The frame parser is fuzzed (see [testing.md](testing.md)). A malformed frame closes
   the channel for that attempt, and the attempt continues without in-VM telemetry: the event is
   counted and the attempt is marked truncated.

## 4. Retention and post-teardown handling

- **Export during the attempt.** In-VM telemetry is exported live to the OTel pipeline. It is also written
  to a bounded, per-attempt **log archive**, compressed JSON lines, *on host storage outside the
  workspace*.
- **Before teardown.** The collector flushes the archive. The archive is referenced from
  `AttemptResult.telemetry.log_archive_ref`. Teardown never deletes the archive. It deletes everything
  else.
- **Where archives go:**
  - **L1:** archives stay on the host's local telemetry directory, with a configurable retention
    (default 14 days **[EXTRAP]**). The OTel pipeline holds them for longer.
  - **Later:** shipping archives to the chain-chronicler attestation or telemetry store is a
    chain-chronicler L2 concern **[EXT]**.
- **Nothing in GitHub.** No in-VM logs are attached to GitHub records, because the records must stay small and
  free of untrusted text. The output record carries `trace_id` and `attempt_id` only.
- **Purge rules.** The archive is excluded from the "no state beyond the attempt" rule because it is
  evidence, not workflow state. No later attempt can read it, and it is never mounted into a VM.

## 5. Live tailing (M1)

- `sandbox-runner tail <attempt_id> [--kind stdout,stderr,task_marshal,process] [--follow]`
  streams the sanitised relay for a running attempt, or replays the archive for an attempt that has
  finished.
- **Access is local only.** Tailing is limited to the host's operator group through the runner's control
  socket with Unix-socket permissions. It offers no network listener in L1.
- **Same sanitised data.** Tailing shows exactly the sanitised and redacted data that goes to export, so it
  never shows a secret.

## 6. Failure behaviour

- **Telemetry is best effort.** An OTLP export failure leads to a bounded buffer, then drops with a counter.
  It never fails or delays an attempt.
- **Teardown confirmation logs synchronously.** Teardown confirmation and `teardown_failed` alerts are
  logged synchronously to local logs, in addition to OTLP.

## Acceptance criteria

- Each attempt produces exactly one root span with all phase child spans, including on failure paths.
- The `attempt_id`, `run_key` and `trace_id` of an attempt can be found in the runner logs, the proxy logs, the
  gateway request headers and the GitHub output record.
- A guest that emits forged `severity=error`, `outcome=published` or a foreign `trace_id` cannot change
  host-side records.
- A guest that floods the channel is throttled. The attempt still completes, and `telemetry.truncated`
  is set.
- The log archive remains available after teardown, and `tail` can replay it.
- A test-seeded secret never appears in any telemetry output.
