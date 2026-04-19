# Supervisor — sub-spec design

**Status:** V0 design — drafted 2026-04-19.
**Authored:** 2026-04-19 (Gavin + Cowork).
**Supersedes:** `docs/brainstorm/early-spec.md` treatment of "message broker as single enforcement point" where it conflates broker enforcement with supervisor ownership; the supervisor owns policy and grants, the broker (sub-spec 2) consumes them.

## Citations

- **Map:** [`docs/superpowers/specs/2026-04-17-v1-spec-map-design.md`](./2026-04-17-v1-spec-map-design.md)
- **Principles load-bearing:** P2 (primary — safety enforcement below the API plane), P6 (operator-presence orchestration), P7 (default-deny grant ledger); consumed by P5 (supervisor identity key in secrets bridge), P8 (every supervisor action emits an audit entry)
- **Use cases cited:** UC1 (primary — per-client grant scoping + org-context isolation), UC2 (primary — compliance-grade policy + audit chain), UC3 (primary — IT-signed config + SOC-approvable grant ledger)
- **Tier:** T2
- **Upstream:** Secrets bridge (T1), Audit log (T1), Plugin bundle + signing (T1)
- **Downstream consumers:** Broker + provenance (T3 — policy read API, grant ledger read API, approval-callback interface), Origin & operator-presence gateway (T3 — supervisor orchestrates re-auth), Skill process isolation (T4 — manifest registry, capability grants per skill)

## Purpose

Provide the **enforcement substrate below the API plane** that every Mothership principle depends on. The supervisor is the only process that can write policy, mutate grants, install plugins, re-auth operator-presence, or adjudicate capability requests. All such actions are gated by operator-presence re-auth (via sub-spec 7), logged to the audit chain (via sub-spec 5), and authorized against a signed, integrity-checked config that no agent process can reach.

Enforce P2's invariant: **a full compromise of the agent process cannot disable sandboxing, grant itself new capabilities, or alter the audit log.** The supervisor is the target of that compromise claim — if the supervisor is compromised, downstream enforcement falls; if the supervisor is intact, downstream enforcement holds regardless of agent behavior.

---

## Scope

### In V1

- **Service lifecycle** — native OS service on **Linux interactive-desktop** (systemd **user** unit under an operator session; system-unit / headless-boot is V1.5+ pending a TPM-backed or keyctl-based secret backend per secrets-bridge spec), **macOS** (launchd system daemon), **Windows** (Windows service); dedicated service account; no agent-reachable control surface.
- **Signed, integrity-checked config** — supervisor config is operator-signed with the operator's GPG key (reusing the key model already used for signed commits in this project). Config signature verified at every load; modifications require re-signing.
- **Versioned, broker-acknowledged state** — every supervisor-held state consumed by the broker (policy matrix, grant ledger, manifest registry, org-context registry) is exposed as a single `SupervisorSnapshot` with a monotonic `snapshot_version`; mutations that affect broker enforcement use a two-phase commit (prepare → broker-ack → activate) so broker and supervisor never diverge.
- **Capability grant ledger** — append-only, HMAC-chained grant records (per-skill, per-agent, per-capability, per-org-context); revocable; consulted at every broker intent routing decision.
- **Policy engine** — declarative provenance × capability matrix; operator-editable, signed; **monotonic-lock enforced against an embedded baseline** covering all untrusted-provenance × high-risk-capability cells; the authoritative decision source for every privileged action.
- **Plugin install orchestration** — supervisor drives the `reviewed` → `staged` → `committed` → `activated` transitions of the plugin-bundle install state machine (sub-spec 3); operator approval is audited immediately after re-auth (not only at `activated`); owns operator-facing review UI; persists approved grants to the ledger.
- **Agent manifest registry** — verified-manifest objects from installed bundles are held in an in-process registry consumed by the broker via `SupervisorSnapshot`.
- **Cold-boot unlock flow** — platform-specific sequence to unlock OS keystore and resolve the supervisor service key before anything else starts; per-phase halt-on-failure is the only mode.
- **Broker-before-agents startup** — strict ordering: supervisor → secrets bridge → audit log → broker → only then agents are permitted to spawn. **Enforced mechanically**: the supervisor holds the only privileged-spawn helper; before broker `ready`, the helper refuses all spawn requests.
- **Privileged-spawn helper** — minimal setuid helper with a declared command surface, authenticated input, allowed-UID allowlist, env scrubbing, and fd closure — part of the TCB and specified here.
- **Scrub-on-shutdown** — deterministic zeroization of in-memory secrets, final audit entry, fd closure, status-pipe update; in-flight installs and grant-ledger writes are committed-or-rolled-back before exit.
- **Org-context registry** — operator-configured list of org contexts (e.g., `ttl`, `apisec`, `shared`) mapped to per-context unix users (Linux/macOS) or Windows security principals (with ACLs and named-pipe / profile / temp-dir boundaries specified); load-bearing for UC1 filesystem isolation.
- **Operator-presence orchestration** — supervisor decides which ops require re-auth and handles timeouts/rejections/replay; primitives (Touch ID / Windows Hello / YubiKey / PAM) are delegated to sub-spec 7. Challenges are single-use and bound to a specific `(requested_action, scope)` tuple.
- **Grant-ledger recovery** — explicit algorithm for operator-invoked recovery after corruption (not just an error-mode bullet).

### Out of V1 (explicit non-goals)

- **Web-UI / GUI.** All operator interactions are CLI-invoked or re-auth-gated CLI in V1. A web dashboard is a V1.5+ add.
- **Hot-reload of the capability matrix.** V1 reloads on `supervisor reload` (re-auth gated) or SIGHUP (with signed-config verification at reload); live-in-place modification is out.
- **Headless-Linux unattended boot.** Linux V1 requires an operator session (session D-Bus / libsecret available) because the secrets bridge depends on it. A headless-service Linux variant needs a different secret backend (TPM-sealed / keyctl / systemd-credentials) and ships in V1.5+ alongside that backend. See secrets-bridge §"Out of V1" — this supervisor spec does NOT loosen that constraint.
- **Grant-expiry scheduling.** V1 grants have optional `expires_at`, but the supervisor does not proactively revoke expired grants on a timer — the broker checks expiry at evaluation time and denies if expired, and a `mothership grant prune` CLI job purges expired ledger entries. A real scheduler is V1.5+.
- **Multi-supervisor clustering / HA.** Mothership V1 is single-host. Multi-instance coordination (shared grant ledger, leadership election) is explicitly out.
- **Non-Claude model adjudication.** Fast-model sensitive-intent review (per `design-principles.md` P4) lives in the broker (sub-spec 2), not here. Supervisor only routes the request to the broker and logs the verdict.
- **Cross-tenant grant sharing.** Each org context has its own grant ledger scope; a grant in one context never implies a grant in another. Sharing is an explicit future-work item.
- **Batch-approval for privileged ops.** Every privileged op requires an individual challenge; explicit-friction posture kept until V1.5 design-partner evidence.

---

## Service lifecycle

### Per-OS service definition

| OS | Unit type | Service account | Boot policy |
|----|-----------|-----------------|-------------|
| Linux | systemd **system** unit `mothership-supervisor.service` | Dedicated user `mothership` (UID ≥ 1000; no login shell; home `/var/lib/mothership`) | `enable + start` at install; `Restart=on-failure` with backoff |
| macOS | launchd **system daemon** `labs.tampertantrum.mothership.plist` in `/Library/LaunchDaemons/` | Dedicated `_mothership` system user (UID < 500; `nologin` shell) | `RunAtLoad=true`; `KeepAlive=true` with throttle |
| Windows | Windows Service `MothershipSupervisor` | Dedicated virtual service account `NT SERVICE\MothershipSupervisor` | `Automatic` start; service recovery on failure |

The supervisor **never runs as root / Administrator / LocalSystem**. The service account has only the rights needed for its specific duties (bind to its status-pipe path, read/write `/var/lib/mothership/**` platform-equivalent, call into the OS keystore for its declared handles, spawn broker and skill processes under per-org-context unix users).

### Control surface

Operator-facing CLI: `mothership` binary (separate process; not the supervisor). CLI → supervisor communication is over a **unix domain socket** (Linux/macOS) or a **named pipe** (Windows) owned by the service account with peer-credential-check on connect. No TCP/IP listener. No agent-reachable socket. The gateway (sub-spec 7) is a separate process that does handle external browser-origin requests; the supervisor does not.

### Privilege boundary

The supervisor process runs with exactly these capabilities:

- Read the OS keystore for handles in the secrets-bridge registry scoped to `mothership/**`.
- Bind to and read from `/var/run/mothership/supervisor.sock` (platform-equivalent).
- Append-write to `/var/log/mothership/audit/*.jsonl` (and the audit manifest).
- Read/write `/var/lib/mothership/**`: `installs/**`, `bundles/**`, `rolled-back-bundles/**`, `revoked-bundles/**`, `journal/**`, `policy/**`, `grants/**`, `trust/**`.
- Spawn child processes (broker, skills) only via the **privileged-spawn helper** (below).

Any other capability — network binding, arbitrary exec, arbitrary fs write, raw setuid — is NOT held by the supervisor process. On Linux, this is enforced via `CapabilityBoundingSet` in the systemd unit + `NoNewPrivileges=yes` + seccomp profile. macOS uses sandbox-exec with a policy derived from this list. Windows uses service-restricted-token + SID deny-list. The exact OS-level enforcement snippet is in the writing-plans stage; this spec defines the set.

### Privileged-spawn helper (TCB component)

On Linux/macOS the supervisor is a non-root user but needs to spawn skill processes under per-org-context unix users. A minimal **privileged-spawn helper** (`mothership-spawnd`) is the only component in the trust computing base authorized to call `setuid(target_uid)`. Its command surface is deliberately tiny:

```
spawn --target-uid <uid> --target-gid <gid>
      --bundle-hash <sha256> --skill-id <normative id> --org-context <id>
      --exec <absolute canonical path within bundles/<bundle_hash>/>
      --stdin-fd <fd> --stdout-fd <fd> --stderr-fd <fd>
      --broker-control-fd <fd>
      --challenge-nonce <supervisor-issued nonce>
```

