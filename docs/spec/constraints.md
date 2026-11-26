# Implementation constraints

These are hard rules for the interface designer and implementers. Any deviation needs an
ADR.

## Language and type system

- **Language.** Rust (stable). Clippy runs with `-D warnings`. rustfmt is enforced.
- **Newtypes.** Use newtypes for every identifier and constrained value: `AttemptId`, `RunKey`, `CommitSha`,
  `ImageDigestRef`, `NormalisedPath`, `StageBranch`, `IdentityRole` and so on. Constructors validate,
  so a value that exists is valid.
- **Making invalid states unrepresentable:**
  - The request is split by mode: `CogworksRequest` has `StageBranch` and `OutputRecordMode`, and
    `GateWardenRequest` has neither.
  - `ScopeVerdict::InScope` is required to construct a `PublishPlan`.
- **Secrets:**
  - Secrets are held in `Secret<T>`-style wrappers that implement neither `Debug`, `Display` nor `Serialize`,
    and that zeroise on drop.
  - Secret types live only in infrastructure crates.
- **`unsafe`.** No `unsafe` in the domain crates. In provider glue, `unsafe` needs a `// SAFETY:` comment and
  a reviewer sign-off.
- **`unwrap`/`expect`.** Neither may be used on any data from callers, guests or the network.

## Module boundaries

- **Domain crates.** The domain crates (lifecycle, validation, policies) depend only on port traits and
  pure libraries. They never import an infrastructure crate, an HTTP client, the hypervisor SDK or a
  KMS SDK. CI checks this, for example with `cargo deny` bans or a dependency-graph test.
- **Where secrets and network access live.** The egress proxy is a separate binary. It is the only component
  that resolves egress credentials.
- **Mode logic.** The CLI and library front ends contain no lifecycle logic.

## Error handling

- **Expected failures** are values (`Result<T, E>`), and each one maps to a `ReasonCode`. Expected failures
  include invalid input, scope violations, budget kills and dependency outages.
- **Panics** are bugs. They are caught at the attempt boundary, converted to `failed`, and teardown still
  runs.
- **Error detail** is authored by the runner. Raw guest or caller content is included only truncated,
  escaped and in typed fields.

## Fail-closed rules

If any of these fails, nothing is published:

- the scope check;
- fetching the output schema;
- installing the allowlist;
- confirming that the VM has stopped before collection;
- verifying the bundle;
- comparing the manifest.

## Testing requirements

See [testing.md](testing.md). In summary:

- **Property tests** with proptest for the state machine, path normalisation, mount-plan resolution,
  allowlist composition and the telemetry sanitiser.
- **Fuzz targets** with cargo-fuzz for every parser of caller or guest bytes.
- **Mutation testing** with cargo-mutants on the domain crates. The domain crates need a score of ≥ 90 %. The
  security-critical modules need 100 %, or every survivor justified.
- **Coverage.** Domain crates need ≥ 90 % line coverage.
- **The red-team suite** must be green for M1 exit.

## Performance targets (initial, L1)

| Target | Value |
| --- | --- |
| VM start (create → entrypoint running), cold, reference host | < 10 s (p95) |
| Teardown (kill → confirmed) | < 5 s (p95) |
| Concurrent attempts per 16-core reference host | 4 |
| Wall-clock kill precision | ≤ 5 s after limit |
| Host-side validation + collection overhead (excluding checkout and push) | < 5 s (p95) **[EXTRAP]** |
| Telemetry relay overhead | No measurable guest slowdown below the rate limit **[EXTRAP]** |

These are initial targets. Revise them after M1 measurements. L2 pre-warmed images are
expected to reduce start time.

## Security rules

- No secret inside the VM, ever. That includes environment, mounts, image layers and telemetry config.
- Images are pinned by digest.
- Host git runs with hooks disabled, a safe configuration and no filters.
- There are no network listeners in L1 apart from the egress proxy's link to each VM. The control and tail
  interfaces are Unix sockets with restrictive permissions.
- Every PR gets a human review (T2+).

## Portability

- No cloud-provider SDKs.
- Secrets, identity, storage and compute are reached through ports: any OIDC provider, Vault or any KMS, and
  local storage.
