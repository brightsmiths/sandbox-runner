# Interfaces

The contracts below are owned by this repository. The JSON Schemas in
[`../schemas/`](../schemas/) are normative. This file explains them.

## 1. Inbound: AttemptRequest

**Transport (L1):** a Rust library API plus a CLI (`sandbox-runner run --request <file|->`).
The CLI prints the AttemptResult as JSON on stdout and exits non-zero only when the
runner itself fails. A terminal outcome such as `rejected` exits zero.

We will decide whether to add a long-running service mode, and how it maps onto CogWorks'
provider-neutral runtime interface, after reviewing **[EXT: cog-works ADR-0009]**. See
[ADR-0011](../adr/ADR-0011-library-and-cli-invocation.md).

Schema: [`attempt-request.schema.json`](../schemas/attempt-request.schema.json).

| Field | Required | Notes |
| --- | --- | --- |
| `schema_version` | yes | `"1"` for L1 |
| `caller` | yes | `cogworks` \| `gate-warden` |
| `run_key` | yes | Budget and correlation key at model gateway |
| `stage` | yes | Opaque label |
| `idempotency_key` | yes | Duplicate requests with same key return the original result or `in_progress` |
| `image` | yes | OCI reference **pinned by digest** |
| `repository` | yes | `owner/name` on the configured GitHub host |
| `checkout_sha` | yes | 40-hex (or 64-hex) commit SHA |
| `context_bundle` | no | `{ digest, size_bytes, uri }`; mounted read-only; size ≤ cap |
| `mount_plan` | yes | Either `{ ref }` (fetch from agent-gate) or `{ inline: MountPlan }` |
| `network_profile` | yes | `{ name, version, extras[] }`; extras must be declared in stage contract |
| `limits` | yes | `vcpus`, `memory_mib`, `wall_clock_seconds`, optional `disk_mib`, `output_max_bytes` |
| `identity_role` | yes | e.g. `builder`, `oracle`; selects token, key, uid |
| `output_schema` | yes | `{ repository, path, revision }` or `{ digest }` |
| `output_record_mode` | yes for cogworks | `comment` \| `committed_file` (+ `path`); stage contract decides |
| `output_record_target` | if `comment` | Issue or PR number the comment goes on |
| `stage_branch` | yes for cogworks; forbidden for gate-warden | Must match stage-branch pattern; not protected/default |
| `sign_oracle_manifest` | no | Default false; only valid with `identity_role = oracle` |
| `require_commits` | no | Default true for cogworks, false for gate-warden |

**Semantic validation** (runs after the schema check, before provisioning):

- The image has a `@sha256:` digest.
- The limits are within the host maximums.
- If `caller = gate-warden`, the request has no `stage_branch` and no `output_record_mode`.
- If `caller = cogworks`, the request has a `stage_branch`, it matches the pattern, and it is not
  protected.
- `sign_oracle_manifest` is set only when `identity_role = oracle`.
- The mount plan's default access is `absent` or `read_only`. Its paths are normalised and contain no
  `..`, no absolute paths, no NUL characters and no `.git` write access.
- Every `extras` entry exists in the stage contract's declared extras **[EXTRAP: the stage contract
  comes from the harness repository via CogWorks]**.

## 2. Inbound: AttemptResult (returned to caller)

Schema: [`attempt-result.schema.json`](../schemas/attempt-result.schema.json).

| Field | Notes |
| --- | --- |
| `attempt_id` | ULID |
| `correlation` | `run_key`, `stage`, `caller`, `trace_id` |
| `outcome` | `published` \| `completed` \| `rejected` \| `failed` \| `invalid_request` |
| `reason` | `{ code, detail }`; `detail` is runner-authored, never raw agent text |
| `commits` | `{ base_sha, head_sha, count }` from the bundle (if any) |
| `push` | `{ branch, head_sha }` when published |
| `output_record` | The validated record (inline) and, if published, its GitHub location |
| `dispute_record` | Validated dispute record if raised |
| `scope` | `{ verdict, violations[] }` |
| `oracle_manifest` | `{ digest, signature_ref }` when signed |
| `usage` | Peak memory, CPU seconds, wall-clock, vs limits |
| `timings` | Per-phase durations |
| `telemetry` | `{ trace_id, log_archive_ref, truncated }` |
| `teardown` | `{ confirmed: bool }` |

## 3. Outbound contracts

| Dependency | Call | Contract | Failure handling |
| --- | --- | --- | --- |
| agent-gate | Get mount plan | (repo, sha, role, stage) → MountPlan | `mount_plan_invalid` / retry once then `failed` |
| agent-gate | Scope check | (repo, base..head, role, plan) → ScopeVerdict | **Fail closed**: `scope_check_unavailable` |
| KMS + GitHub App | Issue role token | (role, repository) → token (minutes TTL, contents:write [+ issues/pull_requests:write when `comment` mode]) | `token_unavailable` |
| KMS | Sign | (role=oracle, digest) → signature | `failed` |
| GitHub | Push | refs → stage branch, no force | `push_failed` |
| GitHub | Comment | idempotent upsert keyed by `attempt_id` marker | `record_publish_failed` |
| Model gateway | (via proxy) | Proxy adds credentials + `x-brightsmiths-run-key`, `x-brightsmiths-attempt-id` headers **[EXTRAP: header names]** | n/a (agent-visible HTTP errors) |
| OTel pipeline | OTLP export | Best-effort; never blocks the attempt | Buffer, then drop with counter |

The exact agent-gate interface (whether it is a library, a CLI or a service) is **[EXT]**: it is defined in
the agent-gate repository.

## 4. VM boundary contract (runner ↔ image)

The contract between the runner and any image it can run:

- **Filesystem layout inside the VM** **[EXTRAP: paths]**:
  - `/` is read-only.
  - `/workspace` is the checkout, with access per the mount plan. `.git` is writable only so that the
    agent can commit locally.
  - `/context` holds the context bundle, read-only.
  - `/output` is the only area that is always writable. It receives `commits.bundle`, `record.json`,
    `dispute.json` and `oracle-manifest.json`.
  - `/tmp` is a size-capped tmpfs.
- **User:** a non-root uid chosen per identity role. The VM has no sudo and no setuid binaries.
- **Entrypoint:** the image's entrypoint runs task-marshal, which runs the agent CLI. task-marshal is
  the agent's only interface to its stage. When `task-marshal done` runs, it writes `record.json` and
  produces `commits.bundle` from `checkout_sha..HEAD`.
- **Environment:** no secret values. The environment holds only `HTTP_PROXY`/`HTTPS_PROXY`-style settings
  that point at the proxy over the private link, the non-secret correlation IDs, and the model gateway's
  base URL as seen through the proxy.
- **Telemetry channel:** a vsock port or virtio-console that carries framed events (see
  [observability.md](observability.md)). It is write-only from the VM's point of view.
- **Handoff:** commits only, as a git bundle. **Patches are not supported in L1**
  ([ADR-0006](../adr/ADR-0006-commit-bundle-handoff.md)).

## 5. Versioning

- The schemas carry `schema_version`. Additive optional fields are allowed within a major version. Any
  breaking change needs a new major version, an ADR, and coordinated updates in CogWorks and gate-warden.
- The runner rejects any `schema_version` it does not know with `request_invalid`.

## Acceptance criteria

- Golden example requests and results for each caller mode validate against the schemas.
- Every non-success path produces exactly one `ReasonCode` from [vocabulary.md](vocabulary.md).
- No field in either schema can carry a secret value. Credentials appear only as references.
