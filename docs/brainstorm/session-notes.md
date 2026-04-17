# Mothership Brainstorming — Session Notes

Running log of brainstorming sessions. Pick up from the "Next up" list at the bottom.

---

## Session 1 — 2026-04-15

Starting point: Gavin dropped the initial `early-spec.md` (idea-stage, not building yet). First pressure-testing pass against the spec before it hardens.

### Decisions made and folded into `early-spec.md`

**1. Capability broker unification (was: broker vs. tool-call gate)**
- Tools are no longer called directly. Every tool is an "action agent" addressed through the broker.
- Filesystem writes, tool invocations, memory reads — all messages. Broker is the single enforcement point and single audit stream.
- OS-level sandboxing (namespaces/seccomp) deferred to Phase D as defense-in-depth.

**2. Astromechs pinning**
- Mothership references a specific Astromechs commit SHA. `mothership bump-astromechs` pulls latest, shows diff, requires approval.
- Upgrade to signed-commit verification in Phase D once Mothership has its own release signing infra.

**3. Phase ordering — isolation pulled forward**
- Minimum viable org context isolation (separate unix users + path scoping + context check at agent startup) now lives in Phase B, not Phase D.
- Linux namespaces / cgroups / seccomp upgrade stays in Phase D as defense-in-depth.
- Reason: running TTL and APIsec on the same box without real isolation was aspirational marketing until D.

**4. Runtime prompt-injection detection**
Layered defense, skipping input classifiers:
- **Structural input firewall** (Phase B): deterministic, no ML. Delimits, spotlights, pattern-scrubs untrusted input. Injects canary tokens.
- **Sensitive intent review** (Phase B): fast 7-14B model second-pass on any sensitive intent before broker routes it. Narrow prompt: "agent's declared job is X, does this output match?"
- **Behavioral monitor** (Phase C): broker-side baselines per agent, alert on drift. Pure stats on audit log.
- Skipped input classifiers — too many false positives on legitimate security content (client code literally contains "bypass authentication" strings).

**5. Memory is a broker primitive, not an agent**
- Eliminated the memory-manager agent. Agents send `memory_read` / `memory_write` intents; the handler is deterministic library code, not a model.
- No LLM ever holds the full memory store in context. Can't prompt-inject a library.
- Scope enforcement: intents carry sender's `org_context`, handler refuses any read or write outside that context's memory directory.

**6. Control plane split — Slack cannot approve gated intents**
- Local web UI (localhost + tailnet) is the *only* approval path for sensitive intents and the only place client-confidential payloads are rendered.
- Slack bot: notifications and non-sensitive commands only. Scopes minimized to one private channel. Cannot approve, cannot render confidential payloads.
- Phase D: YubiKey/WebAuthn touch required on gated-intent approvals in the web UI.
- Reason: decouple cloud-service compromise from box compromise. Slack account phish or workspace takeover must not translate to box takeover.

**7. Lifecycle and secret bootstrap (new section in spec)**
- Cold boot: `mothership unlock` prompts interactively for master passphrase. Decrypts age-encrypted secrets blob into broker memory. Broker starts. Agents start in dependency order only after broker is ready.
- Unplanned reboot = manual unlock required. Feature, not bug.
- Shutdown, rotation, restore-from-backup all spelled out.
- Phase D upgrade: TPM-sealed blob + PIN.

**8. Monitors as a distinct agent class**
The biggest architectural move of the session. Monitors are now treated as a separate class with their own rules.

Purpose: reduce cognitive load for a solo operator. Monitors are peripheral vision, not autonomous operators. They direct attention; they do not take action.

The three-level rule:
- **Level 1 — Observation**: read-only external sources, emits notifications. (GitHub, cert, dep, email monitors.)
- **Level 2 — Staged production**: writes files locally to a scoped review queue. *Includes code, drafts, patches, reports, scored prospects.* No real-world effect until a human promotes. (Content drafter, prospect scanner, code suggestion monitor.)
- **Level 3 — Direct action**: anything with an effect outside the box. **Categorically forbidden for any monitor. No carve-outs.**

