# Security

sandbox-runner is a security boundary. This file is its threat model. The system-wide
isolation model is defined in the handbook,
`docs/architecture/isolation-and-security.md` **[EXT]**. This file must stay consistent
with it.

## Assets

| Asset | Why it matters |
| --- | --- |
| GitHub App private keys / role tokens | Push access; forging records; impersonating roles |
| Oracle role signing key | Locking the oracle; a forged signature breaks "oracle never writable by its author" |
| Model gateway / registry credentials | Cost, data exfiltration, budget bypass |
| Host (hypervisor, kernel, other VMs) | Escape compromises every attempt |
| Repositories (default and protected branches) | Integrity of the product |
| Check verdicts (owned by gate-warden) | Forged checks bypass gates |
| Telemetry integrity | Audit evidence; misleading operators |

## Adversaries

1. **A malicious or confused agent** inside the VM. It has full control of the guest userland and
   possibly of the guest kernel.
2. **Poisoned content**: repository files, the context bundle or dependency responses that try to steer
   the agent or the runner.
3. **A compromised or substituted image.**
4. **Compromised caller input**: a malformed or malicious AttemptRequest.

The host operator and the KMS are trusted.

## Trust boundaries

```
VM (untrusted) ──[mounts, proxy link, telemetry channel, exit status]──► Host runner (trusted)
Host runner ──[TLS + workload identity]──► KMS, GitHub, agent-gate
Egress proxy ──[TLS + injected creds]──► model gateway, dependency proxy
```

## Threats and mitigations

| # | Threat | Mitigation | Residual risk | Red-team test |
| --- | --- | --- | --- | --- |
| T1 | **VM escape** via hypervisor/virtio bug | microVM (libkrun) hardware isolation; minimal device set; non-root guest; read-only rootfs; host runner runs unprivileged where the provider allows, with seccomp/cgroups around the VMM; patched host kernel; one attempt per VM | Hypervisor 0-day | RT-ESC-1..3 (privilege probes, device enumeration, known-CVE probes) |
| T2 | **Egress bypass**: direct IP, IPv6, DNS tunnelling, metadata endpoint, host services on bridge | No route except to proxy; no resolver; proxy-only namespace; no CONNECT; host firewall drops everything else from VM interfaces | Covert timing channels via allowed endpoints | RT-NET-1..6 |
| T3 | **Proxy confusion**: Host header / absolute-URI / path tricks to reach a non-allowlisted upstream or a different path | Proxy routes by virtual host only; upstream fixed per rule; path-prefix and method narrowing; request smuggling defences (strict HTTP/1.1 parsing) | Allowed upstream abuse (e.g. exfiltration via model prompts) is bounded by gateway policy | RT-NET-7..9 |
| T4 | **Token exposure**: role token or credentials leak into VM env, mounts, `.git/config`, logs, telemetry, results | Tokens fetched after VM stops; held only in host memory as non-printable types; checkout credentials stripped before mount; proxy strips auth headers both ways; redaction layer; seeded-secret tests | Host memory compromise | RT-SEC-1..4 |
| T5 | **Write outside allowed paths**: symlinks, hard links, `..`, case-folding, `.git` hooks/config, submodules | `absent` paths removed; ro bind mounts; scope pre-check on commits (fail closed); bundle verification rejects symlinks outside repo and new gitlinks; no hooks executed on host (host git runs with hooks disabled and safe config) | agent-gate scope logic bugs | RT-FS-1..6 |
| T6 | **Forge a check or verdict** | No GitHub credentials in VM; runner never creates check runs/statuses; role token permissions exclude `checks:write`/`statuses:write`; gate-warden uses its own App | None known | RT-GH-1..2 |
| T7 | **Forge output or dispute record** to mislead downstream | Record validated against pinned schema outside VM; published by the runner under the role identity with `attempt_id` marker; detail fields never echo raw agent text into runner-authored fields | Agent-authored content inside the record is, by design, untrusted downstream | RT-GH-3 |
| T8 | **Oracle manifest tampering**: get a manifest signed that does not match the tree | Runner recomputes digests from the bundle head and signs only on match; signing key only in KMS; only for `identity_role=oracle` | oracle-guard computation bugs | RT-ORC-1 |
| T9 | **Cross-attempt contamination** | Fresh VM, workspace and caches per attempt; no write-shared cache; L2 caches strictly read-only | Host page cache side channels | RT-ISO-1 |
| T10 | **Resource exhaustion / host DoS** | Host-enforced cgroup/VM limits; wall-clock watchdog; disk quotas; output size caps; per-host concurrency cap | Noisy-neighbour performance effects | RT-RES-1..3 |
| T11 | **Image substitution / supply chain** | Digest-pinned images only; optional signature verification of images **[EXTRAP]** | Compromised upstream build | RT-IMG-1 |
| T12 | **Malicious bundle/record parsing** on host (zip-bomb-like bundles, deep JSON, huge objects) | Size/object/depth caps; parsers fuzzed; no execution of guest content; host git invoked with safe config, no hooks, no filters | Parser 0-days | Fuzz targets (see testing.md) |
| T13 | **Telemetry spoofing**: guest forges host events, severities, outcomes or trace IDs | Host-assigned source/correlation; reserved attributes stripped; guest events nested under `in_vm` scope | None known | RT-OBS-1 |
| T14 | **Log injection / terminal escape** via guest output into operator logs or `tail` | Escaping of control/ANSI chars; framed newlines; content only in typed fields; never format strings | Operator tooling that re-interprets fields | RT-OBS-2 |
| T15 | **Telemetry flooding** to hide activity or exhaust host disk | Per-frame and per-attempt caps, rate limits, drops counted; attempt flagged truncated | Loss of diagnostic detail for that attempt | RT-OBS-3 |
| T16 | **Secret echo via telemetry** (agent prints upstream response containing credential) | Proxy strips credential headers in responses; redaction over in-VM stream; seeded-secret test | Novel secret formats | RT-OBS-4 |
| T17 | **Push to wrong branch** (default/protected, force push) | Branch pattern validation; no-force push; token scoped to repo; branch protection on default branch | Misconfigured branch protection | RT-GH-4 |
| T18 | **Prompt injection via repository content** | Out of scope for the runner's enforcement (CogWorks/harness concern); the runner's guarantees hold regardless of agent behaviour | Agent produces bad but in-scope work (caught by gates) | — |

## Security rules (summary)

- Nothing secret enters the VM. That includes environment variables, mounts, image layers and telemetry
  configuration.
- Fail closed: if the scope check, schema fetch or allowlist installation fails, the attempt is not
  published.
- The host never executes guest-produced content. Host git runs with
  `core.hooksPath=/dev/null`, `protocol.file.allow=never`, no filters and no fsmonitor.
- Every PR to this repository gets a human review (T2+).

## M1 exit criterion

The red-team suite in [testing.md](testing.md) must show that an agent **cannot**:

- write outside its paths;
- reach the internet;
- read secrets;
- forge a check.

It must also show that an agent cannot spoof host telemetry.
