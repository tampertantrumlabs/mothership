# Audit log — sub-spec design

**Status:** V0 design — drafted 2026-04-19.
**Authored:** 2026-04-19 (Gavin + Cowork).
**Supersedes:** nothing in `docs/brainstorm/early-spec.md` (the brainstorm does not specify an audit log beyond "monitor levels"; this sub-spec is the first engineering-forcing treatment of P8).

## Citations

- **Map:** [`docs/superpowers/specs/2026-04-17-v1-spec-map-design.md`](./2026-04-17-v1-spec-map-design.md)
- **Principles load-bearing:** P8 (primary); P5 (consumes audit HMAC key handle); P2 (write path unreachable from agent process); P4 (provenance-gated decisions are logged here); P7 (grant decisions are logged here)
- **Use cases cited:** UC1 (primary — per-client audit report as security attestation), UC2 (primary — compliance-grade chain of custody), UC3 (primary — SIEM stream for SOC)
- **Tier:** T1
- **Upstream:** Secrets bridge (HMAC key handle `mothership/audit/hmac`)
- **Downstream consumers:** Supervisor (T2), Broker + provenance (T3), Skill process isolation (T4), Origin & operator-presence gateway (T3), Secrets bridge (T1 — export/import events)

## Purpose

Provide every other Mothership sub-spec with an append-only, tamper-evident write path for every capability invocation, approval decision, config change, plugin install, grant change, secret-bridge privileged op, and supervisor action. Enforce P8's invariant: **a full compromise of both the agent and supervisor processes cannot hide its tracks by rewriting the log without being detectable by an operator running `audit verify` against an out-of-band copy.**

The log is the ground truth the runtime produces about itself. Every other principle's compliance check depends on being able to read the log and see what happened. A leak or silent drop here defeats the entire trust story.

---

## Scope

### In V1

- **Storage:** append-only JSON Lines (JSONL) segments on local disk, size- and time-bounded, with segment manifest chaining.
- **Tamper evidence:** per-entry HMAC-SHA256 chain; per-segment roll-up MAC; monotonic sequence number independent of wall-clock.
- **Write API:** typed in-process Rust API with a small event-shape registry (generated constants; no free-form strings for event types).
- **Verify CLI:** `mothership audit verify` with local-only and external-mirror-diff modes.
- **External mirror adapters:** trait-based interface; V1 ships **local-file** and **S3 object-lock** adapters. Syslog / SIEM adapters deferred to V1.5 unless UC3 design partner requires earlier.
- **Retention:** operator-configurable minimum retention (default: 1 year local, indefinite on S3 object-lock); size-capped rotation with oldest-segment archival before deletion.
- **Chain-restart markers:** explicit `audit-chain-restart` event for secrets bridge import and for operator-authorized recovery from corruption.

### Out of V1 (explicit non-goals)

- **Querying / indexing.** The log is append-only and read-sequentially or via external mirror tooling (SIEM, grep, operator scripts). Mothership ships `verify` and a cursor-based read API; it does not ship a query engine.
- **Encryption at rest of the local log.** The log is integrity-protected, not confidentiality-protected. Payloads must not carry secret material by design (see "Payload hygiene" below). Operators who need confidentiality use full-disk encryption — out of Mothership's scope.
- **Real-time streaming push adapters beyond local-file and S3.** Syslog, SIEM webhooks, Kafka connectors, etc., ship in V1.5+ unless a design partner hard-requires earlier.
- **Log signing with asymmetric keys.** HMAC is sufficient for tamper evidence when paired with the external mirror. Ed25519-signed entries considered for V2; the extra key-management surface is not justified in V1.

---

## Event model

### Event shape (stable contract)

Every entry is a single JSONL line with these fields, in this order:

| Field | Type | Description |
|-------|------|-------------|
| `seq` | u64 | Monotonic sequence number, chain-scoped. Starts at 1 in the genesis entry of the chain. |
| `ts_wall` | RFC3339 UTC | Wall-clock timestamp. Informational. |
| `ts_mono_ns` | u64 | Nanoseconds since process start (monotonic). Used for intra-process ordering when wall-clock drifts. |
| `key_epoch` | u32 | Identifier of the HMAC key epoch under which `mac` is computed. Increments on every key rotation / import / recover. Binds an entry to a specific key version so post-rotation entries cannot silently verify under the wrong key. |
| `event_type` | enum str | Declared event-type constant. Must match a generated-constant value; free-form strings fail validation at write time. |
| `actor` | object | `{ kind: "supervisor" \| "broker" \| "skill" \| "agent" \| "operator" \| "origin-gateway", id: str }`. |
| `org_context` | str \| null | Org-context namespace (TTL / APIsec / shared / null for substrate-level events). |
| `payload` | object | Event-type-specific fields; validated against an event-type schema at write time. |
| `prev_mac` | hex str | HMAC of the previous entry (genesis entry uses the segment-0 genesis MAC). |
| `mac` | hex str | HMAC-SHA256 of canonical-JSON-serialized `{seq, ts_wall, ts_mono_ns, key_epoch, event_type, actor, org_context, payload, prev_mac}` using the key at handle `mothership/audit/hmac` at the specified `key_epoch`. |

### Event-type registry

A single source file `audit-events.yaml` declares every event type, its payload schema, and its severity. Codegen emits:

- Rust enum + payload structs (supervisor, broker, skill-isolation, gateway, secrets-bridge)
- TypeScript enum + Zod schemas (CLI, future web UI)
- Python enum + Pydantic models (test fixtures, ops tooling)

