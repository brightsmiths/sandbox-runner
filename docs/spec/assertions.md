# Behavioural assertions

Each assertion is testable. The IDs are stable: do not renumber them. Append new ones.
Assertions tagged **[RT]** belong to the red-team suite, which is the M1 exit criterion.

## Request validation

1. **Unpinned image is refused**
   Given a request whose `image` has no `@sha256:` digest
   When the runner validates it
   Then the outcome is `invalid_request` with `image_not_pinned`
   And no VM, workspace or allowlist is created.

2. **Limits above host maximum are refused**
   Given `limits.memory_mib` above the configured maximum
   When validated
   Then the outcome is `invalid_request` / `request_invalid`.

3. **gate-warden requests cannot push**
   Given `caller = gate-warden` and a `stage_branch`
   When validated
   Then the outcome is `invalid_request`.

4. **Protected or default branch is refused**
   Given `stage_branch` equal to the default branch or a protected branch
   When validated
   Then the outcome is `invalid_request`.

5. **Unsafe mount plan is refused**
   Given a mount plan containing `..`, an absolute path, NUL, or a default of `read_write`
   When validated
   Then the outcome is `invalid_request` / `mount_plan_invalid`.

6. **Undeclared egress extra is refused**
   Given a `network_profile.extras` entry that the stage contract does not declare
   When validated
   Then the outcome is `invalid_request`.

7. **Idempotent requests**
   Given a completed attempt with idempotency key K
   When a request with key K is submitted again
   Then the original AttemptResult is returned
   And no new VM is created.

## Isolation and the VM

8. **Read-only root filesystem [RT]**
   Given a running attempt
   When a guest process writes anywhere under `/` outside the writable mounts
   Then the write fails.

9. **Non-root [RT]**
   Given a running attempt
   When the guest inspects its uid and tries to escalate privileges (sudo, setuid binaries, capabilities)
   Then the uid is non-zero
   And every escalation attempt fails.

10. **No secrets in the VM [RT]**
    Given a host whose KMS, proxy and GitHub App hold seeded canary secrets
    When the guest searches its environment, filesystem, mounts, `.git/config`, process
    arguments and proxy responses
    Then no canary value is found.

11. **Absent paths are absent [RT]**
    Given a mount plan marking `secrets/` as `absent`
    When the guest lists or opens `secrets/` by any path, including symlinks and case variants
    Then the path does not exist.

12. **Read-only paths are read-only [RT]**
    Given a mount plan marking `tests/locked/` as `read_only`
    When the guest writes, renames, deletes or hard-links within it
    Then each operation fails.

## Egress

13. **Deny by default [RT]**
    Given a running attempt
    When the guest connects to any public IP, an IPv6 address, `169.254.169.254`, or a
    host IP other than the proxy
    Then the connection fails.

14. **No DNS escape [RT]**
    Given a running attempt
    When the guest issues DNS queries for arbitrary names, including a tunnelling pattern
    Then no query leaves the host.

15. **Allowlisted model gateway works with injected credentials**
    Given a profile that allows the model gateway
    When the guest sends a plain-HTTP request to the gateway's virtual host
    Then the proxy forwards it over TLS with the injected credential and the `run_key`/`attempt_id` headers
    And the response the guest receives contains no credential.

16. **Non-allowlisted host denied [RT]**
    Given a profile without `declared_extra: lore-keeper`
    When the guest requests the lore-keeper virtual host
    Then the proxy denies the request
    And increments `egress.requests{decision=deny}`.

17. **Host-header confusion is ineffective [RT]**
    Given an allowed virtual host A
    When the guest sends a request whose Host header or absolute URI names an
    upstream outside the allowlist
    Then the proxy does not contact that upstream.

18. **Ecosystem is extensible by configuration**
    Given a configuration that adds a `package_registry` rule with `ecosystem = npm`
    When an attempt uses a profile containing it
    Then requests to the npm virtual host are allowed
    And no domain-crate code changed.

## Budgets

19. **Wall-clock kill**
    Given `wall_clock_seconds = N`
    When the guest runs longer than N
    Then the host kills the VM within N + 5 s
    And the outcome is `failed` / `budget_wall_clock`.

20. **Memory limit**
    Given `memory_mib = M`
    When the guest allocates beyond M
    Then the VM is killed or the guest process is OOM-killed by the host-set limit
    And the outcome is `failed` / `budget_memory` (or `agent_exit_nonzero` if the guest survives it).

## Collection and scope

21. **Commits are the handoff**
    Given a guest that committed on top of `checkout_sha` and ran `task-marshal done`
    When collected
    Then a CommitBundle with base `checkout_sha` is verified
    And a patch file in `/output`, if present, is ignored.