Test: if the output causes anyone other than Gavin to see or be affected by it, it's level 3.

Two new low-privilege broker intent classes:
- `observation` — routed to notification delivery + audit log.
- `staged_output` — writes to a scoped review-queue path; broker enforces path scope.

Shared monitor constraints:
- Per-monitor unix user + secrets slice.
- Egress proxy with per-monitor hostname allowlist. Poll-only, no webhooks ever.
- Broker-side heartbeat + external dead-man's-switch (different delivery channel, not through Slack).
- External creds minimum-scoped, read-only where possible.
- Notifications are plain text, no links, no buttons. Action flows happen in the web UI.
- Missed cron runs on resume are skipped, not backfilled.

**9. Review queue as a trust boundary + promotion as the chokepoint**
- Review-queue contents are *untrusted by construction* — regardless of which local monitor wrote them, the content was shaped by upstream attacker-influenceable input.
- Any read from a review-queue path flows through the Trust boundary pre-processor, same as fresh external input.
- Engagement agents cannot implicitly read the queue. Contents only enter an engagement context through explicit human-initiated promotion.
- `promote_from_queue` is a distinct gated intent class. Always passes through sensitive intent review AND human approval in the web UI. Logged with full lineage (originating monitor, originating external source, staged path, approver, timestamp, target action).
- This is the *only* path by which monitor-originated content can cause a level-3 effect.

### The invariant the spec now holds

> No byte of external attacker-influenceable content reaches a level-3 action without passing through: (a) the Trust boundary pre-processor, (b) a fast-model sensitive intent review, and (c) a human approval in the local web UI.

### Still unstressed / not yet discussed

These were raised during the session but deferred:

1. **Backup story** — 2TB single-SSD contains encrypted client data AND the only copy of the secrets blob. One failed drive = very bad day. Needs: encrypted backup destination, restoration drill, secrets-blob backup strategy, test-restores on a cadence.

2. **Approval fatigue** — The security model depends on gated approvals being *meaningful*. Several intent classes now route through the gate (action agents, external sends, `promote_from_queue`, uncertain intent reviews). If daily approval volume is high, rubber-stamping becomes inevitable. Needs a UX/design pass: batching, risk-tiering, default-deny timeouts, "why is this sensitive?" justification in every prompt, review-after-the-fact sampling. Worth stress-testing whether the human-in-the-loop actually scales.

3. **Monitor-specific open questions that came up but weren't fully resolved:**
    - Cron lifecycle on resume after long downtime — confirmed "skip, don't backfill" but hasn't been reconciled with situations where the missed run matters (e.g., cert expiry alert during the downtime window).
    - What happens to a monitor that *wants* a credential the vault doesn't have yet? Bootstrap story for adding a new monitor needs thought.

4. **Model supply chain** — Ollama pulls model weights from a registry. Weights aren't signed. A compromised weight file is a prompt-injection payload baked into the model itself. Needs: pinned model hashes in config, verification at pull time, decision about whether to mirror weights locally.

5. **Model selection shootout methodology** — spec has it as an open question but no method. Needs a reproducible eval harness against a known finding corpus before picking the primary model.

6. **Ryzen AI / ROCm reality check** — spec assumes it works. Likely doesn't work as smoothly as the spec implies. Worth early validation before committing the stack.

### Next up

**Priority 1 — Approval fatigue.** Everything hinges on human approvals being real. This should be stress-tested before the spec gets any bigger. If it doesn't scale, the security model falls apart in practice even if it holds in theory.

**Priority 2 — Backup story.** Easier to reason about, lower architectural stakes, but a real operational gap that could cost the whole dataset.

**Priority 3 — Model supply chain.** Smaller in scope but lives on the same layer as the rest of the security invariant and shouldn't be left dangling.

Pick up with: "Let's work through approval fatigue."
