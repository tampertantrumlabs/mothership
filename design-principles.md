# Mothership Design Principles

**Status:** V0 — anchor doc. These principles predate the spec and outlive iteration on it.
**Source evidence:** `../../../Research/openclaw.md`
**Created:** 2026-04-16
**Last revision:** 2026-04-16

---

## Purpose

This is not a spec. It is the non-negotiable frame around which every Mothership spec, feature, and code change must sit. When a design decision is in question, it is checked against these principles. A proposed feature that violates one of them is either rejected or the principle is explicitly amended with written justification tied to new research, a new threat model, or an incident.

This doc exists because OpenClaw — the category-defining local agent runtime — accumulated 138 CVEs (7 Critical, 49 High), a ~37% malicious-skill rate in its marketplace, 40,000+ internet-exposed instances, and a 1.5M-token backend breach in under six months. Every one of those outcomes traces to a design decision that was made (or not made) before the runtime shipped. Mothership starts with the commitments.

## How this doc is used

- **Design review gate.** Every new spec in `Projects/Tampertantrum/mothership/` must cite which principles it honors and explicitly flag any tension.
- **Claude Code context.** Any session working on Mothership reads this doc first. It is referenced from `mothership/CLAUDE.md` once that file exists.
- **Change process.** Principles are deliberately stable. Amending one requires a dated entry in the "Principle changes" section at the bottom of this doc, with the research, incident, or threat-model change that motivated the amendment. Implementation tradeoffs do not amend principles; they either honor them or fail the review.

---

## The principles

### P1 — Capability separation (no lethal trifecta in one agent)

**Principle.** An agent process that reads untrusted content is not the same process that can write to shell, filesystem, or send external messages. Capability boundaries are enforced at the OS-process level, not by config.

**OpenClaw's failure.** A single agent held private-data access + untrusted web/email ingest + shell execution + external messaging simultaneously. Prompt injection via web content → shell execution was a structural feature of the design, not an implementation bug. CVE-2026-22708 (indirect prompt injection via browsing) and CVE-2026-30741 (prompt-injection-driven code execution) are both instances of this one architectural choice.

**Mothership commitment.** Agents are split by capability class (reader, planner, executor, broker). Handoffs between them cross a typed RPC boundary carrying the minimum necessary payload. A reader agent cannot directly invoke an executor; it emits intents that a separate supervisor routes subject to policy.

**Compliance check.** For any new feature, draw the capability graph. If a single process holds two of `{untrusted-input-ingest, secrets-read, shell-exec, external-send}`, the design fails.

---

### P2 — Safety enforcement below the API plane

**Principle.** Sandboxing, approval gates, and capability grants are enforced by a supervisor process that the agent cannot reach through any API it exposes. The supervisor has its own authentication and its own audit log.

**OpenClaw's failure.** In the CVE-2026-25253 exploit chain, the attacker disabled the sandbox by setting `exec.approvals.set = off` and escaped Docker by setting `tools.exec.host = gateway` — both through the same WebSocket the agent used for normal operation. The sandbox was *configuration data that the agent could rewrite*. As Mav Levin (DepthFirst) put it, these defenses were designed to contain a malicious LLM, not a malicious operator — and thus failed against either.

**Mothership commitment.** The supervisor runs as a separate OS service (systemd unit / launchd agent / Windows service) with its own config file, signed and integrity-checked on load. Configuration changes require an operator credential that no agent process ever possesses. The audit log is written through a handle the agent process cannot open for write.

**Compliance check.** Would a full compromise of the agent process allow an attacker to disable sandboxing, grant new capabilities, or alter the audit log? If yes, the design fails.

---

### P3 — Signed, capability-declared skills

**Principle.** Every extension ships with a publisher identity (signature), a capability manifest (explicit declaration of everything it will access), and a reviewed provenance. Unsigned extensions either do not run or run in a hard sandbox with zero host capabilities.

**OpenClaw's failure.** ClawHub let anyone with a week-old GitHub account publish skills. Snyk's ToxicSkills audit found ~37% of 3,984 scanned skills were malicious. Cisco documented credential harvesting and silent data exfiltration in third-party skills. The eventual remediation — VirusTotal scanning *after the fact* — treats supply chain as a scanning problem when it is actually a publisher-identity problem.