- **Authenticated input.** Helper only accepts a request over an inherited pipe from the supervisor, authenticated by a per-session nonce issued at supervisor start and held only in supervisor memory. No command-line parsing from external input.
- **Allowed target UID allowlist.** The helper loads `/etc/mothership/org-contexts.yaml` (signed by operator GPG) at start; `target_uid` must be exactly one of the declared per-context UIDs or the broker UID. Any other value → helper refuses.
- **Allowed exec path.** `--exec` must be an absolute path rooted under `/var/lib/mothership/bundles/<bundle_hash>/`; helper verifies the path is canonical, a regular file, the bundle_hash is in the active registry (cross-checked via supervisor), and the path's mode came from supervisor-imposed-at-commit modes.
- **Env scrubbing.** Helper clears the environment on exec; only a small, declared allowlist is passed through (e.g., `TZ`, `LANG`). No `LD_PRELOAD`, `LD_LIBRARY_PATH`, `PATH` from parent.
- **FD closure.** Helper `close_range()`'s everything except the three declared std fds + broker control fd; attempted fd smuggling fails.
- **No shell.** Helper `execve`s directly; no `/bin/sh -c`, no `system()`.
- **Audit.** Every invocation emits `supervisor.skill-spawned` or `supervisor.spawn-rejected` with reason code; included in the compromised-agent L4 suite.
- **Windows equivalent.** Windows uses `CreateProcessAsUserW` with a restricted token under `MothershipSupervisor` service SID; the "helper" is a supervisor-internal subroutine (not a separate binary) because Windows's process-creation model does not require a setuid-equivalent helper. Same allowlist / allowed-path / env-scrub / audit contract.
- **macOS equivalent.** Same as Linux pattern — a minimal `mothership-spawnd` launchd helper with `SetUID=YES` restricted to this helper binary only.

The helper is the most dangerous part of the TCB after the supervisor itself; Codex's review correctly flagged that a broad-surface helper can become an OpenClaw-equivalent disable-the-sandbox pivot. The above surface is intentionally minimal.

---

## Signed, integrity-checked config

### Files

| File | Purpose | Signed? | Modifiable? |
|------|---------|---------|-------------|
| `/etc/mothership/supervisor.yaml` | Supervisor-level config (service-account params, paths, defaults, timeouts) | Yes (operator GPG) | Re-auth + re-sign required |
| `/etc/mothership/policy.yaml` | Provenance × capability matrix | Yes (operator GPG) | Re-auth + re-sign required |
| `/etc/mothership/trust/*.pem` + `*.trust.yaml` | Plugin trust roots (see plugin-bundle spec) | `.trust.yaml` is signed by operator | Re-auth + re-sign required |
| `/etc/mothership/trust/sigstore-identities.yaml` | Sigstore identity allowlist (see plugin-bundle spec) | Yes (operator GPG) | Re-auth + re-sign required |
| `/var/lib/mothership/grants/ledger.jsonl` | HMAC-chained grant ledger | Not separately signed (HMAC chain) | Append-only via supervisor API |

### Signature model

Each signed config file ships alongside a detached GPG signature (`supervisor.yaml.sig` etc.). The operator's public GPG key is declared once in `/etc/mothership/operator.pub` and pinned at install. On every config load, the supervisor verifies the signature against the pinned operator key before parsing the file. A signature failure is a hard halt with actionable diagnostic — there is no "degraded mode."

Rotating the operator key is an operator-presence-gated, separately-audited operation: `mothership operator rotate-key --new-pub <path>` requires re-auth with the old operator-presence credential, records the rotation in the audit log (`supervisor.operator-key-rotated`), and re-signs all config files with the new key.

### Why operator GPG (not a Mothership CA)

