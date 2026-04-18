# Secrets bridge — sub-spec design

**Status:** V0 design — committed 2026-04-18.
**Authored:** 2026-04-18 (Gavin + Cowork).
**Supersedes:** the secrets portion of `docs/brainstorm/early-spec.md` §"Lifecycle and secret bootstrap" (principle-wins: `design-principles.md` P5 overrides the age-encrypted-blob approach).

## Citations

- **Map:** [`docs/superpowers/specs/2026-04-17-v1-spec-map-design.md`](./2026-04-17-v1-spec-map-design.md)
- **Principles load-bearing:** P5 (primary); also consumed by P2 (supervisor identity key), P6 (operator-presence gating of privileged ops), P8 (audit HMAC key)
- **Use cases cited:** UC2 (primary — regulated-professional credentials), UC1, UC3
- **Tier:** T1 (no substrate dependencies)
- **Upstream:** OS keystore APIs only
- **Downstream consumers:** Supervisor (T2), Broker + provenance (T3), Audit log (T1), Origin & operator-presence gateway (T3)

## Purpose

Provide every other Mothership sub-spec with the only legitimate path to read, enroll, export, import, and delete secret material (API keys, OAuth tokens, service credentials, HMAC keys). Enforce P5's invariant: **no plaintext secret exists outside the OS keystore's internal state except during the scope of a single resolve call**.

The bridge is the smallest and purest sub-spec in V1. Its correctness is load-bearing for every other sub-spec — a leak here defeats P5 regardless of what the supervisor, broker, or audit log do.

---

## Scope

### In V1

- **Backends:** DPAPI (Windows), Keychain (macOS), libsecret / GNOME Keyring via D-Bus (Linux desktop with session bus)
- **Operations:** `enroll`, `resolve`, `export`, `import`, `delete`
- **Surface:** CLI only
- **Type-level controls** on the `SecretBytes` and `SecretHandle` types

### Out of V1 (explicit non-goals)

- **Headless Linux without libsecret** — tracked as a pre-scoped V1.5 deliverable; candidate strategies (TPM-backed file backend, keyctl with TPM-sealed persistence, emerging systemd-credentials pattern) to be evaluated then.
- **File-based fallback of any kind.** Shipping a weaker backend as a blessed option poisons the trust story P5 exists to protect. No fallback in V1.
- **Rotation (`mothership secret rotate`).** Deferred to V1.5 — no specific V1 UC drives it, and rotation is a supervisor-layer orchestration concern that re-enrolls new handles and deprecates old ones.
- **Web-UI paths** for any operation. All operations are CLI-invoked in V1; web UI is a later add if needed.

---

## Handle model

### Shape

A `SecretHandle` is a **namespaced string** that IS the keystore key (prefixed and scoped for Mothership).

Examples:

- `mothership/supervisor/service-key`
- `mothership/audit/hmac`
- `mothership/plugin/<publisher-id>/signing-key`
- `mothership/agent/<agent-id>/hmac`
- `mothership/org/<org-context>/memory-subkey`

### Typo safety via generated constants

A single source file `secrets-registry.yaml` declares every handle that the substrate may use. A build-time codegen step emits:

- Rust constants (used by the bridge, supervisor, broker, audit log)
- TypeScript constants (used by CLI / future web UI)
- Python constants (used by test fixtures and ops tooling)

Typos fail at compile time in every language. Adding a new handle category is a one-line change to the YAML plus a rebuild.

### Persistence

Handles are stable across process restarts. The handle IS the keystore key; no in-memory mapping table exists. Re-enrollment is only required if the OS keystore itself is reset (reinstall, user-profile rebuild, OS reimage).

### Out of scope for the handle

- **Rotation policy.** Supervisor-layer concern (see "Out of V1" above).
- **Access scoping** (e.g., "handle X is resolvable only by agent Y"). Enforced by the Supervisor's grant ledger, NOT by the handle itself. The bridge is a passive resolver; authorization happens at the caller.

