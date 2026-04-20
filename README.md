# Mothership

> **Status: V0 — early-stage. Design phase complete (2026-04-19). Phase II build just starting. Not production-ready. APIs and on-disk formats will change.**

Mothership is a secure runtime substrate for locally-deployed AI agents. It is not itself an agent product — it is the host that agents like Nous Research's Hermes, or next-generation agents, deploy onto to run in environments where existing consumer-grade agent runtimes aren't trusted.

Mothership exists because OpenClaw — the category-defining consumer local-agent runtime — accumulated 138 CVEs (7 Critical, 49 High), a ~37% malicious-skill rate in its marketplace, 40,000+ internet-exposed instances, and a 1.5M-token backend breach in under six months. Every one of those outcomes traces to a design decision made (or not made) before the runtime shipped. Mothership starts with the commitments.

## Design anchors

Before any code, read these:

1. **[`design-principles.md`](./design-principles.md)** — the 8 non-negotiable principles every spec and feature is checked against.
2. **[`use-cases.md`](./use-cases.md)** — who deploys Mothership and what they can't do without it (V1 beacon portfolio: UC1 solo founder, UC2 regulated professional, UC3 security-team-approved corporate).
3. **[`docs/superpowers/specs/2026-04-17-v1-spec-map-design.md`](./docs/superpowers/specs/2026-04-17-v1-spec-map-design.md)** — the V1 spec map: 7 sub-specs across 4 tiers, hybrid build phasing.
4. The four **T1+T2 sub-specs** (Phase I, complete):
   - [Secrets bridge](./docs/superpowers/specs/2026-04-18-secrets-bridge-design.md)
   - [Audit log](./docs/superpowers/specs/2026-04-19-audit-log-design.md)
   - [Plugin bundle + signing](./docs/superpowers/specs/2026-04-19-plugin-bundle-signing-design.md)
   - [Supervisor](./docs/superpowers/specs/2026-04-19-supervisor-design.md)

## Current phase

Per the V1 spec map's hybrid phasing:

| Phase | Status |
|-------|--------|
| I. Design T1 + T2 | ✅ **Complete** |
| II. Build T1 + T2 (Secrets bridge → Audit log + Plugin bundle → Supervisor) | 🚧 **In progress — this repo is being scaffolded** |
| III. Design T3 (Broker + provenance, Origin & operator-presence gateway) | Pending |
| IV. Build T3 | Pending |
| V. Design + build T4 (Skill process isolation) | Pending |

First-usable Mothership (capable of running a real agent with enforcement) lands at end of Phase IV.

## Stack

- **Primary language:** Rust (edition 2024). Chosen for the systems-level requirements of every T1+T2 sub-spec: OS keystore FFI (DPAPI / Keychain / libsecret), fsync-disciplined HMAC-chained logs, `SecretBytes` with deterministic zeroization, privileged-spawn helper as a TCB component, cross-platform native service lifecycle.
- **Workspace:** single Cargo workspace; per-sub-spec crates under `crates/`. Each crate lands with its full L1–L4 test suite per the owning spec's acceptance criteria.
- **Build tooling:** codegen crates for the secrets registry, audit-events registry, and manifest schema — source-of-truth YAML → Rust / TypeScript / Python types with shared golden-vector parity.
- **CI:** Linux / macOS / Windows matrix via GitHub Actions.
- **Minimum Rust:** 1.85.

## Build (preview — no crates yet)

```
cargo fmt --all -- --check
cargo clippy --all-targets --all-features
cargo test --all-features
```

The workspace currently has no crate members. The first crate (`mship-secrets`, the Secrets bridge) lands in the next PR.

## Platform scope for V1

- **macOS, Windows** — interactive desktop + system-service / launchd-daemon support (see Supervisor spec §Service lifecycle).
- **Linux** — V1 ships **interactive-desktop only** (systemd user unit under an operator session). Headless-Linux unattended-boot is V1.5+, blocked on a secrets-bridge backend that doesn't depend on session D-Bus (TPM-sealed / keyctl / systemd-credentials candidates).

## Contributing

This is an early-stage TamperTantrum Labs project. Planned as open source once the substrate is useful. For now, contributions are not being solicited — the design docs are the current surface for feedback.

Every PR on this repo requires:
- Verified signatures (GPG) on all commits.
- Approval via pull-request review (direct pushes to `main` are blocked).
- CI green across Linux / macOS / Windows.

## Related reading (external)

- [Hermes Agent](./research/hermes-agent.md) — Nous Research's reference architecture at the agent layer. Mothership hosts agents of this class.
- [OpenClaw research](../../Research/openclaw.md) *(workspace-root)* — the failure-case evidence the design principles are written against.

## License

Dual-licensed under either of:

- Apache License, Version 2.0 ([LICENSE-APACHE](./LICENSE-APACHE))
- MIT license ([LICENSE-MIT](./LICENSE-MIT))

at your option.