Writing an event type not in the registry fails at compile time. Adding a new event type is a one-line YAML change plus a rebuild.

Seed event types for V1 (not exhaustive; each sub-spec adds its own during its design):

- `supervisor.started`, `supervisor.stopped`, `supervisor.config-loaded`, `supervisor.config-rejected`
- `policy.matrix-updated`, `policy.grant-created`, `policy.grant-revoked`, `policy.approval-decision`
- `plugin.install-requested`, `plugin.install-approved`, `plugin.install-rejected`, `plugin.signature-verified`, `plugin.signature-failed`
- `broker.intent-admitted`, `broker.intent-denied`, `broker.provenance-violation`, `broker.sensitive-intent-reviewed`
- `skill.started`, `skill.stopped`, `skill.capability-violation`
- `gateway.reauth-success`, `gateway.reauth-failure`, `gateway.origin-rejected`, `gateway.session-expired`
- `secrets.enroll`, `secrets.export`, `secrets.import`, `secrets.delete`, `secrets.corruption-detected`
- `audit.chain-restart`, `audit.segment-rolled`, `audit.verify-ran`, `audit.mirror-push-failed`

### Payload hygiene (schema taint rules)

**Approach: schema-level taint, not name-based denylist.** Every field in every payload schema carries one of four taint labels:

| Label | Semantics | Example |
|-------|-----------|---------|
| `audit_safe` | Plain value, no hygiene constraints. | `handle_name`, `severity`, `seq` |
| `opaque_hash` | Must be a hex-encoded hash; no raw material. Codegen emits a newtype requiring hash construction. | `content_hash`, `recipient_key_fingerprint` |
| `secret_ref` | Must be a SecretHandle (from Secrets bridge). Codegen rejects any concrete secret value here. | `signing_key_handle` |
| `forbidden_raw` | Field is explicitly annotated as not-permitted-in-audit. Codegen fails compilation if any event schema uses this. Present only as a documentation-level marker for fields that **cannot** appear in audit. | (not used directly; referenced in denied schemas) |

**Recursive validation.** The CI linter walks each schema recursively: nested objects, arrays of objects, map values, enum variants, and generated bindings. Every leaf field must carry a taint label, explicitly. A missing label fails CI — silent-default-to-safe is prohibited. This closes the nested-escape hole Codex flagged: fields like `headers.authorization`, `credentials.value`, `request.text`, `metadata.blob` cannot slip through because their parent schema must have declared their taint labels.

**Name-pattern backstop.** In addition to the taint walk, a secondary lint scans for suggestive field names (`*_token`, `*_key`, `password`, `secret`, `raw_content`, `body`, `auth*`, `credential*`). Any match without an `opaque_hash` or `secret_ref` taint fails CI with a directed error. Backstop, not primary defense.

**Runtime assertion.** Writers constructed via generated types cannot bypass taint at compile time (Rust) or at build time (TS / Python). Arbitrary-map payload constructors do not exist in the public API.

---

## Chaining scheme

### Per-entry MAC

`mac_n = HMAC-SHA256(K_audit, canonical_json(entry_n_without_mac))`

where `entry_n_without_mac` includes `prev_mac = mac_{n-1}`. Canonical JSON is RFC 8785 (JCS) serialization to eliminate whitespace / field-order ambiguity.

### Genesis MAC per chain

First segment of a fresh install begins with:

`prev_mac_0 = HMAC-SHA256(K_audit, "mothership-audit-chain-v1" || install_fingerprint)`

where `install_fingerprint` is a supervisor-computed value stable across restarts but unique per install (derived from supervisor identity key, deterministically, no secret leakage).

This makes the genesis MAC bind to the specific install, so two installs cannot be silently spliced together.

### Segment rotation

Segments roll on the **first** of:

- Size threshold reached (default 100 MiB; operator-configurable).
- Time threshold reached (default 24 hours wall-clock; operator-configurable).
- Operator-issued `mothership audit roll` (manual).

On roll:

1. Write a final entry `audit.segment-rolled` with payload `{closing_seq, closing_mac, next_segment_id, key_epoch}`. This entry is itself chained with its own `mac`.
2. Compute segment MAC `seg_mac = HMAC(K_audit, closing_mac || segment_metadata || key_epoch)`. This is a **manifest-level** check, not a chain-continuity value.
3. Append a line to the segment manifest `audit.manifest.jsonl` with `{segment_id, first_seq, last_seq, first_ts, last_ts, closing_mac, seg_mac, prev_seg_mac, key_epoch_range}`.
4. **Chain continuity into next segment: the first entry of the new segment carries `prev_mac = closing_mac`. It does NOT use `seg_mac`.** `seg_mac` exists only to let `verify` detect manifest tampering; entry-level chain continuity is `prev_mac = closing_mac` end-to-end.

Segment files are named `audit.YYYYMMDD.NNNN.jsonl` (lex-sortable).

### Chain-restart semantics (explicit)

An `audit.chain-restart` event is written when:

- `secrets import` restores a secrets-bridge backup that rotates `K_audit`.
- Operator runs `mothership audit recover --from <segment>` after a documented corruption event.
- Operator runs `mothership audit rotate-key` (V1.5+; not in V1).

**Chain-restart payload (required fields):**

