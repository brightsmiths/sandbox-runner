# Edge cases and failure modes

| # | Situation | Behaviour |
| --- | --- | --- |
| E1 | Agent produces no commits, `require_commits = true` | `rejected` / `no_commits`; output/dispute records still returned in result |
| E2 | Agent produces no commits, `require_commits = false` (e.g. review stage, gate-warden) | Proceed; nothing pushed; record published (cogworks) or returned (gate-warden) |
| E3 | Agent supplies a patch file instead of commits | Not supported in L1: ignored; if no bundle, E1 applies. Patch support is a possible future extension ([ADR-0006](../adr/ADR-0006-commit-bundle-handoff.md)) |
| E4 | Bundle contains commits not descending from `checkout_sha`, extra refs, or rewritten history | `rejected` / `bundle_invalid` |
| E5 | Bundle oversize or too many objects | `rejected` / `bundle_invalid` |
| E6 | Commits touch both allowed and disallowed paths | `rejected` / `scope_violation` listing violating paths; nothing pushed (all-or-nothing) |
| E7 | Commit introduces symlink pointing outside repo or a new submodule gitlink | `rejected` / `bundle_invalid` **[EXTRAP]** |
| E8 | Stage branch already exists and push is a fast-forward | Push normally |
| E9 | Stage branch exists and push is not a fast-forward | `failed` / `push_failed`; no force push; caller decides (e.g. new branch name) |
| E10 | Stage branch is default or protected | `invalid_request` at validation |
| E11 | `checkout_sha` not found | `failed` / `checkout_failed` |
| E12 | Valid dispute record, no output record | Outcome `rejected` / `output_invalid` with dispute attached and published |
| E13 | Output record and dispute both present and valid | Publish both; outcome follows normal path |
| E14 | Output schema unreachable | `invalid_request` (fail closed before VM) |
| E15 | Wall-clock budget hit during agent run | VM killed; `failed` / `budget_wall_clock`; any partial output ignored |
| E16 | Wall-clock budget hit during host-side collect/publish | Host-side phases have their own timeouts independent of the agent budget; budget applies to the VM run only |
| E17 | Role token expires mid-push | Retry once with a fresh token; then `failed` / `token_unavailable` |
| E18 | Runner crashes after push, before output record | Caller retries with same `idempotency_key`; Publisher detects existing branch head = bundle head and upserts the record (idempotent) |
| E19 | Runner crashes mid-run | Startup Reaper destroys orphan VM/workspace/allowlist; attempt reported `failed` on idempotent re-query **[EXTRAP: requires a small local attempt journal, deleted after result delivery]** |
| E20 | Duplicate concurrent request (same idempotency key) | Second call returns `in_progress` / first result |
| E21 | Teardown fails | `teardown_failed` annotation; host quarantined; alert |
| E22 | Context bundle over size cap or digest mismatch | `invalid_request` / `context_bundle_invalid` |
| E23 | agent-gate unreachable for scope check | `failed` / `scope_check_unavailable` (fail closed) |
| E24 | Repository uses LFS | L1: LFS objects not fetched; LFS pointers only **[EXTRAP]** |
| E25 | Oracle manifest requested but file missing | `rejected` / `manifest_mismatch` |
| E26 | `committed_file` mode but record file missing from commits or differs from validated record | `rejected` / `output_invalid` |
| E27 | Guest floods telemetry or sends malformed frames | Throttle/drop; close channel on malformed frame; `telemetry.truncated = true`; attempt continues |
| E28 | OTLP endpoint down | Bounded buffer then drop with counter; attempt unaffected; local archive still written |
| E29 | Host at capacity | `failed` / `provider_unavailable` (retryable); no queueing in L1 |
| E30 | Unknown `schema_version` | `invalid_request` |
