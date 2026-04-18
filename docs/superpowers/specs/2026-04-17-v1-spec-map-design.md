# Mothership V1 Spec Map

**Status:** V0 — committed 2026-04-17. First engineering-forcing artifact after `design-principles.md` and `use-cases.md`.
**Authored:** 2026-04-17 (Gavin + Cowork)
**Companion docs:**
- [`design-principles.md`](../../../design-principles.md) — the *what it must be* (8 non-negotiable principles)
- [`use-cases.md`](../../../use-cases.md) — the *what it must do for whom* (V1 beacon portfolio: UC1, UC2, UC3)
- [`docs/brainstorm/early-spec.md`](../../brainstorm/early-spec.md) — early architecture brainstorm (pre-principles; superseded by principles where they disagree)
- [`research/hermes-agent.md`](../../../research/hermes-agent.md) — reference architecture at the agent layer
- [`research/openclaw.md`](../../../research/openclaw.md) — counter-design target; failure-case evidence
- [`research/claude-code-cowork-research.md`](../../../research/claude-code-cowork-research.md) — extension-model reference

---

## Purpose

This map is the **table of contents for Mothership V1 engineering work**. It decomposes the substrate into seven sub-specs, sequences design and build across four dependency tiers, and fixes the rules by which each sub-spec is authored, reviewed, and accepted.

It is not itself a design for any component. Each sub-spec listed here will be authored as its own document under `docs/superpowers/specs/` and carry its own detailed design, data flows, error handling, and test plan.

## How to use this doc

- **Every Mothership V1 sub-spec MUST cite:**
  1. This map by path
  2. ≥1 UC from `use-cases.md`
  3. The principles it load-bears from `design-principles.md`
  4. Its tier (T1–T4) and its direct upstream/downstream sub-specs
- **Principle wins.** If a sub-spec design conflicts with a principle, the principle wins. Amending a principle requires a dated entry in `design-principles.md` per its own change process.
- **Map amendments.** Adding, removing, or resequencing sub-specs; changing tiers; or changing a sub-spec's acceptance test requires a dated entry in this map's Amendments log with justification.
- **Open questions stay here.** Cross-cutting questions that don't resolve inside a single sub-spec live in the Open Questions section. Resolution closes the entry with a pointer to the resolving sub-spec.
- **The map does not pick tools, languages, or directory layouts.** Those decisions happen inside each sub-spec, subject to `design-principles.md` and `CLAUDE.md`.

---

## Terminology

Three different things are called "skill" in the surrounding literature. This map — and every Mothership sub-spec — uses only the first:

| Term | Meaning | Source |
|------|---------|--------|
| **skill** (Mothership) | An **MCP server** packaged as a signed bundle with a capability manifest. | `design-principles.md` P3 |
| **skill** (Claude Code / Cowork) | An instruction folder (SKILL.md + optional scripts) that Claude reads as a knowledge document. | Not a Mothership primitive; may be hosted *inside* an agent's own plugin bundle. |
| **skill** (Hermes) | A self-improving procedural-memory document. | Agent-layer concept; same treatment as Claude Code "skill." |

Other canonical terms used throughout this map and downstream sub-specs:

- **agent** — a process running under the broker (reader, planner, executor, action-agent, or monitor per `early-spec.md`). Agents have their own signed manifests distinct from skill manifests.
- **plugin** — a signed bundle containing zero or more skill manifests, zero or more agent manifests, and zero or more hook definitions. The plugin is the **install unit**.
- **hook** — a policy extension point at a named broker lifecycle event. Runs sandboxed, operator-authored.
- **intent** — a typed message flowing through the broker. Every side effect in the substrate is an intent.

---

## The seven sub-specs

Each entry is the map-level sketch only. Full design lives in the sub-spec's own document.

### Sub-spec 1 — Supervisor
- **Tier:** T2 &nbsp;|&nbsp; **Principles:** P2, P6, P7 &nbsp;|&nbsp; **UCs:** UC1, UC2, UC3
- **Owns:** service lifecycle (systemd / launchd / Windows service); signed, integrity-checked config; capability grant ledger (per-skill, per-tenant, revocable); policy engine (provenance × capability matrix, operator-editable); plugin install approval flow; agent manifest loading; cold-boot unlock flow; broker-before-agents startup ordering; scrub-on-shutdown; org-context registry + unix-user mapping; orchestration of operator-presence re-auth (delegates primitives to sub-spec 7).
- **Consumes:** secrets bridge (own service key); audit log (policy changes, grants, approvals, installs); plugin bundle schema.
- **Produces:** policy read API; grant ledger read API; approval-callback interface (for broker); org-context resolution API.
- **Acceptance test:** *A compromised agent process cannot rewrite policy, grant itself new capabilities, or alter the audit log; every such attempt is blocked at the supervisor boundary and produces an audit entry.*

