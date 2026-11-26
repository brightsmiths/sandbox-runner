# AGENTS.md — sandbox-runner

Guidance for humans and coding agents working in this repository.

## What this service is

sandbox-runner runs **one stage attempt** of the Brightsmiths software factory inside a
fresh, short-lived, strongly isolated microVM. It collects the attempt's commits and
records, checks their scope, pushes them under a short-lived role token, and destroys
the VM. It is a **security-boundary component**. Start with
[`docs/spec/README.md`](docs/spec/README.md).

The system-level architecture lives in the handbook repository
[brightsmiths/handbook](https://github.com/brightsmiths/handbook) under `docs/`
(currently on the `feat/initial` branch).

## Repository layout

| Path | Contents |
| --- | --- |
| `docs/spec/` | Specification set (the source of truth for behaviour) |
| `docs/schemas/` | JSON Schemas (draft 2020-12) for the attempt request and result. This repository owns them. |
| `docs/adr/` | Architecture Decision Records (`ADR-NNNN-kebab-name.md`, see `ADR_TEMPLATE.md`) |
| `src/`, `crates/` | Rust implementation (not yet created) |

## Production standards

- **Treat this repo as tier T2+.** Every PR gets a human review, whoever wrote it.
- **Secrets never enter the VM.** That includes tokens, keys, credentials and telemetry credentials. If a change could
  put one there, stop and raise it. See [ADR-0004](docs/adr/ADR-0004-no-secrets-in-vm.md).
- **Never decide pass/fail or which stage runs.** gate-warden decides pass/fail and CogWorks decides which stage runs.
- **No state beyond an attempt.** Nothing is write-shared across attempts.
- **No cloud-specific dependencies.**
- **Error handling.** Return expected errors as `Result` values. Do not panic on input. Secret-bearing types must not
  implement `Debug` or `Display`.
- **Dependencies between layers.** Domain logic depends only on traits. Infrastructure adapters implement those traits.
- **Untrusted content.** Everything produced inside the VM is untrusted data: commits, records, logs and telemetry.
- **Scratch folders.** Never reference git-ignored scratch folders from committed files. Cite the handbook repository instead.

## Pre-implementation checklist

1. Read the relevant spec files and ADRs. Check [`docs/spec/constraints.md`](docs/spec/constraints.md).
2. Find the behavioural assertions (`docs/spec/assertions.md`) that the change must satisfy.
3. Write or extend tests first. Use unit, property, fuzz and mutation tests where
   [`docs/spec/testing.md`](docs/spec/testing.md) requires them.
4. Any security-relevant change must update `docs/spec/security.md` and add or extend a red-team test.
5. Any significant decision gets a new ADR. Do not edit an accepted ADR. Supersede it instead.

## Build and test

Rust stable, `cargo fmt`, `cargo clippy -- -D warnings`, `cargo test`. The integration and red-team suites need a
KVM-capable host (see `docs/spec/testing.md` and `docs/spec/operations.md`).