```
{
  "restart_reason": "secrets-import" | "operator-recover" | "key-rotation",
  "prior_closing_mac": <hex>,            // mac of the last entry of the old chain, if available
  "prior_closing_seq": <u64>,            // seq of that entry
  "prior_key_epoch": <u32>,              // key_epoch of the previous chain
  "new_genesis_mac": <hex>,              // HMAC(K_audit_new, "mothership-audit-chain-v1" || install_fingerprint || new_key_epoch)
  "new_key_epoch": <u32>,                // incremented; new_key_epoch > prior_key_epoch
  "recovery_authority": {                // who authorized the restart
    "method": "operator-presence-reauth" | "secrets-import-reauth",
    "challenge_id": <str>,               // audit.gateway.reauth-success seq in the prior chain, if reachable
    "operator_identity_fingerprint": <hex>
  }
}
```

The restart entry's own `mac` is computed under the **new** `K_audit` and the **new** `key_epoch`. This is the pivot point: entries before the restart verify under the prior key; the restart entry and everything after verify under the new key. `verify` handles this explicitly — it requires a continuous key-epoch transition (`new_key_epoch = prior_key_epoch + 1` unless documented otherwise) and rejects any restart where the transition is non-monotonic or where `prior_closing_mac` does not match the actual last-entry MAC of the prior chain as stored locally or on the mirror.

**Fsync ordering for restart** (mandatory): (a) the new key is resolved and held in a scoped `SecretBytes`; (b) the `audit.chain-restart` entry is fully written and fsynced under the new key + new epoch; (c) only then does the supervisor publish the new key epoch to downstream components. If step (b) fails, fail-closed mode activates and the new key is not adopted. This prevents the "later entries signed with a key whose activation was never durably recorded" failure Codex flagged.

`verify` recognizes restart markers and verifies each pre-restart and post-restart segment independently, with a single summary line per restart showing the epoch transition and the authority method.

---

## Write API

### In-process Rust API (authoritative)

```rust
trait AuditWriter {
    fn emit<E: AuditEvent>(&self, event: E) -> Result<AuditSeq, AuditError>;
}
```

where `AuditEvent` is the generated-constant event-type trait. `emit` is synchronous, ordered, and **fail-closed**: if the write cannot be persisted and acknowledged, the call returns an error and the caller's op MUST abort.

### Durability

Each entry is `fsync`'d before `emit` returns, V1 default. This is expensive; batched group-`fsync` is a V1.5 optimization gated on measurable bottleneck. The correctness argument for per-entry fsync: a capability invocation that completes without its audit entry durable on disk violates P8.

### Access control

The audit log file handles are opened by the supervisor process at startup with mode `O_APPEND | O_WRONLY` and have their fd passed to in-process components via capability references, never via path. No component outside the supervisor's process family has `open()` access to the log directory (enforced by fs permissions + per-skill process sandbox from sub-spec 4). Agent and skill processes emit audit events via broker-mediated intents that route into the supervisor's in-process `AuditWriter`.

This design defeats "compromised skill silently truncates the log" — the skill has no path to the file. It does not defeat "compromised supervisor silently drops writes" — that is the external mirror's job.

### Fail-closed-on-unavailable

If the supervisor determines the audit writer cannot persist (disk full, fs read-only, HMAC key unresolvable), the supervisor transitions to **audit-unavailable** mode. This state is explicit, documented, and operator-understood — it is an intentional denial-of-availability posture per P8, not a bug. Operators who want a liveness-over-integrity posture cannot have it; that's the trade-off Mothership makes and consumers accept.

#### Audit-unavailable downstream contract

For each consumer, state machine on `audit-unavailable`:

| Consumer | Action during `audit-unavailable` |
|----------|-----------------------------------|
| Supervisor — grant create | **Deny.** No new grants issuable without durable audit. |
| Supervisor — grant revoke | **Allow, in-memory only.** Revocation itself fails-safe (reduces capability); kept in-memory; persisted to audit on recovery with a backdated marker. |
| Supervisor — plugin install | **Deny.** Install always requires durable audit. |
| Supervisor — config load | **Deny.** Config changes require audit. |
| Broker — intent admission (low-risk, read-only, `operator-typed`-only chain) | **Allow** with in-memory best-effort buffering; drained to audit on recovery. |
| Broker — intent admission (any other) | **Deny.** |
| Skill process isolation — skill startup | **Deny.** New skills require durable audit. |
| Skill process isolation — skill shutdown | **Allow.** Shutdown is fail-safe; logged on recovery. |
| Secrets bridge — `resolve` | **Allow** (already not logged per secrets-bridge design). |
| Secrets bridge — `enroll` / `export` / `import` / `delete` | **Deny.** Privileged secrets ops always require durable audit. |
| Origin + operator-presence gateway — re-auth for recovery path | **Allow** via **minimal-authority sub-log** (see below). |
| Origin gateway — re-auth for any non-recovery privileged action | **Deny.** |

#### Minimal-authority recovery path

To avoid the deadlock Codex flagged — where the gateway needs audit to record a recovery re-auth, but audit is what's unavailable — the supervisor holds a **minimal-authority sub-log** on a pre-reserved, separately-sized path (default 1 MiB, never rotates, always available unless the entire disk is read-only). Only the following entries may be written to it:

- `audit.minimal.reauth-challenge-issued`
- `audit.minimal.reauth-challenge-result`
- `audit.minimal.recovery-approved`
- `audit.minimal.recovery-rejected`
- `audit.minimal.transition-recorded` (the last entry of minimal log before main log resumes)

