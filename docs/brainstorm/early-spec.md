# Mothership — Early Spec

> The Mothership is TTL's secure, locally-hosted agent runtime. The Astromechs deploy from here, report findings back here, and the Mothership coordinates everything. Takes the useful parts of OpenClaw (always-on, skills, memory, messaging control) and rebuilds them with a security-first architecture.
>
> **Status:** Idea stage. Not building yet. Spec for reference when the time comes.
> **Hardware:** GMKtek box, 96GB unified memory, Ryzen AI (NPU), 2TB SSD.
> **Timeline:** Tier 4/5. Build the standalone pieces first (Ollama + Astromechs + scanning pipeline), unify into Mothership once the workflows are proven.
> **Repo (planned):** `tampertantrum/mothership` (private initially, evaluate open-sourcing the generic runtime layer later)

---

## What Mothership is

A local agent orchestration system that:

- Runs entirely on TTL-owned hardware. No client code, findings, or credentials touch cloud LLM APIs.
- Hosts multiple specialized agents that can communicate securely through a message broker.
- Provides a skills/plugin system with explicit permission scoping.
- Persists memory and context across sessions.
- Can be controlled via Slack, Discord, or a local web UI.
- Supports multiple organizational contexts (TTL engagements, APIsec work) with strict data isolation between them.

## What Mothership is NOT

- Not a product (for now). Internal tooling.
- Not a general-purpose assistant. Every agent has a defined scope.
- Not a cloud service. Runs on one box on Gavin's network.
- Not OpenClaw with a different name. The architecture is fundamentally different in how it handles trust, permissions, and inter-agent communication.

---

## Core architecture

### Inference layer

- **Ollama** as the local model server. Supports multiple models concurrently, has a clean HTTP API, handles model management.
- **Primary model:** Best available open 70B+ model (Llama 3, Qwen 2.5, DeepSeek, or whatever leads at build time). Used for code review, finding generation, complex reasoning.
- **Fast model:** 7-14B model for triage, summarization, content drafting, message routing. Low latency, runs alongside the primary model given 96GB of memory.
- **Embedding model:** nomic-embed-text or similar, for RAG indexing. Lightweight, runs through Ollama.
- **Ryzen AI NPU:** Explore using for the fast model or embedding model to free GPU/CPU resources for the primary model. ROCm support through Ollama.

### Agent framework

Each agent is defined by a markdown file (same format as Astromechs) plus a manifest that declares:

```yaml
id: frontend-security-review
name: Frontend Security Review
type: review  # review | monitor | generator | triage
model: primary  # which model tier to use
permissions:
  filesystem:
    read: ["${ENGAGEMENT_DIR}/**"]
    write: ["${ENGAGEMENT_DIR}/findings/**"]
  network: none
  shell: none
  messaging:
    send_to: [triage-agent, findings-collector]
    receive_from: [engagement-manager, triage-agent]
  skills: [threat-modeling, prompt-injection-defenses]
triggers:
  - type: manual
  - type: message
    from: engagement-manager
    intent: review_request
```

Key design decisions:

- **Agents cannot exceed their declared permissions.** The runtime enforces this, not the agent prompt.
- **Agents use the same markdown prompt files as the public Astromechs repo, pinned by commit.** Mothership references a specific Astromechs commit SHA in its config. Upstream improvements flow in only through an explicit `mothership bump-astromechs` step that pulls latest, shows the diff, and requires approval before the new revision takes effect. This prevents upstream changes (accidental, malicious, or just experimental) from silently altering local agent behavior mid-engagement. Upgrade to signed-commit verification once Mothership has its own release signing infra.
- **Agent manifests are version-controlled and reviewed.** No agent runs without an approved manifest.

### Message broker

Agents communicate through a central broker, never directly.

**Message format:**

```json
{
  "id": "uuid",
  "timestamp": "iso8601",
  "from": "frontend-security-review",
  "to": "triage-agent",
  "intent": "findings_report",
  "sensitivity": "client-confidential",
  "org_context": "ttl:engagement:clientx",
  "payload": { ... },
  "signature": "hmac-sha256-of-payload"
}
```

**Broker rules:**