### Sub-spec 2 — Broker + provenance
- **Tier:** T3 &nbsp;|&nbsp; **Principles:** P1, P4 &nbsp;|&nbsp; **UCs:** UC1, UC2, UC3
- **Owns:** single typed-RPC pipe for all agent↔tool / agent↔agent / agent↔memory / agent↔fs traffic; provenance taxonomy (`operator-typed`, `trusted-local`, `web-content`, `third-party`, `skill-output`, `supervisor-authored`); deterministic input pre-processor (delimiters, spotlighting, canaries, pattern scrubbing); provenance propagation through tool calls and agent handoffs; capability-matrix evaluator; named lifecycle events with sandboxed operator-authored policy hooks (PreToolCall, PostToolCall, PermissionRequest, SessionStart, SessionEnd, Stop, SubagentStop, etc.); built-in deterministic intent handlers (memory, audit-query); memory scoping semantics (per-project, per-agent, per-org-context); sensitive-intent fast-model second-pass; per-agent rate limits; behavioral baseline stats (hook for Phase C).
- **Consumes:** supervisor (policy + grants + approval callbacks + plugin manifest registry); audit log (decision per invocation); secrets bridge (handle resolution on approved egress).
- **Produces:** tool-call invocation API (for skill process isolation); agent-handoff API (for agent runtimes, external to substrate); memory-read/write intent API; audit-query API.
- **Acceptance test:** *An input tagged `web-content` cannot cause a shell-exec, external-send, or secret-read tool call regardless of what the model emits, without at least one `operator-typed` intent in the causal chain; every denial produces an audit entry.*

