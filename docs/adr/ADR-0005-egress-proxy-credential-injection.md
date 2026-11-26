# ADR-0005: Egress via host-side proxy with credential injection and an extensible allowlist

- **Status:** Proposed
- **Date:** 2026-11-27
- **Deciders:** sandbox-runner maintainers (human review required, T2+)
- **Related:** [egress.md](../spec/egress.md), [security.md](../spec/security.md) T2, T3

## Context

Egress must be deny-by-default. Only the model gateway and the dependency proxy are allowed, plus any extras
declared in the stage contract. The proxy must inject the credentials. The dependency proxy serves crates
only in M1, but later discipline packs will need npm, PyPI, OCI and others.

## Decision

1. **The proxy runs on the host.** It is a separate process in its own network namespace, and it is the
   VM's only network peer.
2. **The VM speaks plain HTTP to the proxy** over a private link, addressing virtual hostnames. The proxy
   originates TLS to the real upstream and injects credentials from `SecretSource`. The VM has no MITM CA, no
   CONNECT tunnelling and no DNS resolver for external names.
3. **The allowlist is a list of `EgressRule`s.** Each rule has a `destination_kind`, an open `ecosystem`
   string for package registries, a virtual host, an upstream, optional method and path narrowing, a
   `credential_ref`, and limits. Adding a new ecosystem is a configuration change: a new rule template. It
   needs no code change in the domain or the proxy.
4. **Correlation headers.** The proxy adds the `run_key`, `attempt_id` and `traceparent` headers on
   upstream requests.

## Alternatives considered

| Option | Pros | Cons |
| --- | --- | --- |
| Plain HTTP to proxy, proxy-originated TLS | No credentials or CA in VM; full request visibility | Clients must be configured for virtual hosts |
| TLS-intercepting (MITM) proxy | Transparent to clients | CA trust in VM; complexity; still need injection |
| SNI/CONNECT allowlist | Simple | Cannot inject credentials; credentials would need to be in VM |
| In-VM firewall rules | Simple | Agent controls the VM |
| Closed enum of registries | Type-safe | Every new ecosystem needs a code change |

## Consequences

- Images, or discipline packs, must configure package managers and the agent CLI to use the virtual hosts.
- The proxy is security-critical code: it needs fuzzing and red-team coverage.

## Compliance

Assertions 13–18, the RT-NET tests, and the `fuzz_proxy_request` fuzz target.