- Every message is signed by the sender. The broker verifies the signature before routing.
- The broker checks that the sender is allowed to send to the recipient (per both agents' manifests).
- The broker checks that the intent is in the sender's allowed output intents and the recipient's allowed input intents.
- Messages tagged with `sensitivity: client-confidential` cannot route to agents that have `network` permissions. This prevents data exfiltration by design.
- All messages are logged to an append-only audit log.
- Rate limiting per agent: no agent can send more than N messages per minute (prevents runaway loops).
- **Human approval gate:** messages with certain intents (execute_shell, send_external_message, delete_files) require human confirmation via the control interface before the broker delivers them. This is configurable per intent, not per agent.

**Why a broker instead of direct communication:**

- Single enforcement point for all security rules.
- Audit trail in one place.
- If an agent gets prompt-injected, the broker contains the blast radius. A compromised agent can't send "execute rm -rf /" to another agent because the broker checks both the intent and the recipient's permissions.
- Easy to add new agents without updating every existing agent's config.

### Organizational isolation

Mothership supports multiple "org contexts" with strict data separation:

```
/data/
├── ttl/
│   ├── engagements/
│   │   ├── clientx/        # client code, findings, reports
│   │   └── clienty/
│   ├── knowledge/           # finding library, references, RAG index
│   ├── content/             # drafts, voice configs, content calendar cache
│   └── memory/              # agent memory for TTL context
├── apisec/
│   ├── work/                # APIsec-related code and data
│   ├── knowledge/           # APIsec-specific references
│   └── memory/              # agent memory for APIsec context
└── shared/
    ├── models/              # Ollama model files (shared across contexts)
    └── skills/              # skill files (shared, read-only)
```

Rules:

- An agent running in `ttl:engagement:clientx` context can only access `/data/ttl/engagements/clientx/` and `/data/ttl/knowledge/` and `/data/shared/skills/`.
- An agent in `apisec` context cannot access any TTL engagement data, and vice versa.
- Mothership enforces this at the filesystem level (Linux namespaces or simple chroot/bind mounts).
- Memory is scoped per org context. An agent remembers TTL things when running TTL work and APIsec things when running APIsec work. No bleed.

### Skills system

Skills are the same format as Astromechs skills (SKILL.md files). Mothership makes them available to agents as retrievable context.

Differences from OpenClaw's skills:

- **Skills are passive reference documents, not executable code.** A skill provides knowledge; the agent decides what to do with it. Skills do not have shell access, network access, or any runtime permissions. This eliminates the entire class of "malicious skill" attacks that plague OpenClaw.
- **Executable actions are a separate concept: "tools."** Tools are code (Python functions, shell scripts) that an agent can invoke. Each tool has its own permission manifest, reviewed and signed. The set of available tools is small and controlled, not a community marketplace of 100+ unvetted plugins.

### Tools (executable actions)

Tools are the things that actually do stuff: run a scanner, write a file, send a Slack message, query a database.

```yaml
id: run-semgrep
name: Run Semgrep Scanner
type: tool
permissions:
  filesystem:
    read: ["${TARGET_DIR}/**"]
    write: ["${OUTPUT_DIR}/semgrep-results.json"]
  shell:
    allowed_commands: ["semgrep"]
    args_pattern: "--config auto --json --output ${OUTPUT_DIR}/semgrep-results.json ${TARGET_DIR}"
  network: none
inputs:
  - name: target_dir
    type: path
    required: true
  - name: output_dir
    type: path
    required: true
```

- Tools declare exactly what commands they can run, what files they can touch, and what network access they need.
- Mothership enforces these at invocation time.
- **Tools are not called directly — they are action agents addressed through the broker.** A review agent requesting a semgrep scan sends a `semgrep_scan` intent message to the `semgrep` action agent. This unifies the security model: tool invocations go through the same signature check, intent allowlist, sensitivity rules, human approval gates, and audit log as every other message. There is no side channel from agent to side effect. See Phase B of the build order for the full pipeline.
- New tools require a review (adding a file to the repo and getting it approved).

### Memory system

Persistent memory across sessions, scoped per org context.

- Short-term: conversation context within a single task/engagement. Stored in-memory, cleared when the task completes.
- Long-term: learned facts, preferences, project context. Stored as files in the org context's memory directory.
- Format: similar to the Cowork memory system (markdown files with frontmatter, indexed by a MEMORY.md file).
- **Memory is a broker primitive, not an agent.** Agents send `memory_read` and `memory_write` intents through the broker; the handler on the other end is deterministic library code, not a model. No LLM ever holds the full memory store in its context. This eliminates the "prompt-inject the memory-manager and own everything" failure mode — you can't prompt-inject a library. The "everything is a message" principle is preserved; the endpoint just isn't an agent.
- Scope enforcement: memory intents carry the sender's `org_context`, and the handler refuses any read or write outside that context's memory directory. An agent can only touch its own scoped slice.
- Encryption at rest: all memory files encrypted with a subkey derived from the master passphrase (see Lifecycle section). Decrypted only in broker memory during runtime, never written back in plaintext.

### Control interface

How Gavin (and potentially Brenda) interact with the runtime. The key rule: **approval authority for gated intents never leaves the box.**

- **Local web UI (primary for anything sensitive):** FastAPI + simple frontend. Dashboard showing running agents, recent activity, audit log, memory index. Handles all human-approval gates for sensitive intents. Handles any display of client-confidential findings. Bound to localhost + tailnet only — no public exposure, no cloud dependency. This is the *only* place a gated intent can be approved.
- **Slack bot (notifications and non-sensitive commands only):** Posts alerts, status pings, scan completions, and "approval needed → open web UI to review" notifications. Can receive non-sensitive commands (start a scan, list running agents). *Cannot approve gated intents. Cannot display client-confidential payloads.* Bot token scopes are minimized: send/read in one private channel, nothing else. This decouples cloud-service compromise (Slack account phish, token theft, workspace takeover) from box compromise.
- **CLI:** For Gavin when he's SSH'd into the box. Full capabilities including gated approval (SSH is already a strong authentication path to the box itself).
- **Phase D upgrade:** gated approvals in the web UI require a hardware-key touch (YubiKey / WebAuthn). Web UI compromise alone is then insufficient — attacker also needs physical possession of the key.

### Lifecycle and secret bootstrap

The security model only holds if the box's state transitions are thought through, not just steady state. Every primitive in this spec assumes keys and secrets are already in place (HMAC signing keys per agent, memory encryption master key, Slack OAuth token, vault creds, Astromechs pin). Lifecycle defines how they get there.

**Cold boot (Phase B design):**

1. Box powers on. Mothership runtime does not auto-start any agent.
2. Gavin logs into the box (SSH or local console) and runs `mothership unlock`.
3. `mothership unlock` prompts for the master passphrase on the terminal — interactive, never in a file, never in an env var.
4. The master passphrase decrypts a single secrets blob on disk (age-encrypted) containing: per-agent HMAC private keys, memory subkeys per org context, Slack bot token, vault credentials, any other long-lived secrets.
5. Decrypted secrets live in broker memory only. Never written back in plaintext. Never logged.
6. Broker starts. Agents start only after the broker is up and their HMAC keys are loaded.
7. Startup order is enforced — any agent that attempts to start before the broker is ready is refused.

**Warm boot / unplanned reboot:** same as cold boot. Box processing client data *should* require a human unlock after a reboot — it's a feature, not a bug. Monitoring agents will be dark until you unlock. Alerting on that downtime is a separate concern, handled by an external dead-man's-switch (not by Mothership itself, since Mothership is down).

**Shutdown:** `mothership shutdown` instructs the broker to drain in-flight messages, flush audit log, zeroize secrets in memory, and exit. Agents are terminated in reverse-startup order.

**Restore from backup:** encrypted backups contain the data directories and the secrets blob. Restoring onto a new box requires the master passphrase (and, in Phase D, the TPM-sealed variant requires re-sealing to the new TPM).

**Secret rotation:** `mothership rotate-key <agent-id>` generates a new HMAC keypair for an agent, re-encrypts the secrets blob, and reloads the broker. Memory subkeys can be rotated with a re-encrypt pass.

**Phase D upgrade — TPM sealing:** the secrets blob gets sealed to the box's TPM plus a PIN. `mothership unlock` prompts for the PIN instead of a full passphrase; TPM unseals only if the box is in the same boot state. Survives reboots without a full passphrase re-entry, still useless if the SSD is stolen, still bound to the specific hardware. Interactive unlock remains the fallback.

### Monitoring and scheduled producers

**Purpose.** These agents exist to reduce cognitive load for a solo operator running multiple concurrent concerns (TTL engagements, APIsec work, content, prospects, domain health, dependency hygiene). They are Gavin's peripheral vision. While he's focused on client work or asleep, they watch things he cares about and populate review queues so that when he has attention, the right work is already staged. They are *not* autonomous operators. They direct attention; they do not take action on his behalf.

**The three-level rule.** Every agent output falls into one of three levels. Monitors may produce levels 1 and 2 only. Level 3 is categorically forbidden for any monitor.

1. **Observation** — read-only from external or local sources, emits messages that become notifications and audit entries. Never persisted as state. (GitHub events, cert checks, dep CVEs, email flags.)
2. **Staged production** — writes output to a designated, scoped review area on the local filesystem. This *includes writing code, drafting posts, generating reports, producing scored candidate lists, any local file creation*. Writes have no real-world effect until a human promotes them through the web UI. A draft is not a publish. A generated patch is not a merge. A scored row is not an outreach email. A report is not a delivery.
3. **Direct action** — anything with an effect outside the box: pushing a commit, sending an email, publishing a post, hitting an API that mutates external state, renewing a cert, triggering a scan on engagement data, writing into an engagement directory. **Monitors cannot do this. Ever. No exceptions, no per-monitor carve-outs.**

The test is simple: if the output causes *anyone other than Gavin* to see or be affected by it, it's level 3 and a monitor cannot produce it. Local file creation, including code, is fine — it sits in a review queue until Gavin acts on it.

**Message classes (broker-enforced).** Two new low-privilege intent classes:

- `observation` — routed to notification delivery and the audit log. Payload may contain arbitrary text; delivered as plain text with no actionable links (see Control interface).
- `staged_output` — writes to a scoped path under the org context's review queue (e.g., `/data/ttl/review-queue/content-drafts/`, `/data/ttl/review-queue/code-suggestions/`). The broker enforces that the path is inside the declared queue and nowhere else. No action agent in the system treats a review-queue path as a trigger. Promotion to real effect is a human-initiated flow through the web UI.

**Catalog of initial monitors (all Level 1 or 2):**

- **GitHub monitor** *(observation)* — new issues, PRs, vulnerability alerts across TTL repos. Read-only API access. Summaries posted as notifications.
- **Email monitor** *(observation)* — IMAP read-only. Flag messages matching domain/keyword rules. Never replies, never sends.
- **Cert/domain monitor** *(observation)* — TLS expiry, DNS drift, security headers on TTL domains. Never renews, never modifies DNS.
- **Dependency monitor** *(observation)* — watches TTL lockfiles against a local OSV database. Flags new CVEs. Never patches, never opens PRs.
- **Content calendar drafter** *(staged production)* — checks Notion calendar, drafts due posts to `/data/ttl/review-queue/content-drafts/`. Never publishes, never writes to Notion itself.
- **Prospect scanner** *(staged production)* — scores job board signals, writes candidate rows to `/data/ttl/review-queue/prospects/`. Never sends outreach, never writes to any external system.
- **Code suggestion monitor** *(staged production)* — e.g., dependency upgrade patches generated locally. Writes patch files to `/data/ttl/review-queue/code-suggestions/`. Never pushes, never opens PRs, never writes into engagement source directories.

**Shared constraints for all monitors.** Derived from the monitor security discussion:

- Cron trigger in manifest. Missed runs on resume are skipped, not backfilled.
- Each monitor runs as its own unix user with its own secrets slice (per-monitor isolation).
- External credentials minimum-scoped, read-only wherever the service supports it.
- All network egress through the runtime egress proxy with a per-monitor hostname allowlist. No inbound network. No webhooks, ever — polling only.
- Broker-side heartbeat: the broker tracks the last time each monitor produced output; missing heartbeat raises an alert. External dead-man's-switch watches Mothership itself.
- Notifications to Slack are plain text, no links, no buttons. Any review flow happens in the local web UI, not from a Slack action.
- Staged outputs are readable by the web UI's review interface and by the engagement agents *only* when a human explicitly promotes them into an engagement context.

**The review queue is itself a trust boundary.** A staged output was produced by an agent whose inputs came from external, attacker-influenceable sources (GitHub issue text, email bodies, web pages, dependency metadata). The content of the review queue is therefore *untrusted by construction*, regardless of which monitor produced it. Consequences:

- Any read from a review-queue path goes through the Trust boundary pre-processor (delimiting, spotlighting, canary injection, pattern scrubbing) exactly as if it were freshly scraped from the internet. A drafted patch file is not safer because a local monitor wrote it — its *content* is still shaped by upstream untrusted input.
- Engagement agents cannot implicitly read the queue. The queue is not on any agent's default read scope. Contents only enter an engagement context through an explicit, human-initiated promotion.

**Promotion is the one place level-3 action happens for monitor-originated content.** Moving a staged output from the queue into a real-world effect — publishing a drafted post, applying a generated patch, sending a scored prospect into outreach, filing a report — is a distinct intent class: `promote_from_queue`. It carries full weight:

- Always a gated intent. Human approval required through the local web UI, never through Slack.
- Always passes through the sensitive intent review (fast-model second-pass) before the approval gate fires. The reviewer checks that the staged content matches what a human would reasonably expect given the queue's purpose and the promotion target.
- Always logged with the full lineage: originating monitor, originating external source (e.g., the specific GitHub issue URL or email message-id), staged output path, approving human, timestamp, target action.
- Promotion is the *only* mechanism by which a monitor's output can influence any level-3 outcome. There is no other path. If you find yourself wanting one, the answer is to add a staged output + a promotion flow, not to grant a monitor direct action.

---

## Security model summary

| Threat | Mitigation |
|--------|-----------|
| Prompt injection in one agent affects others | Broker enforces message routing rules; agents can't send intents they're not allowed to; sensitivity tagging prevents data exfil |
| Malicious skill/plugin | Skills are passive documents, not code. Tools are executable but reviewed, signed, and permission-scoped |
| Agent tries to access wrong client data | Org context isolation at filesystem level (namespaces/bind mounts); runtime enforces path scoping per agent manifest |
| TTL and APIsec data cross-contamination | Separate org contexts with no shared data paths (only shared models and read-only skills) |
| Credential theft | Credentials in a secrets manager (not env vars); agents access secrets through a vault tool with per-secret ACLs; secrets never appear in agent memory or logs |
| Runaway agent (infinite loop / resource exhaustion) | Rate limiting on messages; timeout on inference calls; resource caps per agent (memory, CPU) |
| Exposed admin port | No admin port. Control via authenticated Slack bot, local-network-only web UI, or SSH CLI |
| Unauthorized access to web UI | Bind to localhost or LAN only; require auth; no public exposure |
| Audit trail tampering | Append-only log file; consider signing log entries |

---

## Tech stack (planned)

| Component | Tool | Why |
|-----------|------|-----|
| LLM inference | Ollama | Simple, supports ROCm for Ryzen AI, manages models, HTTP API |
| Primary model | Best 70B+ open model at build time | Code review quality |
| Fast model | Best 7-14B open model at build time | Triage, summarization, routing |
| Embeddings | nomic-embed-text via Ollama | Lightweight, good quality for RAG |
| Vector store | LanceDB | Embedded (no server), fast, Python-native |
| Message broker | Custom (Python, Redis or SQLite-backed) | Simple; don't need Kafka for a single-box system |
| Mothership runtime | Custom Python | Reads agent manifests, enforces permissions, routes to Ollama |
| Scanning tools | Semgrep, Gitleaks, Trivy, nuclei | Open source, industry standard |
| Web UI | FastAPI + lightweight frontend | Local dashboard, no heavy framework needed |
| Secrets | age encryption or SOPS | File-based, no server dependency |
| Scheduling | cron + systemd timers | Simple, reliable, no orchestration overhead |
| OS | Linux (Ubuntu Server or Fedora) | Best Ollama/ROCm support for AMD |

---

## Build order (when the time comes)

This is not a "build it all at once" project. Build the pieces, use them manually, then unify.

**Phase A: standalone pieces (use immediately)**
1. Install Ollama, pull models, test inference quality on code review tasks.
2. Write a simple Python script that reads an Astromechs agent .md file, constructs the prompt, sends code to Ollama, collects findings. Test on a TTL repo.
3. Set up the scanning pipeline (Semgrep + Gitleaks + Trivy) as a shell script. Run on a TTL repo.
4. Set up LanceDB + nomic-embed for RAG. Index the finding library and MCP checklist.
5. Use all of the above manually on 1-2 client engagements. Learn what works.

**Phase B: lightweight orchestration**

The guiding principle of Phase B: **every action an agent takes is a message through the broker.** Filesystem writes, tool invocations, memory reads, external calls — all of them. This makes the broker the single enforcement point and the single audit stream. No side channels.

6. **Agent manifest format + runtime loader.** Mothership core reads manifests, validates them against a schema, and refuses to start any agent whose manifest fails validation or declares capabilities outside an allowlist.

7. **Message broker (capability broker).** SQLite-backed append-only message store with a routing layer. Enforces all of:
    - Sender signature verification (HMAC, key per agent).
    - Sender→recipient allowed by both manifests.
    - Intent is in sender's declared output intents and recipient's declared input intents.
    - Sensitivity tag doesn't violate any downstream capability rule (e.g., `client-confidential` cannot route to any agent with `network` permission).
    - Per-agent rate limits.
    - Human approval gate for configured intents.

8. **Tools as action agents.** No direct tool calls. Each tool (semgrep, gitleaks, file write, slack send) is wrapped as a minimal "action agent" that only accepts specific intents and only performs the corresponding action. A review agent doesn't *call* `run_semgrep` — it sends a `semgrep_scan` intent to the `semgrep` action agent. Action agents have their own manifests declaring their filesystem, shell, and network scope. The broker is now the *only* path to any side effect in the system.

9. **Trust boundary pre-processor.** Every input entering a model's context from outside the runtime (client code being reviewed, scan output, message payloads from lower-trust agents, memory contents, web fetch results, **and any read from a review queue** — see the Monitoring section for why review-queue contents are untrusted by construction) flows through a deterministic pre-processor that:
    - Wraps the untrusted content in unambiguous delimiters.
    - Applies spotlighting (hash-mark or similar per-token tagging so the model can distinguish data from instructions).
    - Strips or escapes known injection patterns (role markers, system-prompt impersonation, embedded tool-call syntax, ANSI escapes, zero-width characters).
    - Injects a unique canary token per input chunk; if that canary later appears in any outbound message, it's logged as a probable leak attempt.
    - No ML in the loop — pure rules, auditable, zero false positives on legitimate security content.

10. **Sensitive intent review (fast-model second-pass).** The broker inspects every message's intent. Intents in the "sensitive" set (any action agent invocation, any external send, any filesystem write outside a narrow whitelist, **any `promote_from_queue` intent that moves a staged output from a review queue into a real-world effect**, anything flagged for human approval) get a second-pass review *before* being routed:
    - The fast 7-14B model receives a narrow, fixed prompt: *"Agent {id}'s declared job is {role}. Here is the output it wants to send: {payload}. Does this match its declared job, or does it look like the agent was redirected?"*
    - Review verdict (pass/fail/uncertain) and reasoning are logged alongside the message.
    - Failed reviews block the message and raise an alert. Uncertain reviews escalate to human approval regardless of the intent's default gate setting.
    - The review model runs with its own prompt context that cannot be influenced by the payload — the payload is spotlit, delimited, and treated as inert text.
    - Cost: one fast-model call per sensitive message. Acceptable given the fast tier is already loaded in memory.

11. **Minimum viable org context isolation.** Pulled forward from Phase D per the phase-ordering discussion:
    - Separate unix users per org context (`ttl-ctx`, `apisec-ctx`).
    - Separate data directories with unix-level permission separation — no shared filesystem access between contexts.
    - Runtime refuses to start any agent whose declared `org_context` doesn't match the unix user it was launched under.
    - Env vars and process args from one context are never visible to the other.
    - This is not cryptographic isolation. It *is* a real boundary that a bug or a successful prompt injection cannot casually cross. Namespaces/seccomp come in Phase D.

12. **Cold-boot unlock flow + secrets blob.** Implement `mothership unlock` — interactive passphrase prompt, decrypts the age-encrypted secrets blob into broker memory, starts broker, starts agents in dependency order. Secrets never written back in plaintext, never logged. See the Lifecycle section for the full sequence.

13. **Local web UI (minimal).** FastAPI + simple frontend bound to localhost + tailnet only. Minimum viable surface: view audit log, view running agents, approve/deny gated intents. This is the only approval path for sensitive intents. No public exposure, ever.

14. **Slack bot (notifications + non-sensitive commands only).** Posts alerts, scan completions, and "approval needed → open web UI" links. Accepts non-sensitive commands only. Cannot approve gated intents. Cannot render client-confidential payloads. Scopes minimized to a single private channel. Slack token lives in the secrets blob, loaded only after unlock.

15. **First end-to-end engagement.** Run a real (or realistic) engagement through Mothership: trigger a review, watch the pipeline flow through trust boundary → agent → broker → intent review → action agent → findings output. Confirm the audit log reconstructs the full sequence of events. Confirm that an injected payload in the client code gets blocked or flagged at the trust boundary *or* the intent review stage. Learn what's slow, what's annoying, what's wrong. Iterate before adding more agents.

**Phase C: always-on monitoring + behavioral defense**
16. Add the heartbeat/monitor agents (GitHub, certs, deps, content calendar).
17. Add the prospect scanner.
18. Expand the web UI from minimum viable (Phase B) to a full dashboard: audit log search, memory browser, per-agent activity, behavioral baselines.
19. **Behavioral monitor (broker-side).** The broker already logs every message. Add a watchdog that maintains per-agent baselines — intent frequency, recipient distribution, payload size, action-agent invocation rate, canary-token hit rate. Alert on drift. Auto-quarantine agents that deviate sharply (e.g., a review agent that suddenly tries to send external messages, or whose output starts matching canary tokens from earlier inputs). Zero model cost — pure stats on the audit log. Catches the slow, novel, and multi-step attacks that per-message review misses.

**Phase D: hardening and polish**
20. Upgrade org context isolation from unix-user-level (Phase B) to Linux namespaces / cgroups / seccomp for defense in depth.
21. TPM-sealed secrets blob + PIN unlock (upgrade from Phase B interactive passphrase).
22. Hardware-key (YubiKey / WebAuthn) requirement on gated-intent approvals in the web UI.
23. Expand secrets management: per-secret ACLs, vault action agent for runtime secret fetching, rotation schedule.
24. Audit log signing (tamper-evident hash chain).
25. Write operational runbooks.
26. Evaluate whether to open-source the runtime (the generic orchestration layer, not the TTL-specific agents or data).

---

## APIsec usage notes

- Runs as a separate org context (`apisec`) with its own data directory, memory, and agent configs.
- Uses the same models and Mothership instance as TTL (no duplicate infrastructure).
- APIsec agent configs are NOT stored in TTL repos. Keep them in a separate, private location on the box.
- No APIsec data flows through TTL agents or vice versa.
- If APIsec has its own compliance requirements about where code is processed, verify this setup satisfies them before using it for APIsec work.

---

## Open questions

1. **Model selection:** Which 70B+ model gives the best code review results as of build time? Needs a shootout.
2. **ROCm on Ryzen AI:** How well does Ollama leverage the NPU? May need testing/tuning.
3. **Slack bot hosting:** Does the bot run on the GMKtek itself (requires the box to be always-on and network-reachable) or on a small cloud instance that proxies to the box?
4. **Brenda's access:** Does she need access to the control interface? If so, the web UI needs to be polished enough for non-CLI users.
5. **Backup strategy:** 2TB SSD is the single copy of engagement data. Encrypted backups to a second drive or cloud storage.
6. **Open-sourcing Mothership:** If we do this, it becomes a companion to Astromechs: "here are the agents (open), here's how to run them locally (open), here's the data and tooling we built on top (private)." Good flywheel, but only after Mothership is stable and the security model is proven.