22. **Bundle ancestry enforced [RT]**
    Given a bundle whose head does not descend from `checkout_sha`, or that carries extra refs
    When collected
    Then the outcome is `rejected` / `bundle_invalid`.

23. **Output record validated outside the VM**
    Given `record.json` that violates the output schema
    When collected
    Then the outcome is `rejected` / `output_invalid`.

24. **Scope violation blocks push [RT]**
    Given commits that modify a path outside the role's read-write set
    When the scope check runs
    Then the outcome is `rejected` / `scope_violation` with the violating paths
    And nothing is pushed.

25. **Scope check fails closed**
    Given agent-gate is unreachable
    When the scope check runs
    Then the outcome is `failed` / `scope_check_unavailable`
    And nothing is pushed.

26. **Manifest signed only on match**
    Given `sign_oracle_manifest = true` and an in-VM manifest that differs from the recomputed one
    When collected
    Then nothing is signed
    And the outcome is `rejected` / `manifest_mismatch`.

## Publishing

27. **Role token never reaches the VM [RT]**
    Given a cogworks attempt that is published
    When the token is issued
    Then the VM has already stopped
    And the token value never appears in any log, telemetry export, result or workspace file.

28. **No check forging [RT]**
    Given any attempt
    When it completes
    Then no check run or commit status was created by the runner
    And the role token's permissions exclude `checks` and `statuses`.

29. **No force push**
    Given an existing stage branch whose head is not an ancestor of the bundle head
    When publishing
    Then the push is not forced
    And the outcome is `failed` / `push_failed`.

30. **Output record is idempotent**
    Given a published attempt whose output-record comment already exists
    When publishing is retried with the same idempotency key
    Then exactly one comment carrying the `attempt_id` marker exists.

31. **gate-warden mode publishes nothing**
    Given `caller = gate-warden`
    When the attempt completes
    Then the outcome is `completed`
    And no push, comment or check is made by the runner.

## Teardown and state

32. **Teardown on every path**
    Given any terminal outcome, including a panic in a phase
    When the attempt ends
    Then the VM, workspace, output area and allowlist are gone
    And `teardown.confirmed = true`.

33. **No cross-attempt state [RT]**
    Given attempt 1 writes a canary into every writable location
    When attempt 2 runs on the same host
    Then attempt 2 cannot observe the canary.

34. **Orphan sweep**
    Given a runner killed mid-attempt
    When the runner restarts
    Then orphaned sandboxes, workspaces and allowlists are destroyed
    And `orphans.swept` is incremented.

35. **Teardown failure quarantines the host**
    Given the provider fails to confirm destruction
    When teardown completes
    Then the host accepts no new attempts
    And an alert-level log is emitted.

## Observability

36. **One root span per attempt**
    Given any attempt, including `invalid_request`
    When it ends
    Then exactly one `sandbox_runner.attempt` span exists, with child spans for each phase entered
    And its `outcome` attribute equals the result outcome.

37. **Correlation across systems**
    Given a published attempt that called the model gateway
    When records are inspected
    Then the same `attempt_id` and `run_key` appear in runner logs, proxy logs, the gateway
    request headers and the GitHub output record.

38. **In-VM output relayed without credentials in the VM [RT]**
    Given a guest that writes to stdout/stderr
    When the attempt runs
    Then those lines appear in the OTel export and in `tail`, tagged `source = in_vm`
    And the VM holds no telemetry endpoint credential.

39. **Telemetry spoofing is ineffective [RT]**
    Given a guest that emits frames with `severity=error`, `outcome=published`, a foreign
    `trace_id` or `sandbox_runner.*` attributes
    When relayed
    Then the host-side records keep host-assigned values for those fields.

40. **Log injection neutralised [RT]**
    Given guest output containing ANSI escapes, CR/LF sequences forging log lines, and invalid UTF-8
    When relayed and tailed
    Then the escapes are escaped, no extra log records are created, and the output is valid UTF-8.

41. **Telemetry flood is bounded [RT]**
    Given a guest that emits more than the per-attempt telemetry cap
    When relayed
    Then excess frames are dropped and counted
    And `telemetry.truncated = true`
    And the attempt outcome is unaffected.

42. **Secrets never in telemetry [RT]**
    Given seeded canary secrets on the host and a guest that echoes every response it receives
    When all telemetry and logs are scanned
    Then no canary value is present.

43. **Log archive survives teardown**
    Given a finished attempt
    When teardown completes
    Then `telemetry.log_archive_ref` points to a readable archive
    And `tail <attempt_id>` replays it.

44. **Telemetry loss does not change outcomes**
    Given the OTLP endpoint is unavailable
    When an attempt runs
    Then the outcome is the same as with telemetry available
    And `telemetry.dropped{reason=export_backpressure}` is incremented.
