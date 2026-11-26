# Assumptions

| # | Assumption | Challenged because | Resolution | Impact / status |
| --- | --- | --- | --- | --- |
| S1 | microsandbox (libkrun) is mature enough for M1 | Young project; feature set (vsock, read-only rootfs, mount granularity, resource limits) must be verified | Provider trait isolates it; spike in first work item verifies required capabilities; Firecracker/Kata are L2 fallbacks | Open until spike; risk to M1 schedule |
| S2 | agent-gate's mount plan and scope check are callable synchronously from the runner | agent-gate interface not yet defined **[EXT]** | Ports `MountPlanSource`/`ScopeCheck`; adapter chosen once agent-gate defines its interface | Open |
| S3 | Credentials can be injected without TLS interception | Agents typically speak HTTPS | VM speaks plain HTTP over a private link to virtual hosts; proxy originates TLS (user-confirmed) | Resolved |
| S4 | Commits (not patches) are sufficient for every stage | Some tools emit patches | User decision: commits via git bundle only; patches unsupported in L1 | Resolved |
| S5 | The dependency proxy only needs crates in M1 | Future discipline packs need npm, PyPI, OCI | Crates-only for M1; allowlist model ecosystem-agnostic | Resolved |
| S6 | A library + CLI is sufficient for M1 callers | CogWorks' runtime interface may expect a service | Library + CLI for M1; revisit after cog-works ADR-0009 **[EXT]** | Resolved for M1 |
| S7 | GitHub-hosted CI can test everything | No KVM on hosted runners | Self-hosted KVM runner for integration/red-team | Resolved; needs a host |
| S8 | Performance: start < 10 s, teardown < 5 s, 4 concurrent / 16 cores | Not stated in the brief | Accepted as initial targets; re-measure after M1 | Resolved (provisional) |
| S9 | Telemetry can leave the VM without network or credentials | OTel SDKs usually export over network | Host-owned vsock/virtio-console relay | Resolved; depends on S1 capability check |
| S10 | "No state beyond the attempt" allows retaining logs | Logs survive teardown | Archives are evidence, not workflow state; never readable by later attempts | Resolved |
| S11 | The stage contract (declared extras, output record mode, record path) reaches the runner via the request | Source of stage contract is the harness repository via CogWorks **[EXT]** | Request carries the resolved values; runner validates extras against host config | Open: confirm with CogWorks spec |
| S12 | GitHub App per role exists and can mint repo-scoped, minutes-lived tokens | App layout is a system decision **[EXT]** | `TokenIssuer` port; role mapping in config | Open: confirm with handbook |
| S13 | At 10× load (40 concurrent) a single host is insufficient | — | L2 host pool; L1 returns retryable `provider_unavailable` | Accepted |