The minimal sub-log is HMAC-chained with the same `K_audit` but carries a distinct chain identifier (`minimal-v1`) so it cannot be spliced into the main chain. On successful recovery, the main chain's first post-recovery entry is `audit.chain-restart` whose `recovery_authority.challenge_id` points to the minimal sub-log entry. `verify --diff-minimal` is a separate mode.

If the minimal sub-log path is **also** unavailable (whole disk read-only / unmounted), the supervisor halts hard — there is no further fallback. An operator intervention at the OS level is required.

#### Out-of-band operator notice

When the supervisor enters `audit-unavailable`, it publishes notice via three operator-visible channels, none of which depend on the audit writer or the disk:

- **Stderr of the supervisor process** — human-readable banner repeated every 15s.
- **OS notification** — platform-native (Windows toast, macOS notification, Linux libnotify) emitted at transition. Non-persistent, operator-advisory.
- **Supervisor status-pipe** — a tiny memory-mapped status file at a fixed path updated every second; `mothership status` CLI reads it directly (no broker, no audit dependency).

This prevents the "silent audit-unavailable" operational failure: an operator running `mothership status` immediately sees the state.

#### Throughput budget (binding)

Per-entry `fsync` in V1 must support **sustained 200 entries/sec** and **burst 1000 entries/sec over 1 second** without entering backpressure; profiling on the V1 target hardware (GMKtek, NVMe SSD) must demonstrate this before Phase II build is complete. If the measured rate is below budget, Phase II halts and group-fsync is pulled forward from V1.5 with a threat-model review. Backpressure semantics: `emit` returns a specific `BackpressureError` that callers treat as a hard `audit-unavailable` trigger (not a retry-and-hope). No silent queueing.

This realizes P8: **if you can't log it, you can't do it — and you know immediately when you can't.**

---

## External mirror adapters

### Adapter interface

```rust
trait MirrorAdapter {
    fn push_entry(&self, entry: &RawEntry) -> Result<(), MirrorError>;
    fn push_segment(&self, segment: &ClosedSegment) -> Result<(), MirrorError>;
    fn health(&self) -> HealthStatus;
}
```

Push semantics:

- **Entry-level push** is the default for events tagged `severity: high` (policy changes, grant changes, plugin installs, secrets import/export, provenance violations). Small, low-volume, tamper-critical.
- **Segment-level push** is the default for all other events. Pushed on segment roll. Bounded bandwidth, not tamper-critical at sub-second resolution.

#### Push-failure handling (coalesced, rate-limited)

If an entry-level push fails, the supervisor emits `audit.mirror-push-failed` locally, **coalesced and rate-limited** per `(mirror_target, error_class)` tuple: at most one `audit.mirror-push-failed` entry per 60 seconds per tuple, carrying a count and a sample of the most recent failure detail. This prevents a mirror-disk-full scenario from recursively flooding the local audit log and accelerating fail-closed on the same shared disk.

Retry with operator-configurable backoff. If entry-level push failures exceed operator threshold and the mirror is configured authoritative, supervisor enters `audit-unavailable`. If the mirror is non-authoritative (default), mirror failures are alerting-only.

Important: `audit.mirror-push-failed` writes itself go through the same fail-closed rules. If the local disk can't persist the push-failure record, the supervisor transitions directly to `audit-unavailable` rather than attempting further local writes.

### V1 adapters

- **local-file** — writes to an operator-specified path outside the default audit directory (e.g., external disk, network mount). Same JSONL format + manifest. Useful for ops who want a simple out-of-band copy.
- **S3 object-lock** — writes each entry (for high-severity) and each closed segment to an S3 bucket with object-lock enabled. UC2's compliance-grade option.

#### S3 object-lock adapter — required configuration

Authoritative-mirror mode is only honored if **every** one of the following is satisfied; any misconfiguration surfaces at adapter init as a hard error and prevents adapter registration:

- **Object-lock mode:** `COMPLIANCE` (not `GOVERNANCE`). Compliance mode forbids root-account retention override; governance mode does not. For audit authoritativeness, compliance mode is required.
- **Retention period:** ≥ operator-configured minimum retention (default 365 days). Shorter retention invalidates the compliance posture.
- **Versioning:** must be enabled on the bucket (object-lock requires it).
- **Legal hold:** adapter does not manage legal holds — they are an operator-attached concern. Adapter documents how to pair with legal hold for litigation contexts.
- **Bucket policy:** must deny `s3:DeleteObject`, `s3:PutObjectRetention` with shorter retention, and `s3:BypassGovernanceRetention` (redundant with COMPLIANCE mode but defense-in-depth). Adapter init validates the bucket policy shape; missing denies fail init.
- **Cross-account write:** strongly recommended — the Mothership install writes into a bucket owned by a different AWS account so local AWS-credential compromise does not grant bucket-admin. Documented as best practice; not enforced at init in V1.
- **Root-account override:** in COMPLIANCE mode, even the AWS account root cannot shorten retention or delete locked objects before expiry. This is the immutability property. Operators who want tunable retention must use **non-authoritative** mode (S3 without object-lock), accepting weaker tamper guarantees.

Non-authoritative S3 adapters (plain S3, no object-lock) are allowed in V1 but cannot be marked authoritative; `audit verify --diff-mirror` emits a warning when run against a non-authoritative mirror.

### Deferred (V1.5+)

- **syslog** (UC3 SOC integration)
- **SIEM webhook** (generic JSON POST; design partner may accelerate)
- **Kafka / Kinesis connectors**

