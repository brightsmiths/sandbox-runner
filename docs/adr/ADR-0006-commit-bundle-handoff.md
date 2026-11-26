# ADR-0006: Commits handed off as a git bundle; patches not supported

- **Status:** Proposed
- **Date:** 2026-11-27
- **Deciders:** sandbox-runner maintainers (human review required, T2+)
- **Related:** [interfaces.md](../spec/interfaces.md), [attempt-lifecycle.md](../spec/attempt-lifecycle.md), [edge-cases.md](../spec/edge-cases.md) E3

## Context

The brief allows "the agent's commits or patch". The user prefers commits ("patches are
nasty"): patches lose authorship and metadata, apply fragilely, and make scope checks and
signing harder. The VM must not hold push credentials.

## Decision

- **Commits are the only supported handoff in L1.** When `task-marshal done` runs, it writes a git bundle
  (`/output/commits.bundle`) containing `checkout_sha..HEAD` of the work branch.
- **The runner verifies the bundle on the host** after the VM has stopped. It checks:
  - the prerequisite is `checkout_sha`;
  - the bundle has a single head;
  - the head descends from `checkout_sha`;
  - size and object caps;
  - no symlinks pointing outside the repository;
  - no new gitlinks.
- **The runner pushes on the host.** After the scope check passes, the runner pushes using the role token
  (ADR-0008). The push is never forced.
- **Patch files are ignored.** A future extension could add patch support behind its own ADR, but only if a
  concrete stage needs it.

## Alternatives considered

| Option | Pros | Cons |
| --- | --- | --- |
| Git bundle, host push | Preserves commits; verifiable ancestry; no creds in VM | Bundle parsing on host must be hardened |
| Patch files | Simple text | Lossy, brittle apply, metadata loss |
| Agent pushes with token | No host git | Violates ADR-0004 |
| Host reads `.git` from workspace directly | No bundle step | Guest-controlled repo config/hooks on host path; larger attack surface |

## Consequences

- task-marshal must implement bundle creation **[EXT: task-marshal]**.
- Host git runs with hooks disabled, a safe configuration and no filters.

## Compliance

Assertions 21, 22 and 29, the `fuzz_bundle_header` fuzz target, and the RT-FS tests.