**Mothership commitment.** Use MCP (Model Context Protocol) as the skill interface — skills are MCP servers. Each MCP server is packaged with a signed manifest declaring every capability it requires. The runtime enforces the manifest: an MCP server declaring `network:read` cannot open a shell. Publisher identity is required; publisher trust level maps to runtime privileges. This integrates directly with TT Labs' `mcp-security-checklist` — Mothership is the runtime that enforces the checklist.

**Compliance check.** Can an attacker with a fresh, unverified identity publish a skill that reads user secrets on a default-configured Mothership? If yes, the design fails. Can a signed skill exceed its declared manifest at runtime? If yes, the design fails.

---

### P4 — Provenance-tagged inputs, capability-gated outputs

**Principle.** Every input to the agent carries a provenance tag (`operator-typed`, `trusted-local-file`, `web-content`, `third-party-message`, `skill-output`, etc.). Capability triggers are gated by provenance class. Web content cannot cause shell execution regardless of what the model "decides."

**OpenClaw's failure.** Raw web content, raw email content, and raw skill outputs all landed in the same model context with no provenance. Any of them could carry instructions the model would then execute. This is the semantic-firewall gap that made indirect prompt injection a class vulnerability rather than an implementation bug (see CVE-2026-22708).

**Mothership commitment.** All inputs carry provenance metadata end-to-end, down to the tool-call layer. The supervisor enforces a declarative capability matrix specifying which provenance classes may cause which tool classes to fire. Shell execution requires at least one `operator-typed` intent in the causal chain. The matrix is operator-editable config, inspectable in the audit log, never hardcoded.

**Compliance check.** For every tool-invocation path, can `web-content` or `third-party-message` provenance cause this path to fire without an `operator-typed` intent? If yes, the design fails.

---

### P5 — OS-native secret storage

**Principle.** Secrets (API keys, OAuth tokens, service credentials) live in the OS-provided keystore — DPAPI on Windows, Keychain on macOS, Secret Service / libsecret on Linux. No plaintext dotfiles. No "encrypted-with-a-key-stored-next-to-the-data."

**OpenClaw's failure.** Tokens stored in `~/.clawdbot` / `~/.openclaw` in plaintext. Multiple independent security writeups flagged this as a probable future infostealer target, the same pattern that makes `~/.npmrc` and `~/.gitconfig` high-value loot. The Moltbook backend breach compounded the problem by leaking 1.5M tokens that were portable because they weren't bound to the originating machine.

**Mothership commitment.** Every secret is referenced by handle. Retrieval goes through an OS-keystore call scoped to the Mothership service identity. Secrets are never written outside the keystore — not in config, not in logs, not in crash dumps. On Windows, DPAPI binds to user profile (machine-bound optional). Export or backup requires explicit operator action and produces an audit event.

**Compliance check.** Does any secret exist outside the OS keystore at any point in its lifecycle, including logs, crash dumps, config exports, or memory-resident past the lifetime of a single API call? If yes, the design fails.

---

### P6 — Strict origin policy, short-lived tokens, operator-presence gates

**Principle.** Localhost binding is necessary but not sufficient. The runtime enforces origin allowlisting on every connection, short-lived per-session tokens, and operator-presence re-authentication for privileged actions. Drive-by browser attacks against a localhost-bound daemon fail by design.

**OpenClaw's failure.** CVE-2026-25253 (CVSS 8.8) worked *because* the daemon bound to localhost — the victim's own browser made the outbound WebSocket connection, and the server had no origin validation. Same-origin assumptions don't apply to WebSocket the same way they apply to XHR. Layered on this: 40,214 internet-exposed instances per SecurityScorecard at disclosure, but even the strictly-local ones were exploitable.

**Mothership commitment.** Origin allowlist is explicit: only operator-approved origins may connect. Per-session tokens expire in minutes, not days. Privileged actions (capability changes, skill installs, config writes, approval-gate toggles) require operator-presence re-auth (Touch ID / Windows Hello / YubiKey / equivalent) via the supervisor, never via the agent.

**Compliance check.** Can a malicious webpage that the operator visits in a normal browser tab cause Mothership to execute any privileged action? If yes, the design fails.

---

### P7 — Capability default-deny, explicit grants per skill

**Principle.** A freshly installed skill has zero capabilities. Every capability in its manifest requires explicit operator grant at install time, per-skill, revocable at any time. The operator UI displays current grants as a single auditable list.