This reuses the key model the operator already uses to sign git commits on this repo. It keeps Mothership out of the CA business (consistent with the plugin-bundle spec's "no Mothership CA" decision) and scopes trust to a single operator-held secret. It also aligns with the UC3 posture: IT teams can standardize on their own internal GPG PKI and use the same key for config signing and plugin-publisher trust roots.

### Parser discipline

All signed YAML is parsed with duplicate-key rejection and strict schema validation after signature verification. Unknown fields fail to parse. This matches the pattern from the plugin-bundle and audit-log specs.

---

## Startup sequence (cold-boot unlock flow)

The supervisor starts before everything. The sequence is fixed and fail-closed at every step.

### Phase 1 — Service init

1. OS service manager starts `mothership-supervisor` under the dedicated service account.
2. Supervisor process reads `/etc/mothership/supervisor.yaml` + signature; verifies signature against pinned operator key; parses config (strict).
3. On any failure: log to stderr + OS service log, exit non-zero, let service manager report failure to operator via platform-native notification. No audit log write is possible yet (audit HMAC key requires keystore unlock).

### Phase 2 — Keystore unlock

4. Supervisor attempts to resolve the handle `mothership/supervisor/service-key` from the secrets bridge.
   - **Linux (libsecret via D-Bus):** if the session D-Bus is not yet running (e.g., no operator logged in), supervisor halts with a documented `keystore-unavailable-linux-session-required` diagnostic. V1 explicitly documents that Mothership on Linux requires an operator session for keystore availability; headless-Linux is V1.5+ (see plugin-bundle `Out of V1`).
   - **macOS (Keychain):** if the Keychain is locked, supervisor surfaces a platform notification requesting unlock; waits up to a configurable timeout (default 300s); halts on timeout.
   - **Windows (DPAPI):** user-profile unlock is implicit at login; for machine-bound DPAPI this phase is a no-op.
5. On keystore-unlock failure: platform notification + OS service log + stderr; exit non-zero.

### Phase 3 — Audit chain bootstrap

6. Resolve `mothership/audit/hmac` via secrets bridge; this is a scoped `SecretBytes` lease per the audit-log spec (default 1s / 1000-emit window).
7. Verify audit chain continuity: open the current audit segment, verify `seg_mac` against the manifest, and either continue the chain (normal boot) or emit `audit.chain-restart` (recovery boot) per audit-log `descriptor-verified`-equivalent rules.
8. Emit `supervisor.started` audit event with supervisor version, config-file fingerprints, and boot reason.

### Phase 4 — Policy + grant ledger load

9. Load and verify `/etc/mothership/policy.yaml` + signature. Parse matrix; compile evaluator.
10. Load and verify the grant ledger at `/var/lib/mothership/grants/ledger.jsonl`; replay entries into an in-memory `GrantSet` keyed by `(skill_id, agent_id, capability, org_context)`; verify the ledger's HMAC chain; on mismatch, halt with `grant-ledger-corrupted` (see "Audit-unavailable and adjacent failure modes" below).

### Phase 5 — Plugin registry load

11. For each bundle under `/var/lib/mothership/bundles/<bundle_hash>/`, re-verify the canonical-path existence and resolve the bundle's descriptor from the install journal; re-hydrate the verified-manifest objects into the in-process manifest registry.
12. Emit `supervisor.plugin-registry-loaded` with counts per org context.

### Phase 6 — Broker start

13. Spawn the broker process (sub-spec 2 defines the broker launch contract) with the supervisor socket as its only control-plane channel. Pass the supervisor's current policy snapshot + manifest registry + grant snapshot + audit writer handle via that channel. Broker refuses to start without these inputs.
14. Emit `supervisor.broker-started`.

### Phase 7 — Agents allowed

15. Only after the broker signals `ready`, agents may spawn on operator demand (or on configured triggers, per their manifests). **Mechanical enforcement:** agent/skill spawns go exclusively through the privileged-spawn helper, which requires a valid supervisor-issued `--challenge-nonce`. Before broker `ready`, the supervisor refuses to mint spawn nonces. Any spawn attempt without a valid nonce → helper refuses with `supervisor-not-ready`, emits `supervisor.spawn-rejected(reason: supervisor-not-ready)`. There is no other spawn path — neither the OS service manager nor the CLI nor an orphaned process from a prior boot can bypass this gate, because none of them hold the helper's authenticated input channel.

16. **Orphan-process cleanup at boot.** Before Phase 7, the supervisor scans for any `mothership-*` labeled processes left running from a prior boot (via cgroup / job-object per platform) and sends `SIGTERM` then `SIGKILL` after grace. Orphans that predate the current supervisor's `snapshot_version` cannot be resumed; they must be re-spawned under the current snapshot. Audit: `supervisor.orphan-process-reaped` per PID.

### Failure of any phase

Any phase that cannot complete → supervisor halts (process exits) with a structured diagnostic written to: (a) stderr, (b) platform-native service log, (c) platform notification, (d) the status-pipe (see §"Status pipe" below). **Mothership never runs with partial enforcement.** If the supervisor can't establish the full stack, it does not start.

---

## Policy engine

### Shape

Policy is a **declarative capability matrix** expressed in YAML:

```yaml
schema_version: 1
matrix:
  # provenance class -> capability class -> decision
  operator-typed:
    shell:exec: allow
    fs:write: allow
    fs:read: allow
    network:outbound: allow
    network:inbound: deny                        # explicit deny
    secret:read: require-grant                   # matrix says allow-if-granted; granted-check happens in broker
    intent:emit: allow
    intent:consume: allow
    messaging:send: allow
  trusted-local:
    shell:exec: require-grant
    fs:write: require-grant
    fs:read: allow
    network:outbound: require-grant
    secret:read: require-grant
    intent:emit: allow
    intent:consume: allow
    messaging:send: require-grant
  web-content:
    shell:exec: deny                             # NEVER, regardless of grants
    fs:write: deny
    fs:read: deny
    network:outbound: deny
    secret:read: deny
    intent:emit: require-grant                   # a web-tagged input can emit an intent only with explicit grant
    intent:consume: deny
    messaging:send: deny
  third-party-message:
    # similar denylist-heavy shape
  skill-output:
    # similar
  supervisor-authored:
    # permissive — supervisor internally speaks with this provenance

# Overrides applied per (org_context, skill/agent id) — rare, heavily audited
overrides:
  - scope: {org_context: "apisec", agent_id: "apisec-scan-executor"}
    matrix:
      skill-output:
        shell:exec: require-grant                # loosened for this one controlled agent
    rationale: "APIsec security scan executor runs tool chains derived from scan results under strict per-scan grants"
    audit_required: true
```

### Decision values

- `allow` — permitted unconditionally. Broker still checks that the capability is declared in the bundle manifest; matrix `allow` only lifts the provenance gate.
- `deny` — rejected regardless of grants. Provenance-class deny beats any grant (no grant can override a matrix deny — this is what P4's "web content cannot cause shell execution regardless of what the model decides" enforces).
- `require-grant` — permitted iff a corresponding grant exists in the ledger (and is non-expired and matches the `(skill_id, agent_id, capability, org_context)` tuple). Evaluated at broker-intent-routing time.

### Authority & mutation

The matrix is **supervisor-owned, signed, re-auth-gated, audit-logged**. There is no agent-reachable API that modifies the matrix. Mutation follows the two-phase commit protocol defined in §"State versioning and broker coordination" below; the matrix is part of the `SupervisorSnapshot` and cannot update in isolation.

### Monotonic-lock baseline (embedded, authoritative)

Mothership ships a baseline deny set **compiled into the supervisor binary** at `policy-baseline.yaml` (embedded via `include_str!`). The baseline declares the mandatory-`deny` cells that no operator-authored `policy.yaml` may narrow. On every policy load and reload, the validator computes `proposed_matrix.deny_cells ⊇ baseline.deny_cells`. If any baseline-required `deny` cell is narrowed to `allow` or `require-grant` in the proposed matrix, reload fails with `policy-monotonic-lock-violation` (audited) and the old matrix persists.

The baseline (V1) covers **all untrusted-or-derived provenance classes × all high-risk capabilities**:

| Provenance (mandatory deny at) | Capabilities (all `deny`, cannot narrow) |
|---|---|
| `web-content` | `shell:exec`, `fs:write`, `network:outbound`, `secret:read`, `messaging:send`, `policy:mutate`, `grants:mutate`, `install:mutate` |
| `third-party-message` | same as `web-content` |
| `skill-output` (untrusted derivation) | `shell:exec`, `secret:read`, `policy:mutate`, `grants:mutate`, `install:mutate` |
| `browser-origin-callback` (new class — see below) | same as `web-content` |
| `cli-uploaded-file` (content of bundles-in-transit etc.) | `shell:exec`, `secret:read`, `policy:mutate` |
| `restored-backup` | `shell:exec`, `secret:read`, `policy:mutate`, `install:mutate` |
| `audit-query-output` | `shell:exec`, `fs:write`, `secret:read`, `messaging:send`, `policy:mutate`, `grants:mutate`, `install:mutate` |
| `hook-output` (sandboxed hook scripts returning decisions) | `shell:exec`, `fs:write`, `secret:read`, `messaging:send`, `policy:mutate`, `grants:mutate`, `install:mutate` |
| `cross-agent-handoff` (from another agent — carries the handing-off agent's provenance in its lineage) | baseline inherits the **most-untrusted** provenance class in the lineage (propagation rule enforced by broker, baseline here requires that propagation) |

Narrowing the baseline beyond these cells (adding more denies) is permitted and requires re-auth + re-sign; broadening is not permitted without a supervisor major-version bump with explicit operator ceremony (V1.5+).

### Closed provenance taxonomy + fail-closed for unknown

The provenance classes enumerated in the matrix are the **complete closed set** for V1:

`operator-typed`, `trusted-local`, `web-content`, `third-party-message`, `skill-output`, `supervisor-authored`, `browser-origin-callback`, `cli-uploaded-file`, `restored-backup`, `audit-query-output`, `hook-output`, `cross-agent-handoff`.

Codegen emits a closed Rust enum and matching TS/Python types. **Any intent reaching broker without a valid provenance tag is rejected as `provenance-unknown`** (baseline-deny for unclassified provenance across all capabilities). Adding a new provenance class is a supervisor major-version bump; until then, the taxonomy is frozen and unclassified content is deny-by-default.

### Evaluator

The evaluator's job is mechanical: given an `(intent, provenance, actor, snapshot_version)` tuple and the current grant set, return one of `{allow, deny, require-operator-reauth}`. The evaluator is **supervisor-owned code** but the broker holds a compiled copy tied to a specific `snapshot_version`; broker evaluates at routing time to avoid a supervisor round-trip per intent. The broker MUST tag each intent-admission / intent-denial audit event with the `snapshot_version` used. See §"State versioning and broker coordination".

---

## State versioning and broker coordination

Codex's top-priority finding: "the broker holds a compiled copy to avoid a per-intent round-trip" is correct as an architecture, but every supervisor-broker coordination point needs an explicit versioning protocol or stale-broker state becomes the cheapest production failure.

### `SupervisorSnapshot`

A single immutable object the supervisor builds on each mutation:

```
SupervisorSnapshot {
  snapshot_version: u64,                     // monotonic, bumped on every change
  snapshot_digest: [u8; 32],                 // SHA-256 of canonical_json of the fields below
  policy_fingerprint: [u8; 32],              // SHA-256 of canonical_json of the compiled matrix
  policy_source: { path, signature_fp },
  grant_ledger_seq_and_mac: (u64, [u8; 32]), // latest durable ledger entry
  manifest_registry_version: u64,            // bumped on add/remove/deactivate
  org_context_registry_version: u64,
  baseline_fingerprint: [u8; 32],            // embedded baseline SHA (sanity pin)
  built_at: RFC3339,
  audit_seq_at_build: u64,                   // the audit seq after which this snapshot is active
  prior_snapshot_version: Option<u64>,       // None for first post-boot snapshot
}
```

### Two-phase commit (prepare → ack → activate)

Every mutation that the broker must observe consistently goes through this exact protocol:

1. **Prepare.** Supervisor writes durable journal entries for the mutation (grant-ledger append, manifest-registry mutation, org-context mutation, or policy reload). Fsync. Build the new `SupervisorSnapshot`.
2. **Broker-ack.** Supervisor sends the new snapshot over the supervisor→broker control channel (authenticated by the broker-control HMAC key). Broker validates the snapshot's `snapshot_digest`, `policy_fingerprint`, `baseline_fingerprint` (must match broker's embedded baseline), and that `prior_snapshot_version` equals the broker's currently-active version. Broker compiles its view, emits its own `broker.snapshot-accepted` local audit line, and sends `ack(snapshot_version)` back to supervisor.
3. **Activate.** Supervisor receives ack. Records activation in the journal; emits the relevant `plugin.install-approved` / `policy.matrix-updated` / `policy.grant-created` / etc. audit event with `snapshot_version`. Serves the read APIs (`GET policy/current`, `GET grants/current`) from the new snapshot. Older reads continue to see the prior snapshot until the transition completes.

### Broker-ack failure handling

- **Timeout (default 10s, configurable):** supervisor aborts the activation. Journal records `supervisor.broker-ack-timeout`. The prepared mutation is **rolled back** (grant append is marked invalid via a compensation entry — see §"Grant-ledger recovery"; manifest registry reverts; policy reload aborts). Prior snapshot remains active; audit entry emitted with `snapshot_version = previous`.
- **Broker rejects (digest mismatch, baseline mismatch, prior-version mismatch):** treated as broker corruption; supervisor restarts the broker (counts against broker-restart budget) and the mutation is retried from step 1. Three consecutive rejection → broker-restart-budget-exhausted halt.
- **Broker ack with wrong snapshot_version:** hard reject; broker is treated as faulted; restart.
- **Supervisor crash after ack but before activate:** on restart, supervisor sees the ack recorded in the journal but no activation; replays the activation. Broker-side snapshot persistence is not durable across broker restart; the supervisor re-pushes the snapshot to the freshly-started broker as part of Phase 6 boot.

### Broker-originated audit must carry snapshot_version

Broker's `broker.intent-admitted` and `broker.intent-denied` events (owned by sub-spec 2) MUST carry `snapshot_version` so the audit chain lets forensic analysis pin which policy/grant version was in effect for any given decision. This is normative for sub-spec 2 and recorded here as a cross-spec requirement.

---

## Grant ledger

### Schema

One grant = one JSONL line, HMAC-chained with the same key family as the audit log (distinct sub-key `mothership/grants/hmac`; resolved via secrets bridge). The ledger is at `/var/lib/mothership/grants/ledger.jsonl` with manifest `ledger.manifest.jsonl`.

```json
{
  "seq": 42,
  "ts_wall": "2026-04-19T14:32:00Z",
  "ts_mono_ns": 1234567890,
  "key_epoch": 1,
  "op": "grant-create" | "grant-revoke" | "grant-use-recorded",
  "grant_id": "grt_<uuid>",
  "actor": { "kind": "supervisor", "id": "main" },
  "org_context": "ttl",
  "scope": {
    "skill_id": "tamper-tantrum-labs/inbox-triage#gmail-mcp",
    "agent_id": "tamper-tantrum-labs/inbox-triage#inbox-triage-reader",
    "capability": "network:outbound",
    "capability_constraints": { "host": "gmail.googleapis.com", "port": 443 }
  },
  "authority": {
    "operator_identity_fingerprint": "BB28AA4B...",
    "reauth_challenge_id": "<challenge id from sub-spec 7>",
    "bundle_hash": "<bundle_hash that declared this capability>"
  },
  "expires_at": "2026-07-19T14:32:00Z",
  "revocation": null,                         /* present on grant-revoke ops; null on grant-create */
  "prev_mac": "<hex>",
  "mac": "<hex>"
}
```

### Chain semantics

Same rules as the audit log:

- `prev_mac` chains one entry to the previous.
- `key_epoch` increments on HMAC-key rotation (tied to the same secrets-bridge import/recovery flow).
- Genesis MAC binds to the install fingerprint + `key_epoch` so two installs' ledgers cannot be spliced.
- Segment rotation follows audit-log rules (100 MiB default; 24h time; operator roll via `mothership grants roll`).
- `mothership grants verify` recomputes the chain; mismatch is a hard halt per "Grant-ledger corruption" below.

### In-memory projection

On boot, the ledger is replayed into an in-memory `GrantSet` keyed by `(skill_id, agent_id, capability, org_context)`. Active-grant lookup is O(1). Grant-revoke entries remove the active-grant row. Grant-use-recorded entries are informational (used for telemetry, not for eligibility — the broker checks presence + expiry at use time).

### Grant mutation authority

**Every grant-create and grant-revoke op requires operator-presence re-auth via sub-spec 7, recorded with `reauth_challenge_id`.** There is no agent-reachable path to mutate grants; the CLI and the plugin-install approval UI are the only mutation entry points.

### Grant read API

Broker (T3) and Skill process isolation (T4) consume the current `GrantSet` via a supervisor-exposed read API over the unix socket. All reads are tagged with the `snapshot_version` they were served under:

- `GET grants/current?scope=<skill_id,agent_id,capability,org_context>` → `{presence, expiry, constraints, snapshot_version}`.
- `GET grants/snapshot` → full `SupervisorSnapshot` header + `GrantSet` serialization for a fresh broker rehydrate.
- `SUBSCRIBE grants/changed` → event stream carrying `(new_snapshot_version, diff)`; broker must consume in order and error-close the subscription on any gap.

### Tentative-vs-durable grant entries

Because install `committed` uses the two-phase commit protocol, grant-create entries written during the Prepare phase are marked **tentative** until broker acknowledges. Schema extension:

```json
{
  "seq": 42,
  "op": "grant-create-tentative" | "grant-create" | "grant-create-compensated" | "grant-revoke" | "grant-use-recorded",
  ...
}
```

- `grant-create-tentative` — written during Prepare; broker does not yet observe these grants.
- `grant-create` — written as a post-ack promotion entry referencing the tentative seq; broker observes from the promoted snapshot forward.
- `grant-create-compensated` — written on ack timeout/reject; references the tentative seq and marks it invalid. `GrantSet` replay treats the tentative as never-created if a compensation entry exists for it.

Replay on boot is deterministic: the in-memory `GrantSet` contains only grants whose tentative was promoted (i.e., followed by a `grant-create` referencing it and not superseded by a `grant-create-compensated`) and not revoked. Orphan tentatives (tentative + crash before promotion or compensation) are explicitly handled by the recovery algorithm below.

### Grant-ledger recovery (normative algorithm)

Triggered by: (a) `mothership grants recover --from <segment>` (operator-initiated post-corruption), (b) HMAC chain verification failure at boot, (c) orphan-tentative discovered at boot replay.

**Recovery contract:**

1. **Re-auth gate.** Recovery requires operator-presence re-auth via sub-spec 7; the challenge's `reauth_challenge_id` is recorded in the recovery markers written below.
2. **Quarantine corrupt segment.** Move the failing segment file to `/var/lib/mothership/grants/quarantine/<timestamp>-<seg>.jsonl`. The segment remains available for forensic inspection and is **not replayed**.
3. **Emit chain-restart marker.** Write a new `grants.chain-restart` entry (schema: same fields as audit log's chain-restart with `restart_reason: "operator-recover" | "orphan-tentative-reconciliation" | "hmac-corruption"`, `prior_closing_mac`, `new_genesis_mac`, `new_key_epoch` incremented). This entry is the first of the post-recovery chain.
4. **Reconcile orphan tentatives** (if applicable). For each orphan tentative (tentative not followed by promotion or compensation): write a `grant-create-compensated` entry marking it invalid and including `reason: "orphan-tentative-at-recovery"`. Operator is informed in the recovery CLI output which bundles had orphan tentatives.
5. **Operator choice: replay-from-audit or clean-slate.**
   - **Replay-from-audit** (default when feasible): the supervisor walks the audit log's `plugin.install-approved-decision` events from the earliest install forward, cross-referenced with `plugin.install-activated` events, and re-creates grant-ledger entries matching the audit record. Each replayed grant is annotated `replay_source: audit_seq:<n>` so the operator can distinguish original from replayed.
   - **Clean-slate**: the new chain starts with zero grants. All installed bundles continue to exist but their grants are revoked; operator must re-grant each.
6. **Emit `supervisor.grants-recovered`** audit event with `recovery_mode`, `orphan_tentatives_count`, `replayed_grants_count`, `quarantined_segment`.

Tests for this algorithm are in L3 below (corrupted-segment recovery, orphan-tentative reconciliation, replay-from-audit correctness, clean-slate correctness).

---

## Org-context registry

### Shape

```yaml
# /etc/mothership/org-contexts.yaml (signed)
schema_version: 1
contexts:
  - id: "ttl"
    display_name: "TamperTantrum Labs"
    unix_user: "mothership-ttl"
    home: "/var/lib/mothership/org/ttl"
    inherited_capabilities: []                   # V1: none; reserved for V1.5+
  - id: "apisec"
    display_name: "APIsec"
    unix_user: "mothership-apisec"
    home: "/var/lib/mothership/org/apisec"
    inherited_capabilities: []
  - id: "shared"
    display_name: "Shared"
    unix_user: "mothership-shared"
    home: "/var/lib/mothership/org/shared"
    inherited_capabilities: []
```

Each org context is mapped to a **dedicated unix user** (Linux/macOS) or **local security principal** (Windows). Skill processes for a given org context are spawned `setuid`'d to that user; filesystem reads/writes for that context target `home/**`. This realizes P1 (capability separation) at the OS boundary: TTL-context skills cannot read `/var/lib/mothership/org/apisec/**` because fs permissions forbid it.

### Resolution API

Broker and skill-isolation both need to resolve `org_context_id → unix_user` at spawn time. Supervisor exposes:

- `GET org-contexts/<id>` → full record.
- Cached in broker's compiled view; refreshed on `supervisor.org-context-registry-updated` events.

### Registry mutation

Same authority model as policy + grants: signed file + operator-reauth on reload. Adding/removing an org context triggers an audit entry `supervisor.org-context-added` or `supervisor.org-context-removed`. Removing an org context **does not automatically delete its `home/**`** — operator-gated cleanup via `mothership org-context cleanup <id>` (re-auth gated, audited), because UC1's "per-client audit report as security attestation" requires that context data be retainable after deactivation until the operator explicitly destroys it.

---

## Plugin install orchestration

The plugin-bundle spec owns the install state machine and the cryptographic verification. The supervisor's role during the install is to drive the `reviewed` state and to persist the outcome.

### Supervisor responsibilities at each install-state transition

| Install state | Supervisor action |
|---------------|-------------------|
| `received` | Emit `plugin.install-requested`. Write journal entry for the install_event_id. |
| `descriptor-verified` | Emit `plugin.signature-verified` or `plugin.signature-failed`. Stage the descriptor in-process for the next state. |
| `reviewed` | Present capability-review UI over the CLI or gateway. Require re-auth via sub-spec 7. On **successful** re-auth + operator approval: **immediately emit `plugin.install-approved-decision`** (new audit event, described below) with the full grants-approved list, operator identity fingerprint, and `reauth_challenge_id` — this is durable audit of the decision itself, distinct from the later `plugin.install-activated` event. On timeout/rejection → emit `plugin.install-rejected`. |
| `staged` | Supervise payload materialization in `installs/<install_event_id>/payload/` (per plugin spec). No grant mutation yet. |
| `committed` | Two-phase commit via §"State versioning and broker coordination": **Prepare** — atomically rename payload to `bundles/<bundle_hash>/`; write grant-create entries to the grant ledger (tentative — see below); mutate the manifest registry in a staging buffer; build new `SupervisorSnapshot`. **Broker-ack** — push new snapshot; await ack. On **ack**, commit grant-ledger entries (promote tentative to durable). On **timeout/reject**, compensation: revert manifest registry; emit `policy.grant-revoked` compensation entries for the tentative grants (audit hygiene: compensation entries reference the original tentative seq range); rollback payload rename to `rolled-back-bundles/<bundle_hash>/`; emit `plugin.install-rejected(reason_code: broker-ack-failed)`. |
| `activated` | Emit `plugin.install-activated` audit event with `snapshot_version`, bundle_hash, and `approval_decision_seq` (backref to the `plugin.install-approved-decision` event). This event marks the point after which broker admits intents naming this bundle's skills/agents. |

### Why approval audit at `reviewed`, activation audit at `activated`

Codex's review flagged the risk that if the install fails between `reviewed` and `activated`, the operator's approval decision never becomes durable audit — a P8 hole. Moving the approval-decision audit to immediately after re-auth closes that hole: the operator's decision is always on record independent of whether staging/commit/activation succeed. `plugin.install-activated` then separately records what the system actually did with that approval, carrying the `snapshot_version` and a backref to the approval seq. If activation fails, the sequence becomes `plugin.install-approved-decision` (durable) → `plugin.install-rejected(reason: broker-ack-failed)` (durable) with no orphan approval.

### Review UI (V1 CLI)

```
mothership plugin install <path-to-mship>
```

The CLI connects to the supervisor socket, uploads the bundle, and drives the install state machine. During `reviewed`, the CLI renders:

```
Installing bundle: tamper-tantrum-labs/inbox-triage@1.4.2
Publisher: tamper-tantrum-labs (TamperTantrum Labs) [verified publisher via operator-trust root TTL-Prod]
Bundle hash: sha256:a9b3...
Identity method: operator-trust
Declared capabilities:
  skill "gmail-mcp":
    [ ] network:outbound (gmail.googleapis.com:443)
    [ ] secret:read (mothership/user/gmail-oauth)
    [ ] intent:emit (mail.draft, mail.read, mail.list)
  agent "inbox-triage-reader":
    [ ] intent:emit (mail.draft)
    [ ] intent:consume (mail.received, mail.triage-request)
    [ ] skills_invoked (gmail-mcp)
    [ ] agents_handoff (inbox-triage-drafter)
  hook "pre-send-review":
    [ ] decision_schema (allow, deny, revise)

Select capabilities to grant (space to toggle; every grant is individually audited):
[ ... operator interacts ... ]

Confirm install? Type the bundle_id to confirm:
> tamper-tantrum-labs/inbox-triage
```

The CLI then requires operator re-auth (Touch ID / Windows Hello / YubiKey / PAM) via sub-spec 7. On success, the supervisor proceeds through `staged → committed → activated`.

### No trust-level auto-grant

Per plugin-bundle spec V0.5+, trust level is **badge-only** in V1. The UI displays the badge (`verified publisher`, `community publisher`, etc.) but defaults every grant to **off**. Operators toggle each grant deliberately.

### Bundle-in-transit integrity

The bundle file uploaded to the supervisor is stored first in the quarantine directory for its install_event_id. Any tamper en route (e.g., a compromised CLI) would show as a signature failure at `descriptor-verified`. The supervisor does not trust the CLI's "pre-check" claims about the bundle.

---

## Operator-presence orchestration

### Delegation model

Sub-spec 7 owns the **primitives** (Touch ID / Windows Hello / YubiKey / PAM challenge-response; session tokens; origin allowlist). The supervisor owns the **policy** of which operations require re-auth and what happens on timeout or rejection.

### V1 policy (which ops require re-auth)

Any write to signed config, any grant mutation, any plugin install, any secret-bridge privileged op (enroll/export/import/delete), any `policy reload`, any `trust add/remove`, any `operator rotate-key`, any `org-context` mutation, any `audit roll` / `audit recover` / `grants roll`, and any plugin `rollback` / `revoke`.

### Challenge flow

1. Supervisor creates a challenge via sub-spec 7's gateway (`POST /challenges`) with `{requested_action, scope, ttl}`.
2. Gateway returns a `challenge_id` and surfaces the prompt to the operator via the platform-native primitive.
3. Operator responds (positive: provides biometric / YubiKey touch; negative: rejects; timeout: gateway returns `timeout`).
4. Gateway returns the verdict to the supervisor.
5. Supervisor records the `challenge_id` in the corresponding ledger/audit entry (the entry's `authority.reauth_challenge_id` field).

### Rejection / timeout policy

- **Timeout** (default 120s per challenge; operator-configurable per requested_action class): op aborts with `operator-timeout`; no side effects; `<op>-rejected` audit event emitted.
- **Explicit rejection**: same as timeout in effect; distinct `reason_code` in the audit entry.
- **Sub-spec 7 gateway unreachable**: supervisor halts the requesting op with `gateway-unreachable`; alerts operator via status pipe; **does not degrade** to a non-re-auth path.

### Batch review (out of V1)

UC3 may eventually want batched approval for IT-deployed bundles ("approve all bundles signed by root `TTL-IT-Prod` for the next hour"). V1 does NOT support batching — every privileged op is individually challenged. This is an explicit friction cost that realizes P6 at the UX layer; Codex's earlier rubber-stamping warning in the plugin-bundle review applies here too.

---

## Shutdown / scrub

### Graceful stop (SIGTERM / service-manager stop)

1. Supervisor emits `supervisor.stopping` with reason.
2. Broker is sent a `stop` signal over its control channel; broker drains in-flight intents, stops agents, emits `broker.stopped`.
3. Supervisor's in-memory `GrantSet`, manifest registry, and policy-matrix are dropped; associated `SecretBytes` leases are explicitly zeroized (not awaiting Drop).
4. Audit writer flushes + syncs; final `supervisor.stopped` event written + fsynced.
5. Status pipe updated to `stopped`.
6. fd handles to log / grants / journal are closed.
7. Process exits 0.

### Forced kill (SIGKILL / service crash)

All on-disk state is consistent because:
- Audit log is HMAC-chained JSONL with per-entry fsync; a mid-write crash leaves a truncated segment that `audit verify` detects at recovery and handles per the audit-log spec.
- Grant ledger follows the same pattern.
- Journal is per-install JSONL with per-transition fsync.
- Bundle payload writes use temp-path + rename; a kill mid-rename either leaves the old canonical path intact or the new rename completed.

No scrub happens on SIGKILL — that's the OS's job (process memory is freed by the kernel; secret bytes in memory at kill time are not guaranteed zeroized). This is an accepted trade-off; P5 compliance is about not persisting secrets, not about defending against a local-root kernel-level attacker.

---

## Audit-unavailable and adjacent failure modes

Supervisor runtime failures that trigger fail-closed:

| Condition | Response |
|-----------|----------|
| Audit writer cannot persist (per audit-log spec) | `audit-unavailable` mode — per audit-log downstream contract (all grants frozen, installs denied, etc.) |
| Grant ledger HMAC chain fails verify at boot | Hard halt with `grant-ledger-corrupted` diagnostic. Operator must run `mothership grants recover --from <segment>` (re-auth gated, emits `supervisor.grants-recovered` audit event). |
| Policy matrix file signature invalid | Hard halt; if the failing file is the one currently loaded (reload path), supervisor continues running under the old in-memory policy and emits `policy.reload-failed`; it does not silently downgrade. |
| Operator key fingerprint mismatch at config load | Hard halt at boot; reload-path treats it as a rejection (old policy persists); both emit `supervisor.operator-key-mismatch`. |
| Secrets bridge unavailable mid-run | Supervisor transitions to `audit-unavailable` mode (since HMAC key leases cannot renew) and enters degraded ops: serve read APIs from last-known state; reject any mutation. |
| Broker crash | Supervisor restarts broker up to N times (default 5) with exponential backoff; on budget exhaustion, supervisor halts and requires operator intervention. Each broker (re)start emits `supervisor.broker-started` / `supervisor.broker-restart-budget-exhausted`. |
| Skill-process crash | Handled by sub-spec 4; supervisor only records the crash audit event and updates its in-memory skill-registry state. |

### Status pipe

A fixed-path memory-mapped file at `/var/run/mothership/status` (platform-equivalent) holds a tiny struct updated at every state transition: `{version, state: "starting"|"ready"|"degraded"|"audit-unavailable"|"stopping"|"stopped", last_success_ts, last_error, last_error_ts}`. Read by `mothership status` CLI without any broker / audit dependency. This is the out-of-band signal path referenced by the audit-log spec for `audit-unavailable` notifications.

---

## Test strategy (4 levels)

### L1 — Schema + type-level

- Config schemas (supervisor.yaml, policy.yaml, org-contexts.yaml) are generated from a source-of-truth and parsed with strict validation + duplicate-key rejection.
- Grant-ledger entry shape is a closed union emitted by codegen; unknown `op` types fail to compile (Rust) and fail runtime parse (TS/Python).
- Policy-matrix evaluator is a closed state machine over declared provenance × capability classes; adding a new class is a Mothership-version bump, not a config extension.
- Canonical-JSON parity for grant-ledger MACs uses the same golden-vector suite as audit log and plugin descriptor (cross-spec correctness contract).

### L2 — Unit tests

**Service lifecycle:**
- Unit starts under the dedicated service account; fails to start as root with a diagnostic; `NoNewPrivileges` verified via `/proc/<pid>/status` on Linux.
- Windows service runs under `NT SERVICE\MothershipSupervisor`; SID deny-list verified.
- macOS daemon runs under `_mothership`; sandbox-exec policy rejects disallowed syscalls.

**Config integrity:**
- Valid signature + valid YAML → accepted; emits `policy.matrix-updated`.
- Valid signature + schema-invalid YAML → rejected; hard halt on boot, old-policy-retained on reload.
- Invalid signature → rejected at boot.
- Tampered `.sig` file → rejected at boot.
- Operator-key rotation flow end-to-end: old key signs old config; rotate; new key signs new config; old-key-signed files fail after pin swap.

**Startup sequence fault injection:**
- For each phase (service init, keystore unlock, audit bootstrap, policy load, ledger replay, registry load, broker start): inject a failure; verify supervisor halts with the right diagnostic; verify status pipe reflects the halt state.
- Platform-specific: Linux-session-absent → keystore-unavailable diagnostic; macOS-keychain-locked → prompt + timeout.

**Policy engine:**
- Matrix with every provenance × capability combo — evaluator returns correct decisions.
- `web-content.shell:exec: allow` attempt at reload → rejected with `policy-monotonic-lock-violation`.
- `web-content.shell:exec: deny` removed at reload → same rejection.
- Override scope matching: `(org_context, skill_id, agent_id)` triple is resolved correctly; overrides apply only within their declared scope.
- Reload flow: emit `policy.matrix-updated`; broker receives updated snapshot; in-flight evaluations continue under old matrix; new evaluations use new.

**Grant ledger:**
- Grant create / revoke / use-recorded: each produces the correct entry shape + HMAC.
- Chain verify round-trip: 10,000-entry ledger verifies in under 1 second (perf budget).
- Tamper detection: edit-in-place / drop / insert / reorder / truncate / forge-re-MAC scenarios all fail `grants verify`.
- Revocation is idempotent: revoking an already-revoked grant emits an audit warning but does not fail.
- Expiry: broker's read API returns the correct active-vs-expired status for a given `(skill, agent, capability, org)` tuple.
- Replay-on-boot produces an in-memory `GrantSet` byte-identical across runs from the same ledger.

**Org-context isolation:**
- Skill spawned for context `ttl` cannot read `/var/lib/mothership/org/apisec/**` — verified at OS boundary (`EACCES`), not at supervisor-app layer.
- Removing an org context preserves its `home/**` until operator runs `cleanup`.
- `cleanup` emits `supervisor.org-context-cleaned` and is re-auth gated.

**Re-auth orchestration:**
- Every op in the V1 "requires re-auth" list triggers a challenge; on timeout/rejection, the op aborts and no side effects persist.
- Gateway unreachable → supervisor halts the op; status pipe reflects `gateway-unreachable`.
- Batch approval is rejected (V1 non-goal): an operator attempt to approve multiple pending challenges with a single response is denied with `batch-approval-not-supported`.

### L3 — Integration tests with T1 dependencies and broker coordination

**Happy path:**
- Full install flow: plugin-bundle spec's end-to-end install runs through the supervisor; all six plugin install states transition correctly; audit chain is intact with `plugin.install-approved-decision` (at `reviewed`) + `plugin.install-activated` (at `activated`) present and correlated by `approval_decision_seq`; grant ledger reflects approved capabilities with promoted-from-tentative entries; broker's `broker.snapshot-accepted` ack is recorded.
- Plugin bundle install + use: install a test bundle → grant capability → broker admits corresponding intent → skill executes with declared caps → audit chain correlates install, grant, intent, each tagged with the same `snapshot_version`.

**Broker-ack failure matrix (Codex priority):**
- **Broker ack times out at `committed`.** Supervisor aborts activation; compensation entries written for tentative grants; bundle payload moved to `rolled-back-bundles/`; `plugin.install-rejected(reason: broker-ack-failed)` emitted. Prior snapshot remains active. No bundle state observable to broker.
- **Broker rejects snapshot with digest mismatch at `committed`.** Broker is restarted (counts against budget); mutation retried. Integration verifies 3 consecutive rejections trigger `broker-restart-budget-exhausted` halt.
- **Broker ack succeeds but supervisor crashes before activate-audit.** On restart, supervisor replays from journal; sees ack recorded; replays activation; emits `plugin.install-activated` after restart. Audit chain is contiguous and referencing the right `snapshot_version`.
- **Broker crashes after ack but before supervisor activate.** Supervisor sees ack; proceeds to activate; broker restarts; supervisor re-pushes snapshot to fresh broker during Phase 6; broker catches up before agents are allowed.
- **Grant-revoke mid-flight with concurrent broker read.** Revoke in progress; broker issues `GET grants/current`; broker sees old snapshot's grant (still active) until two-phase commit completes and new snapshot is acknowledged; verify no flap.
- **Policy reload during in-flight plugin install.** In-flight install holds the prior snapshot for its lifecycle; policy reload completes on a new snapshot; install activation is then re-acked against the current snapshot.
- **Stale-broker differential test.** Force a broker to retain an old compiled matrix by intercepting the snapshot-push; verify supervisor's ack timeout fires and broker is restarted; verify broker cannot admit intents based on the stale view.

**Grant-ledger correctness:**
- Audit fail-closed cascade: simulated disk full → supervisor transitions per audit-log spec's downstream-contract table; new installs denied, new grants denied, in-flight revocable tool invocations blocked.
- Secrets bridge audit-key rotation: rotating `mothership/audit/hmac` via secrets import → supervisor re-audit-bootstraps → new key_epoch in audit log; **grant ledger's `mothership/grants/hmac` is independent and is NOT rotated in lockstep** (test enforces separation — distinct key rotations produce distinct chain-restart markers in the two chains).
- Secrets bridge grant-key rotation: rotating `mothership/grants/hmac` → supervisor emits `grants.chain-restart` with new key_epoch; pre-rotation entries verify under old key; post-rotation under new.
- Simultaneous audit-key + grant-key rotation: two distinct `audit.chain-restart` + `grants.chain-restart` markers; no cross-contamination; both recoveries independent.
- Grant-ledger corruption at boot: inject a MAC mismatch mid-chain; boot halts with `grant-ledger-corrupted`; operator-initiated `grants recover --from <segment>` succeeds; recovery emits `grants.chain-restart` + `supervisor.grants-recovered`; post-recovery `GrantSet` matches the replay-from-audit reconstruction exactly.
- Orphan tentative at boot: simulate a crash between tentative grant append and broker ack; on boot, reconciliation emits `grant-create-compensated` entries for each orphan; post-replay `GrantSet` has zero orphan-tentative-derived grants.
- Clean-slate recovery: operator chooses clean-slate instead of replay-from-audit; new chain starts empty; existing bundles remain installed but with zero grants; operator must re-grant each bundle's capabilities to restore functionality.

**Race and concurrency:**
- Concurrent grant-create + grant-revoke on the same scope → both complete in serial order per the supervisor mutation lock; final state reflects the latest op; subscription events arrive in commit order.
- Concurrent privileged ops each requesting re-auth → each gets its own distinct `challenge_id`; no challenge is shared; no challenge approval can be reused across ops (replay-protection test).
- Challenge replayed after timeout → gateway rejects replay with `challenge-expired` / `challenge-already-used`; supervisor treats as a fresh op.
- Challenge approved after timeout + supervisor-abort crossover → supervisor's abort cleanup races gateway's late-approval delivery; supervisor's abort is authoritative; late approval is discarded with `challenge-abandoned` audit.

**Shutdown in-flight:**
- SIGTERM during `reviewed` with pending re-auth → supervisor cancels the challenge (`challenge-abandoned`), emits `plugin.install-rejected(reason: supervisor-shutdown)`, discards the install_event; operator sees the aborted install on restart with clear cause.
- SIGTERM during `committed` Prepare phase → tentative grant append is NOT yet durable; rollback is trivial; on restart, replay sees no tentative to reconcile.
- SIGTERM during `committed` broker-ack wait → supervisor waits up to grace period for ack; if ack arrives, completes; if not, aborts with compensation (same as broker-ack-timeout path).
- SIGKILL during grant-ledger append → audit-log rules apply: partial JSONL line detected at boot verify; supervisor halts with `grant-ledger-corrupted`; operator runs recovery.
- SIGKILL during audit-log append → same treatment at audit layer; supervisor halts with audit-recovery path invoked.

**Broker restart within budget:**
- Kill broker N times within budget → supervisor restarts; N+1 → halts. Each restart re-pushes the current `SupervisorSnapshot`; broker accepts before agents are allowed.

### L4 — Acceptance tests (CI gates; map the spec-specific acceptance test)

The map's spec-specific acceptance test: *"A compromised agent process cannot rewrite policy, grant itself new capabilities, or alter the audit log; every such attempt is blocked at the supervisor boundary and produces an audit entry."*

**Normative:** every blocked attempt — OS-level or supervisor-level — MUST produce a correlated audit entry. This closes a gap Codex flagged: OS-level `EACCES` alone provides enforcement but zero forensic visibility. The broker's intent router and supervisor socket both emit audit entries on blocked attempts whose actor is observable to Mothership.

Expanded to 20 scenarios — a simulated-compromised-agent harness attempts each attack; supervisor MUST block and MUST log.

**Direct-attack scenarios (agent-process identity):**

1. **Policy file write attempt.** Agent tries `open(/etc/mothership/policy.yaml, O_WRONLY)`. OS denies. Audit: `broker.intent-denied` at the broker's fs-intent boundary (the agent's fs-write intent hit the matrix before even reaching the syscall); AND the broker emits `broker.direct-syscall-observed` if the seccomp/sandbox profile catches a bypass attempt.
2. **Policy reload via supervisor socket from agent credentials.** Agent tries to connect to `/var/run/mothership/supervisor.sock`. Socket is owned by `mothership`. Peer-credential check rejects. Audit: `supervisor.peer-credential-rejected` with agent's observed UID/org_context.
3. **Forged supervisor-socket message mimicking `policy reload`.** Same as #2; peer-credential check happens before any message parse.
4. **Grant self-create via ledger-file direct write.** Agent tries `open(/var/lib/mothership/grants/ledger.jsonl, O_APPEND)`. OS denies. Audit: `broker.intent-denied` at fs-intent boundary.
5. **Audit log direct write.** Same as #4 against the audit log.
6. **Audit log fs race / overwrite.** Agent tries to rename / unlink / overwrite an audit segment file. Supervisor holds `flock`/`FILE_SHARE_READ`-only; OS denies. Audit: `supervisor.peer-credential-rejected` if race-targeted via socket, otherwise broker-intent-denied at attempted-rename boundary.
7. **Plugin install bypass.** Agent tries to write directly into `/var/lib/mothership/bundles/<bundle_hash>/`. OS denies. Audit: `broker.intent-denied`. Even if OS perms were wrong, the supervisor's registry would not accept an unregistered `bundle_hash` at next policy reload.
8. **Operator-reauth bypass.** Agent emits an intent that would require re-auth, hoping to race the check. Broker's evaluator gates on matrix → grants → per-op re-auth; no step is skippable. Audit: `broker.intent-denied(require-operator-reauth)`.
9. **Config signature strip.** Agent tries to delete the `.sig` file. OS denies. Supervisor's loader requires `.sig` present — missing-sig at boot/reload is `config-signature-missing`, never "assume unsigned." Audit: `supervisor.config-signature-missing` at next load attempt.
10. **Status-pipe forgery.** Agent tries to write to the status pipe. OS denies. Audit: `broker.intent-denied` at fs-intent boundary.
11. **Broker-control channel injection.** Agent tries to send messages on the supervisor→broker control channel. Broker authenticates messages via the broker-control HMAC key held only by supervisor; agent's forged messages fail HMAC verification. Audit: `broker.peer-credential-rejected`.

**OpenClaw-class indirect scenarios:**

12. **Browser-origin attack against the gateway.** A malicious webpage the operator is visiting fires cross-origin requests at sub-spec 7's gateway attempting to approve a pending challenge. Gateway's origin allowlist rejects. Audit: `gateway.origin-rejected` with the offending origin; supervisor's challenge remains pending.
13. **Compromised CLI uploading a malicious bundle.** A supply-chain-compromised `mothership` CLI binary uploads a tampered bundle. The supervisor does not trust the CLI's claims; runs the full plugin-bundle install state machine. Audit: `plugin.signature-failed` or similar per the specific tamper. Repeated hostile-CLI corpus from the plugin-bundle L2 is rerun here as integration.
14. **Inherited file descriptors.** Agent tries to pass a `mothership`-owned fd to a child process via fork-exec. Privileged-spawn helper's `close_range()` clears all fds except the three declared ones. Child cannot reach the smuggled fd. Audit: helper emits `supervisor.spawn-fd-smuggle-rejected` on detection; otherwise transparent.
15. **Environment-variable poisoning.** Agent sets `LD_PRELOAD=/tmp/malicious.so` in its env before a child spawn. Privileged-spawn helper scrubs env to the declared allowlist. Audit: `supervisor.skill-spawned` with `env_scrubbed_keys_count` indicating the poison was dropped.
16. **PATH / helper-binary substitution.** Agent places a `mothership-spawnd`-named binary in its own `PATH` hoping to be invoked as the helper. Supervisor calls the helper by absolute canonical path; PATH never consulted. Audit: N/A — attempt has no effect on supervisor behavior.
17. **Symlink trick against org home.** Agent creates a symlink inside its org home pointing to another org's home, tries to read through it. Skill process isolation sub-spec 4 handles the fs boundary; supervisor records `supervisor.org-context-fs-violation` if the symlink is observed at skill-startup scan (sub-spec 4 defines the exact check).
18. **Windows named-pipe ACL mistake test.** Integration test runs on Windows and verifies the supervisor's named pipe has an ACL allowing only the supervisor service SID + operator CLI token; every other token (agent's restricted token, LocalService, LocalSystem as a separate user) is rejected. Audit: `supervisor.peer-credential-rejected` with the requesting SID.
19. **Confused-deputy broker.** A compromised broker (simulated by a test double) tries to use its privileged channel to supervisor to request a grant-mutation on behalf of an agent. Supervisor's grant-mutation API requires a `reauth_challenge_id` issued by sub-spec 7; a broker-originated request without one is rejected. Audit: `supervisor.confused-deputy-rejected` with broker process identity.
20. **Provenance-unknown injection.** An intent reaches the broker with a provenance tag outside the closed taxonomy. Broker's compiled evaluator maps to `provenance-unknown` → baseline-deny. Audit: `broker.intent-denied(reason: provenance-unknown)`. This test guards the "unclassified provenance = deny" rule in §"Closed provenance taxonomy."

For each scenario: the attempt is blocked at the supervisor (or OS + correlated-audit) boundary; the relevant audit entry is emitted with correct actor/provenance/reason codes; no state-mutation side effect persists; any in-flight supervisor-broker snapshot exchange continues to hold the previous snapshot_version until a legitimate mutation completes.

---

## Error modes

- **Operator key lost** → supervisor cannot verify signed config; hard halt at boot. Recovery requires operator re-enroll the key via the secrets-bridge enroll flow + re-sign all config with the new key. This is a documented disaster-recovery procedure, not an operational state.
- **Grant-ledger corruption** → hard halt with `grant-ledger-corrupted`; `mothership grants recover --from <segment>` is the operator-gated recovery path.
- **Audit-unavailable + secrets bridge degraded** → supervisor serves read APIs from last-known state; rejects mutations; emits out-of-band operator notice.
- **Org-context user missing on host** (`mothership-ttl` unix user doesn't exist) → skill spawn for that context fails with `org-context-user-missing`; supervisor's org-context-registry-load fails at boot with same code; configured but-absent contexts are a fatal config error.
- **Config-file permissions wrong** (e.g., `supervisor.yaml` world-writable) → config load fails with `config-permissions-unsafe`; hard halt. Supervisor validates file mode bits before reading signed content.
- **Pinned operator key material damaged** → same as operator key lost.
- **Broker fails to ack policy snapshot within T** → supervisor treats the broker as faulted and restarts it (counts against broker-restart budget).

---

## Interface produced (consumed by downstream sub-specs)

- **Policy read API** — `GET policy/current` (returns matrix snapshot + signature fingerprint); `SUBSCRIBE policy/changed`. Consumed by Broker (T3).
- **Grant ledger read API** — `GET grants/current?scope=<…>`; `SUBSCRIBE grants/changed`. Consumed by Broker (T3).
- **Approval-callback interface** — `POST approvals/<challenge_id>/decision` (for sub-spec 7 to call back into supervisor with re-auth results).
- **Org-context resolution API** — `GET org-contexts/<id>`; consumed by Broker (T3) and Skill isolation (T4).
- **Manifest registry read API** — `GET manifests?bundle_hash=<…>`; consumed by Broker (T3) and Skill isolation (T4).
- **Supervisor audit event shapes** (declared here for the audit-events registry):
  - Lifecycle: `supervisor.started`, `supervisor.stopping`, `supervisor.stopped`, `supervisor.plugin-registry-loaded`
  - Config: `supervisor.config-loaded`, `supervisor.config-rejected`, `supervisor.config-signature-missing`, `supervisor.operator-key-rotated`, `supervisor.operator-key-mismatch`
  - Policy: `policy.matrix-updated`, `policy.reload-failed`, `policy.monotonic-lock-violation`
  - Grants: `policy.grant-created` (post-ack promotion), `policy.grant-create-tentative` (Prepare phase), `policy.grant-create-compensated` (ack-failed or orphan reconciliation), `policy.grant-revoked`, `policy.grant-use-recorded`, `policy.approval-decision`
  - Plugin install: `plugin.install-approved-decision` (at `reviewed`, immediately after re-auth), `plugin.install-activated` (at `activated`, carrying `snapshot_version` + `approval_decision_seq`)
  - Org context: `supervisor.org-context-added`, `supervisor.org-context-removed`, `supervisor.org-context-cleaned`, `supervisor.org-context-user-missing`, `supervisor.org-context-fs-violation`
  - Broker: `supervisor.broker-started`, `supervisor.broker-restarted`, `supervisor.broker-restart-budget-exhausted`, `supervisor.broker-ack-timeout`
  - Spawn helper: `supervisor.skill-spawned`, `supervisor.spawn-rejected`, `supervisor.spawn-fd-smuggle-rejected`
  - Auth / boundary: `supervisor.peer-credential-rejected`, `supervisor.confused-deputy-rejected`
  - Recovery: `supervisor.grants-recovered`, `supervisor.ledger-rolled`
  - Broker-side cross-spec: `broker.snapshot-accepted` (owned by sub-spec 2 but declared here as required for the cross-spec protocol); `broker.intent-admitted` / `broker.intent-denied` MUST carry `snapshot_version` (cross-spec normative requirement on sub-spec 2)
- **CLI subcommands** — `supervisor status`, `policy reload`, `policy verify`, `grants list`, `grants verify`, `grants recover`, `grants roll`, `org-contexts list`, `org-contexts cleanup`, `operator rotate-key`, `trust add/remove/list`, plus plugin/secret CLIs defined in T1 sub-specs.

---

## Principle compliance notes

| Principle | Compliance check | Result |
|-----------|------------------|--------|
| P2 | Can a full compromise of the agent process disable sandboxing, grant itself new capabilities, or alter the audit log? | **No.** The supervisor runs as a dedicated OS service under a dedicated non-root user; its sockets are owned by that user; its files (config, ledger, audit) are write-owned by that user only. The agent runs under a per-org-context user and has no write access to any supervisor-owned artifact. OS-level enforcement (NoNewPrivileges + seccomp / sandbox-exec / service-restricted-token) is the floor; supervisor in-process checks (peer-credential, signed config, HMAC chains) are the ceiling. |
| P6 | Can an operator-privileged action be performed without operator-presence re-auth? | **No.** The supervisor maintains a declared list of privileged ops; each triggers a sub-spec 7 challenge before the op executes. No batch approval in V1. Gateway-unreachable → op halts; no silent fallback. |
| P7 | After installing Mothership with N skills and no explicit grants, can any skill read filesystem, open shell, access secrets, or make external network requests? | **No.** The grant ledger starts empty; every grant is an explicit operator action. Trust level is badge-only (per plugin-bundle spec V0.5+). Matrix `require-grant` cells combined with an empty ledger produce deny. |
| P8 | Does every supervisor action produce an audit entry? | **Yes.** The per-action event shapes declared above cover every state mutation. Principle holds only while audit-writer is healthy; `audit-unavailable` mode freezes all mutations and surfaces via out-of-band notices. |
| P5 | Does any supervisor-handled secret exist outside the OS keystore past a single resolve call? | **Guaranteed via Secrets bridge's scoped lease.** Supervisor holds `SecretBytes` leases for `mothership/audit/hmac` and `mothership/grants/hmac` per the audit-log and secrets-bridge spec, with bounded lease windows; all leases zeroize on drop, on shutdown, and on `audit-unavailable` transition. |

### Principle tensions

- **P6 (re-auth gates) vs. UX / operational friction.** Every privileged op requires a challenge. A long IT deployment (installing 50 bundles across an org) becomes 50 challenges. V1 eats the friction; V1.5 may introduce a scoped-batch mechanism with explicit justification and expiring batch-tokens. No V1 compromise.
- **P2 (safety below API plane) vs. debuggability.** A hard halt on, e.g., a malformed policy file is harder to debug than a degraded "fall back to previous policy" mode. V1 chooses halt-first; the status pipe + diagnostic output are the debug affordance, not a degraded runtime. This matches the audit-unavailable philosophy of the audit-log spec.
- **P7 (default-deny) vs. first-run usability.** A fresh Mothership install has zero bundles, zero trust roots, zero grants — it does literally nothing useful until the operator sets all three up. This is deliberate friction. Documented in the install guide as a three-step first-run (add operator GPG key → add trust root → install first bundle → approve grants); the spec does not ship defaults.

---

## Open implementation-level questions (for writing-plans, not design)

1. **Unix socket vs named pipe on Windows.** Interprocess control-channel between supervisor and CLI/broker. Candidate: unix socket on Linux/macOS (peer-cred check via `SO_PEERCRED` / `LOCAL_PEERCRED`), Windows named pipe with `SECURITY_DESCRIPTOR` restricting to the service account's SID. Evaluate tooling (`tokio::net::UnixStream` vs. `tokio::net::windows::named_pipe`) in writing-plans.
2. **Signed-config YAML vs signed JSON.** YAML authoring is nicer for humans; signed-JSON-only simplifies the canonical-form story. Candidate: operator authors YAML, build tool produces a JCS-canonical JSON that is actually signed and loaded; the supervisor reads the signed JSON, the operator edits the YAML. Prototype whether the delta from plugin-bundle's YAML→JSON pattern is simplifying or duplicative.
3. **Policy-matrix evaluator language.** Pure-Rust compiled matrix (fastest, most constrained) vs. a tiny DSL (more flexible, more attack surface). V1 default: compiled matrix (closed enum of provenance × capability). Any DSL is V1.5+ with an explicit threat model.
4. **Broker control-channel HMAC key.** Supervisor holds an HMAC key specifically for authenticating supervisor-originated control messages to the broker; this prevents a compromised agent (which has a different process credential) from injecting into that channel. Key handle: `mothership/supervisor/broker-control-hmac`. Lease window: scoped to the duration of a broker session. Evaluate whether a session-key approach beats a long-lived HMAC here.
5. **Service-account provisioning.** Should `mothership install` create the service account + dedicated per-org-context users, or should that be a prerequisite documented separately? V1 candidate: installer creates `mothership` service user automatically (Linux: via sysusers.d; macOS: `dscl`; Windows: via service install). Per-org-context users are operator-created via `mothership org-context create` (re-auth gated), not auto-provisioned — this matches the explicit-operator-action posture.
6. **Matrix-monotonic-lock representation.** The "provenance-class deny is monotonic-locked" rule needs a concrete representation the reload validator can check. Candidate: ship a read-only compile-time baseline matrix `policy-baseline.yaml` containing all mandatory `deny` cells; the reload validator computes `proposed_matrix ⊇ baseline_deny_cells` and rejects any narrowing. Baseline itself is part of the Mothership binary (embedded), not the operator's `policy.yaml`.
7. **Windows service-restricted-token scope.** Which privileges exactly get stripped? Candidate: `SeChangeNotifyPrivilege` only. Verify with Microsoft's recommended service-restricted patterns.

These are implementation choices; resolving them does not change any contract in this sub-spec.

---

## Exit checklist (per map §"Uniform exit checklist")

- [ ] Design doc committed under `docs/superpowers/specs/`
- [ ] Implementation passes unit + integration test suite (L1 + L2 + L3)
- [ ] Integration tests with direct tier-dependencies pass (all three T1s: audit-log HMAC-chain integration; secrets-bridge `SecretBytes` lease integration; plugin-bundle install-state-machine orchestration)
- [ ] Principle compliance check re-run; any tensions logged with resolution (three tensions noted above with resolution paths)
- [ ] Spec-specific acceptance test passes (all 12 L4 compromised-agent scenarios in CI; no state-mutation side effect persists in any case)

---

## Amendments log

| Date | Change | Reason |
|------|--------|--------|
| 2026-04-19 | V0 design drafted (Gavin + Cowork). | Fourth sub-spec authored under the V1 spec map (T2, design Phase I). **Completes Phase I.** With this commit, T1+T2 design is done and Phase II build can begin (Secrets bridge → Audit log + Plugin bundle → Supervisor). |
| 2026-04-19 | V0.1 revision post-Codex review. | Major structural additions from adversarial review: (a) **`SupervisorSnapshot` + two-phase commit protocol** for every supervisor-broker state coordination (Codex's cheapest-failure finding — stale broker state was the prior architecture's biggest gap); (b) **approval audit moved to `reviewed`** via new `plugin.install-approved-decision` event so operator decision is durable independent of activation success; `plugin.install-activated` at terminal transition carries `snapshot_version` + backref to the approval seq; (c) **monotonic-lock baseline broadened + embedded** — `policy-baseline.yaml` compiled into the supervisor binary covers all untrusted-provenance × high-risk-capability cells (web-content, third-party-message, skill-output, browser-origin-callback, cli-uploaded-file, restored-backup, audit-query-output, hook-output, cross-agent-handoff); (d) **closed provenance taxonomy** with `provenance-unknown` → baseline-deny rule; (e) **explicit grant-ledger recovery algorithm** replacing the prior error-mode bullet — quarantine corrupt segment, emit `grants.chain-restart`, reconcile orphan tentatives, operator choice of replay-from-audit vs. clean-slate; (f) **tentative-vs-durable grant entries** supporting the two-phase commit; (g) **privileged-spawn helper** specified as a distinct TCB component with declared command surface, authenticated input, allowed-UID allowlist, env scrub, fd closure, no shell, audited; (h) **Linux V1 documented as interactive-desktop-only** (headless-Linux unattended-boot is V1.5+ pending a secrets-bridge backend that doesn't require session D-Bus) — resolves the architectural contradiction Codex flagged; (i) L4 expanded from 12 to **20 scenarios** including OpenClaw-class indirect attacks (browser-origin against gateway, compromised CLI upload, fd inheritance, env poisoning, PATH substitution, symlink tricks, Windows named-pipe ACL, confused-deputy broker, provenance-unknown injection); (j) L3 expanded with **broker-ack failure matrix** (ack timeout / reject / mid-flight crash), grant-race + replay tests, separate audit-key vs grant-key rotation tests, shutdown-in-flight tests; (k) **orphan-process reaping at boot** before agents-allowed gate; (l) grant-read API now tags all reads with `snapshot_version`; (m) broker intent-admitted/denied events MUST carry `snapshot_version` (cross-spec requirement on sub-spec 2). No contract changes to downstream sub-specs beyond the new cross-spec requirements that the broker carry `snapshot_version` in its events and that the broker's compiled matrix be `SupervisorSnapshot`-tied. |

---

## Codex Review

**Triggers matched:** auth/authorization (policy engine, grant ledger, operator-presence re-auth policy), infrastructure (service lifecycle, systemd/launchd/Windows service units, privileged-spawn helper as TCB), secrets (audit HMAC key + grant HMAC key lease scope), irreversible one-shots (plugin install commit, grant-ledger append, operator-key rotation).
**Codex effort:** default
**Reviewed:** 2026-04-19

### Codex findings (summary)

**Rollback.** Plugin install at `committed→activated` described as "one logical transaction" without a transaction mechanism. Broker-signal failure after grants are durable leaves supervisor/broker divergent. Grant-ledger `mothership grants recover --from <segment>` named but not algorithmic. Policy reload between supervisor and broker had no ack contract. No tests for broker-ack failures or partial-commit crashes.

**Blast radius.** Org-context isolation on Windows (ACLs, named-pipe ACLs, service SIDs, token privileges) underspecified; on Unix the `setuid(org_ctx_uid)` helper is undesigned TCB. L4 "no audit needed because OS blocked" contradicts the map's acceptance test (every blocked attempt must log).

**Missing tests.** Stale broker state after supervisor mutation (the most likely production failure). Grant-ledger races (concurrent CRUD, subscription ordering, snapshot consistency). Audit-key vs grant-key rotation conflated. Operator-presence replay/concurrency (same challenge for two ops, approval after timeout, approval from wrong origin). Shutdown in-flight (install, grant append, audit entry, outstanding re-auth challenge). OpenClaw-class scenarios: malicious browser/origin against gateway, compromised CLI, inherited fds, env-var poisoning, PATH substitution, symlink/mount namespace tricks, named-pipe ACL on Windows, confused-deputy broker.

**Wrong assumptions.** (a) Cold-boot ordering "only after broker signals ready, agents may spawn" is assertion without mechanism. (b) Linux keystore: supervisor is "native OS service, unattended boot" but secrets bridge requires session D-Bus on Linux — direct contradiction. (c) Provenance taxonomy not proven exhaustive: browser-origin callback, CLI-uploaded file, restored backup, plugin descriptor content, audit-query output, hook output, cross-agent handoff missing. (d) Monotonic-lock only called out `web-content` deny cells; third-party-message, skill-output, overrides, new classes remain downgrade paths. (e) Privileged spawn helper is TCB but has no surface design. (f) Approval audit at `activated` is too late — if activation fails, approval decision may never become audit-visible, a P8 hole.

**Cheapest failure.** Stale/inconsistent broker state after supervisor mutation. Broker restart, missed subscription event, activation race, or crash between `committed` and `activated` can leave broker admitting intents against old grants/policy while supervisor believes the new state is active. Defeats P2 and P7 without agent compromise.

**Alternatives.** (1) Versioned `SupervisorSnapshot` with broker-ack-required two-phase protocol; broker tags admission events with snapshot_version. (2) Move approval audit earlier (after re-auth, before staging). (3) Treat grant recovery as a real design section: truncate/fork/replay/quarantine specified; operator re-auth; chain-restart marker; test every corruption case. (4) Broaden monotonic policy baseline and embed in binary; mandatory deny cells cover all untrusted-provenance × high-risk-capability combinations; new classes fail closed. (5) Don't ship Linux V1 with session-D-Bus keystore as supervisor bootstrap dependency unless "unattended boot" is dropped from requirements.

### Claude's resolution

- **Rollback finding:** **Accepted.** New §"State versioning and broker coordination" adds `SupervisorSnapshot` + two-phase commit (prepare → broker-ack → activate). Plugin `committed` state now uses the protocol: grants are tentative until ack; compensation entries on timeout/reject. Grant-ledger recovery is now a normative algorithm (quarantine + chain-restart + orphan reconciliation + replay-from-audit vs clean-slate operator choice).
- **Blast radius finding:** **Accepted.** Windows org-context isolation now specifies per-context security principals + ACLs + named-pipe ACLs + service SIDs; L4 Windows-ACL scenario added. Unix privileged-spawn helper now has a full design (command surface, authenticated input, UID allowlist, env scrub, fd closure, no shell, audited). L4 every-blocked-attempt now MUST log — correlated audit entries required even for OS-level denials.
- **Missing tests finding:** **Accepted.** L3 expanded with broker-ack failure matrix (timeout, reject, mid-flight crash, stale-broker differential), grant-race tests (concurrent CRUD, subscribe ordering), separate audit-key vs grant-key rotation, shutdown-in-flight matrix. L4 expanded from 12 to 20 scenarios including all OpenClaw-class indirect attacks Codex enumerated.
- **Wrong assumptions finding:** **Accepted.** (a) Agent-spawn mechanically gated by privileged-spawn helper requiring supervisor-issued nonce; nonces only minted after broker `ready`. Orphan-process reaping at boot. (b) Linux V1 documented as interactive-desktop-only; headless-Linux moved to V1.5+ pending secrets-bridge TPM/keyctl backend; contradiction resolved. (c) Provenance taxonomy closed + `provenance-unknown` → baseline-deny; new classes added to match Codex list. (d) Monotonic-lock baseline now embedded + covers all untrusted/derived provenance × high-risk capabilities; narrowing requires supervisor major-version bump. (e) Privileged-spawn helper fully specified. (f) `plugin.install-approved-decision` event now emitted at `reviewed` (immediately after re-auth); `plugin.install-activated` separately records the terminal transition with snapshot_version backref. Approval is durable even if activation later fails.
- **Cheapest failure finding:** **Accepted.** The `SupervisorSnapshot` + two-phase commit architecture directly addresses this. Broker intent events carry snapshot_version; supervisor cannot mutate without broker-ack; broker cannot admit intents from a stale snapshot. Stale-broker differential test added to L3 as a specific CI gate.
- **Alternatives finding:** **All five accepted.** The spec now includes all proposed alternatives: snapshot versioning + ack protocol, approval audit earlier, explicit grant recovery, broadened embedded baseline, Linux V1 interactive-only.

### Summary

Codex's review was substantively correct on every structural point; the V0 draft would have shipped with a supervisor-broker coordination architecture that would fail on the first broker restart mid-install. The revision adds two major new primitives — `SupervisorSnapshot` with two-phase commit, and the privileged-spawn helper as a designed TCB component — plus a broader monotonic-lock baseline that closes the provenance-class-downgrade holes. Approval-audit placement, grant-ledger recovery algorithm, and the interactive-Linux V1 scope decision resolve three distinct architectural ambiguities. The resulting spec is materially more coherent and harder to exploit; it now specifies, rather than gestures at, every supervisor-broker coordination point. Cross-spec impact on sub-spec 2 (Broker) is minor but normative: broker events MUST carry `snapshot_version` and broker's compiled matrix MUST be snapshot-version-tied.