---

## Operations (V1)

| Op | Description | Re-auth? | Audit event |
|----|-------------|----------|-------------|
| `enroll` | Store a secret under a handle. Fails if handle already exists (no silent overwrite). | Yes | Yes |
| `resolve` | Retrieve a secret; returns `SecretBytes`. Hot path. | No | No (caller's grant decision is logged by Supervisor) |
| `export` | Produce an age-encrypted archive of all Mothership-scoped secrets. | Yes | Yes |
| `import` | Restore from an age-encrypted archive; emits audit-chain-restart marker. | Yes | Yes |
| `delete` | Remove a handle and its secret. | Yes | Yes |

Re-auth uses sub-spec 7 (Origin & operator-presence gateway) primitives — Touch ID, Windows Hello, YubiKey, or PAM-style challenge depending on OS. The bridge calls into sub-spec 7; it does not re-implement operator-presence itself.

`resolve` is deliberately not audit-logged at the bridge layer. It's the hot path and is called from every agent / broker hop; logging per-resolve would flood the audit log. The Supervisor's grant-check is the correct logging point — it records "agent X was granted the ability to resolve handle Y." The bridge executes the resolve after the grant check passes.

---

## Surfaces & auth

- **CLI entry point:** `mothership secret <enroll|resolve-probe|export|import|delete>`
  - `resolve-probe` is an operator-only diagnostic that verifies a handle is resolvable but does NOT print the secret value — ever. It prints `OK (<handle-name>) [N bytes]` or `ERR: <reason>`.
- **No network surface.** The bridge has no listener, no WebSocket, no HTTP endpoint. CLI → in-process API → OS keystore API.
- **Re-auth gate.** Every privileged op (`enroll`, `export`, `import`, `delete`) prompts the operator via sub-spec 7 before the op executes. On failure, the op aborts before touching keystore state.

---

## Export / import

### Export format

- **Encryption:** age to an operator-supplied recipient public key.
- **CLI:** `mothership secret export --out <path> --recipient age1...`
- **No Mothership-managed export key.** Operator owns the recipient key; losing it = losing the backup. This is the correct responsibility model.
- **Contents:** every Mothership-scoped keystore entry, plus metadata (handle names, secret-type tags, enrollment timestamps). NOT operator OS credentials or unrelated keystore entries.

### Import

- **CLI:** `mothership secret import --in <path>`
- Prompts for the operator's age private key.
- Default: **refuses to overwrite** an existing handle. Explicit `--force` flag required for overwrite; every forced overwrite is its own audit event.
- On successful import, emits an `audit-chain-restart` marker in the audit log (see below).

### Audit HMAC key handling (the awkward case)

The audit log's per-entry HMAC chaining key lives in the secrets bridge. This creates two paths for export/import:

| Option | Behavior | Trade-off |
|--------|----------|-----------|
| Include the audit HMAC key in export | Local audit chain can be re-verified after restore, up to the restart marker | Export archive now contains tamper-key material; loss of the archive is more sensitive |
| Exclude it; generate fresh on import | Export archive is slightly less sensitive | Pre-restore audit entries become locally unverifiable |

**Chosen:** include the audit HMAC key in export. Rationale:

- UC2 regulated-pro compliance cases often need post-restore audit verification.
- The out-of-band external mirror (sub-spec 5) is the authoritative cross-restart record anyway. Local HMAC chain is the local-tamper detection mechanism; a clean restore is not local tampering.
- The `audit-chain-restart` marker makes chain discontinuity explicit, not silent. `audit verify` is taught to expect markers and verify each segment independently.

### Audit event shapes

All events include: timestamp, operator identity, operator-presence re-auth method + result.

- **enroll:** handle name, secret-type tag. Value NEVER present.
- **export:** recipient-key fingerprint (of the age public key), list of handle names (not values), destination file path SHA-256.
- **import:** source file path SHA-256, list of handle names imported, `audit-chain-restart` marker emitted on the audit log.
- **delete:** handle name, optional operator-supplied reason string.

---

## Type-level controls

### `SecretBytes`

The only public shape for a revealed secret. Properties:

- `Drop` zeroizes the underlying allocation via the `zeroize` crate with a `compiler_fence` to prevent the compiler eliding the write.
- No `Display` or `Debug` impls — or both print the literal `"[REDACTED]"`. No exceptions.
- **No** `Serialize` / `Deserialize` impl. Serialization attempts fail to compile.
- `.reveal(&self) -> &[u8]` — returns a short-lived borrow. Never clones.
- `.reveal_as_str(&self) -> &str` helper for UTF-8 secrets (tokens, etc.); same borrow semantics.

### `SecretHandle`

Newtype over `&'static str` constrained to the generated-constant set. Constructors are private; only the generated constants can instantiate a handle. Typos cannot compile.

---

## Service identity per OS

Per P5 compliance check: "scoped to the Mothership service identity."

| OS | Scoping mechanism |
|----|------|
| Windows | DPAPI key tied to the user profile Mothership runs under. `CRYPTPROTECT_LOCAL_MACHINE` flag is an operator-selectable option for machine-bound (vs. user-bound) storage. |
| macOS | Keychain access group tied to Mothership's code-signing ID. Unlocked keychain required at runtime; locked keychain surfaces a hard error (no silent fallback). |
| Linux | libsecret collection labeled `mothership`, paths under `/com/mothership/...`. Session D-Bus required; absence surfaces the documented V1.5-gap error on startup. |

---

## Test strategy (4 levels)

### L1 — Type-level controls (design, verified by compilation)

Covered above. The `SecretBytes` and `SecretHandle` types make most accidental leakage a compile error.

### L2 — Unit tests

- Resolve → use → drop → assert zeroized (via allocator instrumentation; see L4).
- `enroll` rejects duplicate handle without `--force`.
- `enroll` / `export` / `import` / `delete` each emit their declared audit event shape.
- Logger mock fails the test if the canary secret bytes ever reach a log sink.
- Import with mismatched recipient key fails with a specific error, not a silent success.

### L3 — Integration tests under sanitizers

- Bridge test suite runs under ASan on nightly Rust. Catches use-after-free / buffer overruns touching secret memory.
- Critical paths (resolve, drop, export encryption) run under `miri` to catch undefined behavior in `unsafe` code.
- MSan skipped — not reliably supported for Rust as of 2026-04.

### L4 — Acceptance test (CI gate)

Deterministic end-to-end canary test; failure blocks merge.

1. Enroll canary `s = "mothership-canary-<256-bit-random-from-test-seed>"`.
2. Resolve via the full call path (CLI → bridge → OS keystore (test-mode backend)).
3. Use the `SecretBytes` in a representative downstream op, then drop.
4. **Primary check** — allocator instrumentation: every freed region that touched the secret is zero at free-time.
5. **Secondary check (belt-and-suspenders)** — heap scan (`/proc/self/mem` on Linux, `mach_vm_read` on macOS, `ReadProcessMemory` on Windows): no `s` bytes found in the process heap.
6. Log-sink scan: no configured log sink contains `s`.
7. Crash-dump simulation: write a minidump; scan; assert `s` not found.
8. Export round-trip: operator receives `s` correctly after `import`; the export's audit event contains the handle name but not `s`.

If the primary check and either the secondary, log, crash-dump, or export-audit check disagree, the test fails and surfaces both results for triage.

### Scope boundary of these tests

This sub-spec's tests verify only what the **bridge** exposes. Downstream leakage (a caller of `resolve` logging the revealed bytes, a broker passing them to a web-content-provenance tool call) is sub-spec 2's (Broker + provenance) concern and is tested there. **Keystore on-disk storage is not a leak** for the purposes of this test — that is P5-mandated storage.

---

## Error modes

All errors surface as hard returns, not silent failures. The bridge never degrades silently.

- **Keystore service unavailable** (service not running, D-Bus absent on Linux, locked keychain on macOS) — hard error. Supervisor cold-boot halts with an actionable diagnostic.
- **Handle not found** — typed `NotFound` error returned to caller; caller decides (supervisor may halt; broker may deny the capability).
- **Keystore returns corrupted / truncated bytes** — hard error plus a `secret-corruption-detected` audit event with the handle name and corruption signature.
- **Concurrent enrollment collision** — second writer fails with `AlreadyExists`. No silent overwrite.
- **Permission denied** by the keystore — hard error at startup with platform-specific diagnostic (`keychain access denied for code-signing ID X`, `DPAPI failed for user profile Y`, `libsecret returned AccessDenied for collection mothership`).

---

## Interface produced (consumed by downstream sub-specs)

- `SecretHandle` — typed newtype; construction restricted to generated constants.
- `SecretBytes` — zeroizing newtype; no serialization; `.reveal()` borrow only.
- `SecretsClient` trait with `enroll`, `resolve`, `export`, `import`, `delete`.
- CLI: `mothership secret <subcommand>`.
- Generated constants files for Rust / TypeScript / Python.

---

## Principle compliance notes

| Principle | Compliance check | Result |
|-----------|-----------------|--------|
| P5 | Does any secret exist outside the OS keystore at any point in its lifecycle, including logs, crash dumps, config exports, or memory-resident past the lifetime of a single API call? | **No.** Type-level controls, 4-level test strategy, and scope boundary ensure it. Keystore-internal state is P5-mandated storage, not a leak. |
| P2 | Can the agent process reconfigure or bypass the bridge? | **No.** The bridge runs in the supervisor's process; agents call it only via the broker-mediated RPC. Agent has no direct handle. |
| P6 | Do privileged bridge ops require operator-presence re-auth? | **Yes.** `enroll`, `export`, `import`, `delete` all route through sub-spec 7 re-auth. `resolve` is not re-auth-gated per the hot-path exception but is authorized by the Supervisor's grant ledger. |
| P8 | Do privileged bridge ops produce audit entries? | **Yes.** Every privileged op emits a declared audit event shape. `resolve` is logged at the Supervisor's grant-check layer, not here. |

No principle tensions identified.

---

## Open implementation-level questions (for writing-plans, not design)

1. **OS keystore crate choice.** `keyring-rs` provides a unified cross-platform API but adds a dependency and hides platform-specific subtleties (e.g., macOS access-group configuration). Direct FFI per-platform is more work but avoids hidden behavior. Evaluate both in writing-plans.
2. **`zeroize` version pinning.** Ensure `compiler_fence` semantics remain strong under future Rust compiler updates. Pin and track upstream.
3. **Allocator instrumentation approach.** Custom global allocator (affects the whole binary; expensive test-mode only) vs. a scoped allocator wrapper for secret allocations (tighter but more code). Evaluate in writing-plans.
4. **Canary test determinism on Windows.** `ReadProcessMemory` against the current process has quirks; may need a companion process. Prototype in writing-plans.

These are implementation choices, not design decisions. Resolving them does not change any contract in this sub-spec.

---

## Exit checklist (per map §"Uniform exit checklist")

- [ ] This design doc committed under `docs/superpowers/specs/`
- [ ] Implementation passes unit + integration test suite (L2 + L3)
- [ ] Integration tests with direct tier-dependencies pass (this sub-spec has no substrate dependencies — the integration is with OS keystore APIs; tested via mock backends in test mode and real backends in platform-specific CI)
- [ ] Principle compliance check re-run; any tensions logged with resolution (see table above)
- [ ] Spec-specific acceptance test passes (L4 canary test in CI)

---

## Amendments log

| Date | Change | Reason |
|------|--------|--------|
| 2026-04-18 | V0 design created (Gavin + Cowork). | First sub-spec authored under the V1 spec map (T1, design Phase I). Supersedes early-spec's age-blob approach per P5. |