**OpenClaw's failure.** OpenClaw's model is "run the agent, grant it shell and filesystem and everything by default, hope individual skills don't abuse it." This left 40,000+ instances one prompt injection away from full compromise, and made the malicious-skill problem catastrophic rather than contained.

**Mothership commitment.** Default deny. Every capability grant is a deliberate operator action. Grants are scoped (per-skill, optionally time-limited), auditable, and revocable. The supervisor maintains the grant ledger; the agent cannot self-escalate or request grants it wasn't given.

**Compliance check.** After installing Mothership and N skills with no explicit grants, can any skill read the user's filesystem, open a shell, access secrets, or make an external network request? If yes, the design fails.

---

### P8 — Tamper-evident audit log

**Principle.** Every capability invocation, every approval decision, every config change, every skill install, every supervisor action produces a log entry in an append-only, integrity-protected log. The operator can verify log integrity out-of-band. A compromised agent or supervisor cannot rewrite history undetectably.

**OpenClaw's failure.** No equivalent exists. Forensics on OpenClaw incidents have relied on external evidence — network captures, endpoint artifacts, third-party logs — because the runtime itself does not produce a trustworthy audit trail. This also made the scale of the marketplace malicious-skill problem much harder to bound than it should have been.

**Mothership commitment.** The supervisor writes the audit log to an append-only store with per-entry HMAC chaining (each entry includes a MAC of the previous entry plus the current entry payload). A CLI command (`mothership audit verify`) recomputes the chain; any rewrite is detectable. Logs are optionally mirrored to an operator-chosen external sink (local file, S3 with object-lock, syslog, SIEM).

**Compliance check.** Can a full compromise of both the agent and the supervisor process hide its tracks by rewriting the log without being detectable by an operator running `audit verify` against an out-of-band copy? If yes, the design fails.

---

## Non-goals

Mothership explicitly does not pursue:

- **Matching OpenClaw's 25+ messaging channel integrations.** V1 may ship with zero messaging channels. Messaging is a capability, not a requirement.
- **A skill marketplace competing with ClawHub on volume.** Identity and review over count. Shipping with ~20 signed, reviewed skills and a clear authoring path for operators is sufficient for V1.
- **A bespoke skill DSL.** Use MCP. Skills are MCP servers. Inherit the ecosystem rather than fork it.
- **Ease-of-setup parity with Claude Code Channels for casual users.** Mothership targets security-conscious operators; setup is opinionated and defensible, not one-click.
- **A built-in LLM abstraction layer.** V1 ships with Claude (direct API + Bedrock). Other models are added only on demonstrated demand with a documented threat-model delta.
- **Proactive / heartbeat behavior in V1.** OpenClaw's proactive agents were one of the more exploitable surfaces. Mothership V1 is operator-triggered; autonomous heartbeats arrive only after the principle framework has been battle-tested.

## Positioning (context, not a principle)

- **Against OpenClaw:** Mothership is the version that did not skip the architecture phase.
- **Against Claude Code Channels:** Mothership is locally hostable, open-source, filesystem-native, and operator-owned. CCC is a reasonable managed option delivered inside Anthropic's safety perimeter; Mothership is the option for operators who want the runtime to be *theirs*, auditable, and forkable.
- **Within TT Labs:** `mcp-security-checklist` is the public statement of expertise. Mothership is the product that implements the checklist. The two reinforce each other.

## Open questions (to resolve before spec)

These are the known places where principles interact with implementation in ways that need research or a decision:

1. **Supervisor ↔ agent RPC:** what transport? Unix domain socket with peer-credential check is the default candidate; evaluate vs named pipes on Windows.
2. **Signing authority for skills:** operator-local trust (sigstore-style, with an operator-chosen trust root) vs a TT Labs-operated CA vs hybrid. The first is more principled; the third may be easier to operationalize for non-expert operators.
3. **Operator-presence primitives per OS:** which libraries / APIs are the right abstractions for Touch ID / Windows Hello / YubiKey / PAM-style challenges? This affects P6.
4. **Audit log external-mirror defaults:** do we ship with a recommended SIEM target (ties to TT Labs' `siem` project) or leave it fully operator-configured?
5. **Capability manifest schema:** does it follow OCI/SBOM conventions, or do we author a Mothership-specific schema that can later be mapped? Principle #3 doesn't mandate one; the downstream tooling decision does.

These open questions do not change the principles; they scope what the first spec must answer.

---

## Principle changes

*(none — this is V0)*