Adapter selection and configuration is operator-set via supervisor config (signed per P2). Adding a new adapter is a plugin-bundle extension point — adapter binaries carry their own signed manifest declaring `network:write` only to the configured mirror endpoint.

---

## Verify CLI

### Subcommands

- `mothership audit verify` — verify the local chain, all segments.
- `mothership audit verify --segment <id>` — verify a specific segment.
- `mothership audit verify --since <ts>` — verify from a timestamp forward.
- `mothership audit verify --diff-mirror <path-or-url>` — verify local chain AND diff against an external mirror; report divergences.
- `mothership audit roll` — operator-invoked segment roll (re-auth gated).
- `mothership audit recover --from <segment>` — operator-invoked chain restart after corruption (re-auth gated; writes `audit.chain-restart`).

### `verify` output

Exits 0 on clean verification; non-zero on any mismatch. Output is structured (human-readable by default, `--json` for tooling):

```
segment audit.20260419.0001.jsonl: OK (seq 1..4732, mac OK)
segment audit.20260419.0002.jsonl: FAIL at seq 5199 (mac mismatch; prev_mac inconsistent)
  first divergent entry: event_type=policy.grant-created, actor=supervisor/main
  diagnosis: prev_mac does not match mac of seq 5198; local chain has been rewritten or inserted into
mirror diff (--diff-mirror s3://...): 2 entries present in mirror, absent from local (seq 5199..5200)
  diagnosis: local chain rewrite; mirror is authoritative; recover via `audit recover`
```

### Re-auth for verify

