# Attempt lifecycle

This file describes the behaviour of each state in the state machine shown in
[architecture.md](architecture.md). Each phase opens its own child span (see
[observability.md](observability.md)).

## 1. Requested → Validated

1. **Assign an ID.** Assign an `attempt_id` (ULID) and set up the CorrelationContext.
2. **Check the idempotency key.** If an attempt with the same `idempotency_key` exists, return its result. If
   that attempt is still running, return `in_progress`.
3. **Validate the request.** Validate it against the schema, then apply the semantic rules in [interfaces.md](interfaces.md).
4. **Resolve the mount plan.** Fetch it from agent-gate or use the inline plan, then normalise it.
5. **Resolve the output schema.** Fetch it at its pinned revision.
6. **Compose the egress allowlist.** Build it from the network profile and the declared extras.

If any of these steps fails, the outcome is `invalid_request` and no VM is created.

## 2. Validated → Provisioned

1. **Create the workspace.** Create a fresh per-attempt workspace directory on host storage that no other
   attempt can see.
2. **Fetch the checkout on the host.** Fetch `repository@checkout_sha` on the host, using a read-only fetch
   credential. Strip the remotes and credential helpers from `.git/config`.
3. **Apply the effective mount plan:**
   - `absent` paths are removed from the presented tree. They are not merely hidden.
   - `read_only` paths are bind-mounted read-only.
   - `read_write` paths are writable and owned by the role's uid.
   - Ownership is set from the plan.
4. **Mount the context bundle.** After checking its digest and size, mount it read-only.
5. **Install the allowlist.** Install the per-attempt allowlist in the egress proxy.
6. **Create the VM.** Use the pinned image and the limits. The VM gets a read-only root filesystem, the
   non-root uid, a network attachment that reaches only the proxy, and the telemetry channel.

Errors here produce `failed` with reason `provider_unavailable` or `checkout_failed`.

## 3. Provisioned → Running → (exit)

- **Start the entrypoint** (task-marshal → agent CLI) and start the BudgetEnforcer's wall-clock timer.
- **Relay telemetry.** The TelemetryCollector relays stdout/stderr, process events and resource samples.
- **Enforce budgets.** When the wall clock runs out, kill the VM: outcome `failed`, reason `budget_wall_clock`.
  When the provider reports an out-of-memory or CPU limit kill, the reasons are `budget_memory` and
  `budget_cpu`.
- **Handle a normal exit.** On a normal exit, whatever the exit code, the VM stops and the attempt moves to
  collection. If the exit code is non-zero and there is no dispute record, the outcome is `failed` with
  `agent_exit_nonzero`. A dispute record is collected and published even when the exit code is non-zero.

## 4. Collected

The VM has **stopped** before any of the following steps runs.

1. **Read the output area** with size caps.
2. **Verify the commit bundle** (when `require_commits`, or when a bundle exists):
   - the bundle is well formed and within the size cap;
   - its prerequisite is `checkout_sha`, and the head descends from `checkout_sha`;
   - there are no refs other than the single work head;
   - the object count is bounded;
   - there are no submodule gitlinks that point to new repositories, and no symlinks that point outside
     the repository **[EXTRAP]**.

   If the bundle fails any of these checks, the attempt is `rejected` with `bundle_invalid`. If no commits
   were produced but they were required, it is `rejected` with `no_commits`.
3. **Validate the output record.** Validate `record.json` against the output schema. If it is invalid, the
   attempt is `rejected` with `output_invalid`. The exception is when a valid dispute record exists. In
   that case the dispute is published and the outcome is `rejected` with `output_invalid` and the
   dispute attached **[EXTRAP]**.
4. **Validate the dispute record**, if one was raised.
5. **Check the oracle manifest**, if signing was requested:
   - recompute the manifest digests from the bundle's head tree;
   - compare them with `oracle-manifest.json`;
   - if they differ, the attempt is `rejected` with `manifest_mismatch`;
   - if they match, sign with the oracle role key through the KMS.

## 5. ScopeChecked

- **Call the scope check.** Call agent-gate's scope check for `checkout_sha..head` with the role and the
  effective mount plan.
- **Handle a violation.** If the verdict is `out_of_scope`, the attempt is `rejected` with `scope_violation`,
  listing the violating paths. Nothing is pushed.
- **Fail closed.** If agent-gate cannot be reached, the attempt is `failed` with `scope_check_unavailable`.

## 6. Publishing (cogworks mode only)

1. **Get the role token.** Obtain it from the RoleTokenBroker.
2. **Push.** Push the bundle head to `stage_branch` with no force flag. If the branch already exists, the push
   must be a fast-forward (see [edge-cases.md](edge-cases.md)).
3. **Write the output record:**
   - `committed_file`: the record must already be in the pushed commits, at the path the stage contract
     specifies. The runner checks that it is there and that it equals the validated record.
   - `comment`: the runner upserts a comment on the target issue or PR that carries a hidden
     `attempt_id` marker. The record is signed as specified by the handbook's signed-comment convention
     **[EXT]**.
4. **Drop the token.** Drop and zeroise it.

Outcome: `published`. Errors produce `token_unavailable`, `push_failed` or
`record_publish_failed`. Every step is idempotent, so the caller can safely retry
`record_publish_failed` with the same `idempotency_key`.

In **gate-warden mode** this phase is skipped and the outcome is `completed`.

## 7. Destroyed

This phase runs on every path:

- **Telemetry export.** Flush the in-VM telemetry and write the bounded log archive (see
  [observability.md](observability.md)).
- **Kill and destroy.** Kill the VM if it is still running, then destroy it.
- **Delete storage.** Delete the workspace, the output area, any tmp and per-attempt caches, and the
  provider's per-VM state.
- **Remove the egress allowlist.** Remove the per-attempt allowlist from the proxy.
- **Confirm.** Check that all of the above is gone and set `teardown.confirmed`.
- **On failure:** if teardown cannot be confirmed, emit an alert, mark the host as quarantined so that it
  accepts no new attempts, and report `teardown_failed` as an annotation on the result.

**On startup**, the Reaper sweeps orphaned sandboxes, workspaces and proxy allowlists that
were left behind by a crashed runner process.

## Sequence (cogworks mode, happy path)

```mermaid
sequenceDiagram
    participant C as CogWorks
    participant R as sandbox-runner
    participant AG as agent-gate
    participant V as microVM
    participant P as egress proxy
    participant K as KMS
    participant G as GitHub
    C->>R: AttemptRequest
    R->>AG: get mount plan
    R->>R: validate, prepare workspace
    R->>P: install allowlist(attempt)
    R->>V: create + start (ro rootfs, non-root)
    V->>P: model / dependency calls (plain HTTP)
    P->>P: allowlist check, inject creds, TLS upstream
    V-->>R: telemetry frames (vsock)
    V->>V: task-marshal done → bundle + record
    V-->>R: exit
    R->>R: verify bundle, validate record
    R->>AG: scope check(base..head)
    R->>K: role token
    R->>G: push stage branch
    R->>G: output record (comment / verify committed file)
    R->>V: destroy; delete workspace; remove allowlist
    R-->>C: AttemptResult (published)
```
