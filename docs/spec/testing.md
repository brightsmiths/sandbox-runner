# Testing strategy

Testing follows the layering in [architecture.md](architecture.md). Domain logic is tested
without a hypervisor. Isolation and security are proven on real KVM hosts.

## Test tiers

| Tier | What | Where it runs | Gate |
| --- | --- | --- | --- |
| Unit | Domain logic with fake ports (fake provider, fake agent-gate, fake KMS, fake GitHub, fake clock) | GitHub-hosted runners | Every PR |
| Property | State machine, path normalisation, mount-plan resolution, allowlist composition | GitHub-hosted | Every PR |
| Contract | JSON Schemas vs golden examples; agent-gate / GitHub adapter contracts (recorded) | GitHub-hosted | Every PR |
| Fuzz (smoke) | Short fuzz runs on all targets (e.g. 60 s each) | GitHub-hosted | Every PR |
| Fuzz (campaign) | Long runs; corpus kept in repo | Scheduled (nightly) | Crashes become regression tests |
| Mutation | cargo-mutants over domain crates | GitHub-hosted | PRs touching domain crates (diff-scoped); full run nightly |
| Integration | Real microsandbox VMs, real proxy, local git server, fake GitHub API | Self-hosted KVM runner | Every PR touching provider/proxy/collector; nightly |
| Red-team | Adversarial guest images exercising [RT] assertions | Self-hosted KVM runner | **M1 exit criterion**; required on main and release |

GitHub-hosted runners lack KVM, so the integration and red-team suites run on a self-hosted
KVM-capable runner (see [operations.md](operations.md)). That runner holds no production
credentials. Test KMS, GitHub App and gateway credentials are canaries.

## Property-based testing (proptest)

Property tests apply wherever the input space is large and the invariants are clear:

- **Lifecycle state machine:**
  - For any sequence of step outcomes, the machine terminates in exactly one terminal outcome
    and then reaches `Destroyed`.
  - No sequence reaches `Publishing` when `caller = gate-warden`.
  - `Published` is reachable only through `ScopeChecked` with an `in_scope` verdict.
  - Each terminal outcome carries exactly one ReasonCode.
- **Path normalisation:**
  - For arbitrary byte strings, the normaliser either rejects the path or produces a path that contains no `..`,
    is not absolute and contains no NUL character.
  - Normalisation is idempotent.
  - The normalised path never escapes the workspace root.
- **Mount-plan resolution:**
  - The effective access of any path is never wider than the most specific entry that matches it.
  - Paths the plan does not match get the default.
  - Resolution is deterministic and does not depend on the order of the plan's entries, given the
    most-specific-wins rule.
- **Allowlist composition:**
  - The effective allowlist is a subset of the profile's rules plus the declared extras.
  - It never contains credential values.
- **Telemetry sanitiser:**
  - For arbitrary input, the output is valid UTF-8 and contains no raw control or ANSI characters.
  - Reserved attributes are always host-assigned.

## Fuzz testing (cargo-fuzz)

Every parser that reads caller-supplied or guest-supplied bytes has a fuzz target:

| Target | Input source | Invariant |
| --- | --- | --- |
| `fuzz_attempt_request` | Caller | No panic; either valid typed request or `invalid_request` |
| `fuzz_mount_plan` | agent-gate / inline | No panic; accepted plans satisfy normalisation properties |
| `fuzz_path_normalise` | Mount plan, bundle tree entries | No panic; never escapes root |
| `fuzz_output_record` | Guest (`record.json`, `dispute.json`) | No panic; bounded memory/depth |
| `fuzz_oracle_manifest` | Guest | No panic |
| `fuzz_bundle_header` / bundle verification | Guest (`commits.bundle`) | No panic; caps enforced; never accepts wrong ancestry |
| `fuzz_telemetry_frame` | Guest (vsock/console) | No panic; malformed frame closes channel; sanitiser invariants hold |
| `fuzz_proxy_request` | Guest HTTP | No panic; non-allowlisted destinations never forwarded |

The fuzz corpora are committed. Every crash becomes a unit regression test.

## Mutation testing (cargo-mutants)

- **Scope.** Run mutation testing on the domain crates: the state machine and result classification,
  request validation, mount-plan resolution, path normalisation, allowlist composition, bundle-acceptance
  rules, manifest comparison and the telemetry sanitiser.
- **What is excluded.** Thin infrastructure adapters are excluded. Integration tests cover them.
- **Expected score:**
  - The domain crates must reach **≥ 90 %** caught mutants.
  - The security-critical modules must reach **100 %**, or every survivor must be justified in a
    reviewed allowlist. These modules are path normalisation, mount-plan resolution, scope and fail-closed
    handling, branch validation, and the telemetry sanitiser's reserved-field handling.
- **How it runs.** On PRs it is diff-scoped (`--in-diff`). It runs in full nightly. A drop below the
  threshold fails the build.

## Coverage

- Domain crates must reach ≥ 90 % line coverage and ≥ 85 % branch coverage.
- Adapters must reach ≥ 70 % line coverage. The rest is covered by integration tests.

Coverage is a floor, not a goal. The mutation score is the main measure of test quality.

## Red-team suite (M1 exit criterion)

This is a set of adversarial guest images and scripts, each tied to [RT] assertions in
[assertions.md](assertions.md) and to threats in [security.md](security.md):

| Group | Probes |
| --- | --- |
| RT-FS (paths) | Write outside rw paths; symlink/hard-link/`..`/case tricks; modify locked tests; commit `.git/hooks`; submodule injection |
| RT-NET (egress) | Direct IPv4/IPv6, metadata IP, host bridge ports, DNS tunnelling, Host-header and absolute-URI confusion, request smuggling |
| RT-SEC (secrets) | Scan env, filesystem, mounts, `/proc`, `.git/config`, proxy responses for canaries |
| RT-GH (forgery) | Attempt to create checks/statuses; forge records; push to default branch |
| RT-ESC / RT-RES / RT-ISO / RT-IMG | Privilege probes, resource exhaustion, cross-attempt canaries, unpinned image |
| RT-ORC | Tampered oracle manifest |
| RT-OBS (telemetry) | Spoofed host fields, log injection, flooding, secret echo |

**M1 exits** when all of the following hold:

- every test in RT-FS, RT-NET, RT-SEC and RT-GH passes on the reference host;
- RT-OBS passes;
- the results are attached to the M1 milestone review.

## Test doubles

- **Fakes behind the ports.** Fakes are preferred over mocks. Each port has an in-memory fake with
  failure injection: timeouts, errors and partial success.
- **Recorded contract tests.** The GitHub and agent-gate adapters have recorded contract tests that are
  re-validated against real sandboxes nightly **[EXTRAP]**.