Read-only `verify` is not re-auth-gated (it's a diagnostic). `audit roll` and `audit recover` require operator-presence re-auth via sub-spec 7 and produce their own audit entries (paradoxically: the act of recovering from a broken chain is itself the first entry of the next chain).

---

## Test strategy (4 levels)

### L1 — Schema + type-level controls (design, verified by compilation)

- Event types are a closed enum; free-form strings fail to compile.
- Payload schemas are generated from `audit-events.yaml`; payloads not matching their schema fail at write time.
- Payload-denylist lint runs in CI on the YAML file; forbidden field names (`*_token`, `*_key`, `password`, `secret`, `raw_content`, `body`) fail CI unless explicitly allowlisted with justification.

### L2 — Unit tests

- HMAC chain math: entries sequence correctly, `mac` is reproducible given the key + epoch, canonical-JSON serialization is deterministic.
- **Bit-exact canonical-JSON parity across Rust/TS/Python.** A golden-vector suite (JSON inputs → expected canonical bytes → expected MAC) must produce identical bytes and identical MACs in all three implementations. This is an acceptance-level constraint pulled into L2 because it's correctness-critical for T1.
- Segment rotation: size / time / manual triggers all produce the expected `segment-rolled` + manifest line + correct new-segment `prev_mac = closing_mac` (NOT `seg_mac`).
- Chain-restart: import event → restart marker with all required payload fields (`restart_reason`, `prior_closing_mac`, `prior_closing_seq`, `prior_key_epoch`, `new_genesis_mac`, `new_key_epoch`, `recovery_authority`) → verify handles pre/post-restart independently → rejects non-monotonic `key_epoch` transitions.
- Genesis MAC binds to install fingerprint + key_epoch: two fresh installs with different fingerprints produce different genesis MACs; same install with different key epochs produces different genesis MACs.
- Fail-closed: simulated disk-full causes `emit` to return error; supervisor transitions to audit-unavailable; capability invocations abort per the downstream contract table.
- **Downstream contract** (one test per consumer row): during audit-unavailable, each listed action is allowed/denied per the table.
- **Minimal-authority sub-log**: reauth challenges write to it; entries are chained with `minimal-v1` identifier; cannot be spliced into main chain; resumed main chain's first entry references minimal seq via `recovery_authority.challenge_id`.
- **Key lease invariants**: max time window honored; max emit count honored; panic-drop zeroizes; mirror retry path does not extend residency; audit-unavailable transition drops lease immediately.
- **Mirror-push-failed coalescing**: 1000 failures against same tuple within 60s produce exactly one local entry with count=1000.
- **Payload taint recursion**: CI linter catches nested `credentials.api_key`, `headers.authorization`, array-of-objects with raw fields, enum variant with forbidden field, and generated-binding constant.

### L3 — Integration tests under adverse conditions

- Power-loss simulation (kill -9 the supervisor mid-write): after restart, `verify` reports the last durable entry, not a corrupted one. No torn writes observed (verified by fsync semantics + journal-fs test host).
- Clock-drift injection: wall-clock jumps backward by 24h; `ts_mono_ns` preserves ordering; `verify` does not false-positive.
- Concurrent-writer stress: 1000-thread emit storm; all entries have unique `seq`; chain verifies.
- Mirror unreachable: local-only chain continues; `audit.mirror-push-failed` is emitted; on recovery, queued entries push in order.

### L4 — Acceptance tests (CI gates; all must pass)

Each of these is a deterministic tamper scenario; each must cause `verify` to fail at the right point.

1. **Edit-in-place.** Modify one field of one entry in a segment. `verify` must fail at that entry with "mac mismatch."
2. **Drop entry.** Remove one entry from a segment. `verify` must fail at the following entry with "prev_mac does not match expected."
3. **Insert entry.** Forge an entry with a plausible shape (no valid MAC) and insert it. `verify` must fail at the inserted entry.
4. **Reorder entries.** Swap two entries. `verify` must fail at the first swapped entry.
5. **Truncate mid-segment.** Truncate a segment file mid-chain. `verify` must fail with "segment ends before declared `segment-rolled` marker."
6. **Full-local-wipe-and-replay.** Delete all local segments; replay a plausible-looking synthetic log. `verify --diff-mirror <mirror>` must fail: mirror entries present locally absent, mac chain of synthetic log fails genesis MAC check.
7. **Rewrite and re-MAC with a forged key.** Replace all MACs with a forged-key re-computation. `verify` must fail at the first entry: the legitimate `K_audit` does not reproduce the MAC.
8. **Segment manifest rewrite.** Modify the manifest to reorder segments or replace `closing_mac` with `seg_mac`. `verify` must fail at the first manifest inconsistency AND at the segment-boundary `prev_mac` check.
9. **Silent-drop by supervisor.** Kill the writer between emit-and-fsync; simulated supervisor drop. Fail-closed mode activates; subsequent invocation attempts abort with an operator-visible error.
10. **Cross-install splice.** Take a valid log segment from Install A (different `install_fingerprint`) and attempt to splice it onto Install B's chain. `verify` must fail at the spliced segment's genesis MAC check because genesis MAC binds to Install B's fingerprint + key_epoch.
11. **Cross-epoch splice.** Take a valid log segment from the prior key_epoch of the **same** install and splice it after a rotation. `verify` must fail at the entry's `key_epoch` field + MAC mismatch (prior-epoch entries cannot verify under new-epoch key).
12. **Key-rotation mid-flight failure.** Begin a `secrets import` → `audit.chain-restart` but fail the fsync of the restart entry. `verify` must show the chain continuing under the old key at the same position (new key never adopted); subsequent emits go through the prior chain, not the failed-restart chain.
13. **Segment-boundary prev_mac misuse.** Generate a chain where `prev_mac` of the first entry of segment N+1 is set to `seg_mac` instead of `closing_mac`. `verify` must fail at that entry with "segment boundary prev_mac must be prior segment's closing_mac, not seg_mac."
14. **Minimal-sub-log splice attempt.** Forge a main-chain entry carrying the `minimal-v1` chain identifier. `verify` must reject the splice on chain-identifier mismatch before MAC check.
15. **Recursive flood under disk-full.** Fill the audit disk; simulate 10,000 push failures in one minute. Assert: at most 1 `audit.mirror-push-failed` coalesced entry per tuple per 60s; `audit-unavailable` transitions cleanly without recursive self-logging loop.

---

## Error modes

- **HMAC key unresolvable at startup** — supervisor halts with actionable diagnostic; no audit log writes are attempted; downstream sub-specs cannot start.
- **Disk full / fs read-only** — fail-closed mode; all invocations abort; `audit-unavailable` alert raised when recovery happens.
- **Corrupted segment detected during `verify`** — diagnostic identifies the first divergent entry; operator can run `audit recover --from <segment>` to start a new chain after reviewing the mirror.
- **Mirror unreachable** — alerting by default, non-blocking; upgrade to blocking via operator config (UC2 compliance-grade deployments).
- **Clock goes backward** — log warning; do not fail; `ts_mono_ns` preserves ordering.
- **Event-type schema mismatch at write** — caller gets typed error; in-process Rust callers fail at compile time (generated constants + payload structs).

---

## Interface produced (consumed by downstream sub-specs)

- `AuditWriter` trait with typed `emit<E>` method. Consumers: Supervisor, Broker, Skill process isolation (via broker mediation), Origin gateway, Secrets bridge.
- `AuditSeq` — opaque monotonically-increasing handle returned by `emit`; used by callers that want to cross-reference entries (e.g., grant-created emits seq, subsequent grant-used events can reference it).
- Generated event-type constants + payload structs (Rust / TS / Python).
- `mothership audit` CLI subcommands (verify, roll, recover).
- `MirrorAdapter` trait + local-file and S3 object-lock adapter implementations.

---

## Principle compliance notes

| Principle | Compliance check | Result |
|-----------|-----------------|--------|
| P8 | Can a full compromise of both agent and supervisor hide its tracks by rewriting the log without being detectable by an operator running `audit verify` against an out-of-band copy? | **No.** Local HMAC chain detects in-place edits, drops, inserts, reorders, truncation, and manifest rewrites. External mirror (S3 object-lock, per UC2) detects full-local-wipe. Fail-closed on persistence failure prevents silent drops. |
| P5 | Is the HMAC key ever resident outside the OS keystore past the scope of a single resolve call? | **No.** `K_audit` is resolved via secrets bridge handle `mothership/audit/hmac`, used to compute one MAC, and its `SecretBytes` dropped. Every entry re-resolves (or uses a scoped short-lived `SecretBytes` held for a bounded emit burst; TBD in writing-plans — see Open Question 2). |
| P2 | Can a compromised agent process rewrite, disable, or redirect the audit log? | **No.** Log files are not `open()`-able by agent or skill processes (fs permissions + process sandbox). Agents emit via broker intents that route into the supervisor's in-process `AuditWriter`. A compromised supervisor remains a risk; mitigation is the external mirror + out-of-band `verify`. |
| P4 | Are provenance-gated decisions logged with enough detail to reconstruct what the matrix denied or permitted? | **Yes.** `broker.intent-admitted` and `broker.intent-denied` event types carry actor, intent type, provenance chain fingerprint, matrix rule that fired, and decision. Full raw-content logging is prohibited by payload hygiene; content hashes are permitted. |
| P7 | Are grant creations, uses, and revocations logged? | **Yes.** `policy.grant-created`, `policy.grant-revoked`, `policy.grant-used` are all declared event types. |

### Principle tensions

- **P5 vs. throughput.** Per-entry HMAC key resolution through the secrets bridge imposes a keystore round-trip per entry. Under high-rate emit (e.g., broker admitting a thousand intents/sec during agent-heavy work), this is a real cost.
  - **Resolution:** Writing-plans evaluates a **scoped `SecretBytes` lease** — supervisor holds the key in a single `SecretBytes` for a bounded emit window (e.g., 1 second or N emits, whichever first), then drops. This preserves P5's "past the lifetime of a single API call" constraint at the API layer (the supervisor's internal emit loop is the API call) while making high-rate emit tractable.
  - The lease approach MUST be validated in the Secrets bridge L4 canary test; the audit log L2 tests include a "lease-window zeroize" check.

