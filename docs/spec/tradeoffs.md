# Tradeoffs

This is a summary of the alternatives considered. Each decision is recorded in full in an ADR.

| Topic | Chosen | Rejected alternatives | Why | ADR |
| --- | --- | --- | --- | --- |
| Isolation unit | One fresh microVM per attempt | Containers; reused VMs | Kernel-level isolation; no cross-attempt state | [0002](../adr/ADR-0002-provider-trait.md), [0003](../adr/ADR-0003-one-fresh-vm-per-attempt.md) |
| Provider coupling | Trait with microsandbox reference | Hard-code microsandbox | L2 second provider without rewrite | [0002](../adr/ADR-0002-provider-trait.md) |
| Secrets | Host-only; none in VM | Short-lived tokens in VM | Any in-VM secret is exfiltratable by a hostile agent | [0004](../adr/ADR-0004-no-secrets-in-vm.md) |
| Egress | Host proxy, plain HTTP from VM, proxy-originated TLS | In-VM firewall; TLS MITM proxy; CONNECT allowlist by SNI | Agent controls the VM; MITM needs CA in VM; SNI allowlisting cannot inject credentials | [0005](../adr/ADR-0005-egress-proxy-credential-injection.md) |
| Egress extensibility | Open `ecosystem` field + rule templates | Closed enum of registries; per-ecosystem proxy code | npm/PyPI/OCI added by config, not code | [0005](../adr/ADR-0005-egress-proxy-credential-injection.md) |
| Result handoff | Git bundle of commits; runner pushes | Patches; agent pushes with a token | Patches lose metadata and are brittle to apply; agent pushing needs a token in VM | [0006](../adr/ADR-0006-commit-bundle-handoff.md) |
| Scope check failure | Fail closed | Fail open with warning | Security boundary | [0007](../adr/ADR-0007-scope-precheck-fail-closed.md) |
| Push credential | Per-role minutes-lived App token, fetched after VM stops | Long-lived PAT; one App for all roles | Least privilege; attribution per role | [0008](../adr/ADR-0008-per-role-short-lived-tokens.md) |
| Manifest signing | Host signs after recomputing digests | Sign in VM; sign without recompute | Key never in VM; agent cannot get arbitrary content signed | [0009](../adr/ADR-0009-oracle-manifest-signing-outside-vm.md) |
| Budgets | Host-enforced wall/CPU/memory; tokens/turns at gateway | Agent self-limits | Agent cannot be trusted to self-limit | [0010](../adr/ADR-0010-host-enforced-budgets.md) |
| Invocation | Library + CLI (M1) | gRPC/HTTP service now | M1 is CLI-driven; avoid premature network surface | [0011](../adr/ADR-0011-library-and-cli-invocation.md) |
| Completion signal | GitHub branch + output record; no queue events in L1 | Queue event as primary signal | GitHub is the system of record | [0012](../adr/ADR-0012-completion-via-github.md) |
| Observability | Host-relayed in-VM telemetry over vsock/console; OTel on host | OTel SDK in VM exporting over network; scrape logs after exit only | No credentials or network in VM; live visibility; anti-spoofing | [0013](../adr/ADR-0013-observability-host-relayed-telemetry.md) |