### Sub-spec 3 — Plugin bundle + signing
- **Tier:** T1 &nbsp;|&nbsp; **Principles:** P3 (install), P7 (install) &nbsp;|&nbsp; **UCs:** UC3 (primary), UC1
- **Owns:** plugin bundle format (signed archive containing skill manifests, agent manifests, hook definitions, MCP server configs); skill manifest schema (declared capabilities, publisher identity, version, dependencies); agent manifest schema (type, permissions, triggers, model tier); hook definition schema; signing spec (what's signed, with what key, how verified); publisher identity model (resolves `design-principles.md` open-question 2); install-time verification procedure.
- **Consumes:** nothing at design time.
- **Produces:** signed bundle format (for supervisor at install); verified-manifest objects (for broker + skill process isolation at runtime).
- **Acceptance test:** *An unsigned or tamper-modified plugin bundle cannot install to a default-configured Mothership; an install attempt is rejected by the supervisor and produces an audit entry.*

### Sub-spec 4 — Skill process isolation
- **Tier:** T4 &nbsp;|&nbsp; **Principles:** P3 (runtime), P7 (runtime) &nbsp;|&nbsp; **UCs:** UC1 (primary — multi-client boundary), UC3
- **Owns:** per-skill OS process model with bounded capabilities; OS-level manifest enforcement (syscall / network / fs constraints derived from manifest); skill lifecycle (start / stop / crash / reload); cross-skill isolation (no shared memory; no shared fs unless manifested); reconciliation of "tools as action agents" (from `early-spec.md`) with MCP skill servers — resolves Open Question 1.
- **Consumes:** broker (all I/O); supervisor (grants per invocation, manifest registry); plugin bundle (capability declarations at runtime).
- **Produces:** MCP-server hosting contract (for skill authors — external); action-agent hosting contract (if resolved as distinct from MCP skill).
- **Acceptance test:** *A skill declaring only `network:read` cannot open a shell, write outside its declared fs scope, or read a secret — attempts are blocked at the OS boundary (not just the broker) and logged.*

### Sub-spec 5 — Audit log
- **Tier:** T1 &nbsp;|&nbsp; **Principles:** P8 &nbsp;|&nbsp; **UCs:** UC1, UC2, UC3
- **Owns:** append-only local store (file format + rotation); per-entry HMAC chaining; `mothership audit verify` CLI; pluggable external-mirror adapters (local file / S3 object-lock / syslog / SIEM — defaults deferred per `design-principles.md` open-question 4).
- **Consumes:** secrets bridge (HMAC key).
- **Produces:** write API (for supervisor, broker, skill process isolation, origin gateway); verify CLI (for operator, out-of-band).
- **Acceptance test:** *Tampering with any entry in the local log causes `audit verify` to fail at that entry; a full-compromise rewrite of the local log is detectable by diffing against the out-of-band external mirror.*

### Sub-spec 6 — Secrets bridge
- **Tier:** T1 &nbsp;|&nbsp; **Principles:** P5 &nbsp;|&nbsp; **UCs:** UC2 (primary), UC1, UC3
- **Owns:** OS keystore abstraction (DPAPI / Keychain / libsecret); handle-based retrieval API (opaque handle, resolved at use); in-memory-lifetime guarantees (scrub after single call); export audit event.
- **Consumes:** OS keystore APIs only.
- **Produces:** handle API (for supervisor, broker, audit log).
- **Acceptance test:** *No plaintext secret exists on disk, in logs, in crash dumps, or in process memory past the scope of a single resolve call; operator-initiated export is the only legitimate secret egress and produces an audit entry.*

### Sub-spec 7 — Origin & operator-presence gateway
- **Tier:** T3 &nbsp;|&nbsp; **Principles:** P6 &nbsp;|&nbsp; **UCs:** UC2, UC3
- **Owns:** origin allowlist (operator-managed); per-session short-lived tokens; re-auth primitives (Touch ID, Windows Hello, YubiKey, PAM); privileged-action gating (capability changes, plugin installs, config writes, approval toggles); approval-fatigue UX (batching, risk-tiering, default-deny timeouts, justification-per-prompt, review-after-the-fact sampling).
- **Consumes:** supervisor (delegates re-auth surface); audit log (every auth event).
- **Produces:** re-auth challenge API (for supervisor); session-token validation (for any external-facing surface).
- **Acceptance test:** *A malicious page in a normal browser tab — or any unapproved origin — cannot cause Mothership to execute any privileged action; every privileged action requires a fresh operator-presence re-auth via the gateway.*

### Uniform exit checklist

Applies to every sub-spec in addition to its spec-specific acceptance test:

- [ ] Design doc committed under `docs/superpowers/specs/`
- [ ] Implementation passes its unit + integration test suite
- [ ] Integration tests with direct tier-dependencies pass
- [ ] Principle compliance check re-run; any tensions logged with resolution
- [ ] Spec-specific acceptance test passes

---

## Dependency DAG

```
T1 (standalone, parallel-designable):
  ├─ Secrets bridge           (depends only on OS keystore APIs)
  ├─ Audit log                (depends on secrets bridge for HMAC key)
  └─ Plugin bundle + signing  (schema + signing spec; mostly standalone)

T2:
  └─ Supervisor               (integrates T1: owns policy + grant
                               ledger + operator-presence orchestration;
                               writes to audit log; uses secrets bridge;
                               loads plugin bundles)

T3 (parallel-designable; both need Supervisor):
  ├─ Origin & operator-presence gateway  (extends supervisor auth surface)
  └─ Broker + provenance                 (consumes supervisor policy + matrix)

T4:
  └─ Skill process isolation  (consumes broker + supervisor + plugin bundle)
```

---

## Build phasing (hybrid)

| Phase | Activity | Sub-specs in scope |
|-------|----------|--------------------|
| **I. Design** | Design T1 + T2 together before any implementation. All four specs committed before Phase II begins. | Secrets bridge, Audit log, Plugin bundle + signing, Supervisor |
| **II. Build** | Implement T1 + T2 in dependency order. | Secrets bridge → Audit log + Plugin bundle → Supervisor |
| **III. Design** | Design T3 against the running T2 keystone (reality-tested interfaces). | Broker + provenance, Origin & operator-presence gateway |
| **IV. Build** | Implement T3. | (T3 sub-specs) |
| **V. Design + build** | T4 — the integration point where broker + supervisor + plugin bundle meet. | Skill process isolation |

Each phase exits when every sub-spec in that phase passes its uniform exit checklist + spec-specific acceptance test.

---

## Open questions

Carried forward from `design-principles.md`, `early-spec.md`, `session-notes.md`, and the three research docs. Each resolves inside a specific sub-spec or by amendment to this map.

| # | Question | Resolves in |
|---|----------|-------------|
| 1 | **Tools as action agents vs. MCP skills.** `early-spec.md` wraps every tool as a broker-addressable action agent; `design-principles.md` P3 says MCP is the skill interface. Are V1 "tools" (a) action agents, (b) MCP skills, or (c) both with a clear division? | Sub-spec 4 (Skill process isolation) |
| 2 | **Approval fatigue UX.** Gated-intent rate must produce meaningful (not rubber-stamped) operator decisions. Needs batching, risk-tiering, default-deny timeouts, justification-per-prompt, review-after-the-fact sampling. | Sub-spec 7 (primary), sub-spec 1 (secondary) |
| 3 | **Model supply chain.** Ollama pulls unsigned model weights. A compromised weight is a baked-in prompt injection. Pin model hashes? Mirror weights locally? | Out of V1 substrate scope; tracked here. Likely resolves in an agent-layer spec or Phase D. |
| 4 | **Backup story.** Single-box data + secrets = single point of failure. Encrypted backup destination, restoration drill, secrets-blob backup strategy, test-restore cadence. | Sub-spec 5 partially addresses via audit external mirror; full backup strategy tracked here for V1.5. |
| 5 | **Claude Code plugin compatibility (stretch).** If Mothership's plugin bundle is a superset of Claude Code's `.claude-plugin/plugin.json` shape, existing CC plugins can be wrapped to run substrate-hosted with real enforcement. Stretch goal; not V1 requirement. | Sub-spec 3 — may declare the bundle shape as CC-compatible. |

---

## Amendments log

| Date | Change | Reason |
|------|--------|--------|
| 2026-04-17 | V0 map created (Gavin + Cowork). 7 sub-specs across 4 tiers. Hybrid build phasing. | First engineering-forcing artifact after principles + use cases lock. |