---

## Open implementation-level questions (for writing-plans, not design)

1. **HMAC key lease window bounds.** Time-bounded (e.g., 1s), emit-count-bounded (e.g., 1000), segment-boundary-bounded, or tuple of all three whichever-first? Default candidate: min(1s wall, 1000 emits, segment roll). Prototype and measure; validate lease-window zeroize semantics cross-reference with Secrets bridge L4 canary (this is a Secrets bridge acceptance extension).
2. **Canonical JSON parity implementation.** `serde_jcs` (Rust) ↔ `json-canonicalize` (TS) ↔ `jcs` (Python) — need a golden-vector test suite validated bit-exact across all three. Golden vectors must cover: unicode normalization edge cases, number rendering (NaN/Infinity rejected), ordering with unicode keys, null handling in optional fields, enum-variant string rendering. Golden vectors live in `audit/golden-vectors.json`.
3. **Oldest-segment archival destination.** After local retention expires, archive where? Default candidate: operator-configured local archive path + S3 Glacier as secondary mirror; design-partner input may refine.
4. **Status-pipe format.** Memory-mapped file layout for `mothership status` to read audit-state without depending on audit writer. Candidate: fixed-size struct with version, state enum, last-success-seq, last-error, timestamp. Cross-platform (mmap on Linux/Mac, `MapViewOfFile` on Windows).

**Closed by this revision (no longer open):**

- ~~Per-entry vs. group fsync~~ → Per-entry required in V1 with binding 200/s sustained, 1000/s burst throughput budget. Group-fsync deferred to V1.5 pending profiling + threat review.
- ~~S3 object-lock governance vs. compliance mode~~ → COMPLIANCE mode required for authoritative-mirror status; governance mode permitted only for non-authoritative mirrors with `verify --diff-mirror` warning.

These remaining items are implementation choices, not design decisions. Resolving them does not change any contract in this sub-spec.

---

## Exit checklist (per map §"Uniform exit checklist")

- [ ] This design doc committed under `docs/superpowers/specs/`
- [ ] Implementation passes unit + integration test suite (L2 + L3)
- [ ] Integration tests with direct tier-dependencies pass (Secrets bridge resolves `mothership/audit/hmac` correctly; `SecretBytes` lease window honors zeroize semantics)
- [ ] Principle compliance check re-run; any tensions logged with resolution (see table above — P5-vs-throughput tension documented with resolution path)
- [ ] Spec-specific acceptance test passes (all 9 L4 tamper scenarios in CI)

---

## Amendments log

| Date | Change | Reason |
|------|--------|--------|
| 2026-04-19 | V0 design drafted (Gavin + Cowork). | Second sub-spec authored under the V1 spec map (T1, design Phase I). |
| 2026-04-19 | V0.1 revision post-Codex review. | Added explicit `key_epoch` field to entry shape and chain-restart payload; clarified `closing_mac` vs `seg_mac` roles at segment boundaries; added fail-closed downstream contract table + minimal-authority recovery sub-log to avoid T1↔T3 deadlock; replaced payload denylist with recursive schema-taint rules; added throughput budget; tightened S3 object-lock to COMPLIANCE-mode-required for authoritative mirrors; added mirror-push-failed coalescing; added out-of-band operator notice via stderr/OS-notification/status-pipe; added 6 new L4 acceptance tests (cross-install splice, cross-epoch splice, mid-flight key-rotation failure, segment-boundary prev_mac misuse, minimal-sublog splice attempt, recursive push-failure flood). See Codex Review section below. |

---

## Codex Review

**Triggers matched:** auth/authorization (HMAC key management, re-auth gating), infrastructure (fs permissions, S3 object-lock, systemd/launchd service), secrets (audit HMAC key lease), irreversible one-shots (append-only audit log, chain restarts).
**Codex effort:** default
**Reviewed:** 2026-04-19

### Codex findings (summary)

**Rollback.** Chain-restart under-specified: `seq` restart behavior, `prev_mac` binding, new-genesis-MAC computation, and fsync ordering around key rotation were ambiguous; could produce apparent valid fork where old/new entries verify under different keys with no independently auditable boundary. Segment-rotation ambiguity: three MAC-like values (`closing_mac`, `seg_mac`, manifest metadata) without a normative statement on which drives next-segment `prev_mac`. Recommendation: explicit `key_epoch` field + normative `audit.chain-restart` payload + normative `prev_mac = closing_mac` at segment boundaries.

**Blast radius.** Fail-closed cascade under-specified for downstream T2/T3/T4 consumers. "Grants frozen" semantically ambiguous (new? revoke? all mutation?). Bootstrap loop risk: Origin+operator-presence gateway is downstream of audit; if recovery re-auth itself needs audit, recovery can deadlock. Recommendation: per-consumer state table + minimal-authority recovery path.

**Missing tests.** HMAC chain math incomplete without splicing attacks (cross-install, cross-segment, cross-key-epoch). Segment-boundary `prev_mac` handoff not explicitly tested. Key lease needs max-time, max-count, panic-drop, mirror-retry, and audit-unavailable-transition tests. `audit.mirror-push-failed` can recurse under disk-full if the failure-event write itself fails. Payload hygiene denylist doesn't cover nested schemas (`payload.credentials.api_key`, `headers.authorization`).

**Wrong assumptions.** RFC 8785 JCS cross-language bit-exact parity cannot remain an open question for a T1 sub-spec — entry MAC depends on it. Per-entry fsync throughput is an unverified assumption; needs target + budget + backpressure behavior. S3 object-lock "authoritative" requires specifying mode + retention + legal hold + root-override semantics; design doesn't state them. `SecretBytes` lease is intent, not enforceable boundary, without explicit memory ownership / clone prevention / crash dump / zeroize guarantees.

**Cheapest failure.** (a) Silent audit-unavailable when disk full — grants freeze, operators may misdiagnose as broken rather than compromised and wipe state. (b) Denylist field-name bypass via nested names like `metadata.blob` or `request.text`. (c) Fail-closed cascade itself is an availability attack surface (fill audit disk → deny revocable tools); design should name this as intentional.

**Alternatives.** (1) Explicit `key_epoch` + structured `audit.chain-restart` payload with `prior_closing_mac`, `new_genesis_mac`, `restart_reason`, authority binding. Materially better than implicit restart marker because rotation/import/recovery become independently checkable under P8. (2) Normative `audit-unavailable` state table per downstream consumer. (3) Schema taint rules (`audit_safe` / `secret_ref` / `opaque_hash` / `forbidden_raw`) recursively validated instead of name-pattern denylist. (4) Rate-limited coalesced `audit.mirror-push-failed` by target + error class.

### Claude's resolution

- **Rollback finding:** **Accepted.** Added `key_epoch` field to entry shape and `mac` input. Added explicit `audit.chain-restart` payload schema with `restart_reason`, `prior_closing_mac`, `prior_closing_seq`, `prior_key_epoch`, `new_genesis_mac`, `new_key_epoch`, `recovery_authority`. Stated normative fsync ordering: new key held in `SecretBytes` → restart entry written under new key + new epoch → fsync → only then publish new epoch to downstream. Clarified segment boundary: next-segment `prev_mac = closing_mac` (NOT `seg_mac`); `seg_mac` is manifest-level check only.
- **Blast radius finding:** **Accepted.** Added per-consumer fail-closed state table covering Supervisor (grant create/revoke/install/config), Broker (intent admission rules), Skill isolation (start/stop), Secrets bridge (all ops), Origin gateway (re-auth). Added minimal-authority sub-log (`minimal-v1` chain identifier, distinct from main chain, pre-reserved 1 MiB) for recovery re-auth challenge that breaks the T1↔T3 deadlock.
- **Missing tests finding:** **Accepted.** Added L2 tests for lease invariants, mirror-push-failed coalescing, payload taint recursion, canonical-JSON bit-exact parity (moved from open-question to correctness-critical L2). Added L4 tests: cross-install splice (#10), cross-epoch splice (#11), mid-flight key-rotation failure (#12), segment-boundary prev_mac misuse (#13), minimal-sublog splice (#14), recursive push-failure flood (#15). Existing L4 #8 strengthened to include the `closing_mac` vs `seg_mac` misuse case.
- **Wrong assumptions finding:** **Accepted.** Canonical-JSON parity moved from open-question to L2 acceptance-grade golden-vector suite across Rust/TS/Python. Added binding throughput budget (200/s sustained, 1000/s burst) with `BackpressureError` semantics; Phase II halts if budget unmet. S3 object-lock hardened: COMPLIANCE mode required for authoritative; retention, versioning, bucket policy denies, cross-account write best practice, root-override behavior all specified. `SecretBytes` lease guarantees cross-referenced to Secrets bridge L4 canary; audit L2 adds lease-window zeroize check.
- **Cheapest failure finding:** **Accepted.** (a) Added out-of-band operator notice via stderr banner, OS notification, and memory-mapped status pipe readable by `mothership status` without audit dependency. (b) Replaced name-pattern denylist with recursive schema-taint rules (`audit_safe`/`opaque_hash`/`secret_ref`/`forbidden_raw`); CI linter walks nested schemas, arrays, maps, enum variants, generated bindings. (c) Named the DoS-via-audit-disk surface explicitly as intentional P8 trade-off.
- **Alternatives finding:** **All four accepted.** `key_epoch` + structured restart payload, downstream state table, schema taint, push-failure coalescing are all now in the spec.

### Summary

Codex's review was substantively correct on every point; no findings were rejected. The revision adds five new load-bearing primitives to the design — `key_epoch` field, normative restart payload, downstream-consumer fail-closed contract, minimal-authority recovery sub-log, schema-taint validation — plus tightens throughput, S3, and cross-language parity from "open question" to "binding requirement." Six new L4 acceptance tests close the splicing / key-rotation / recursive-flood gaps. The spec now resists the concrete attacks Codex enumerated and avoids the T1↔T3 bootstrap deadlock.
