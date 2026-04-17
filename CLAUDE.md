# Mothership

Secure runtime substrate for locally-deployed AI agents. Idea/spec stage — **no code yet** (0 commits on `main`). Target is an open-source, locally-hosted agent runtime for TamperTantrum Labs (Brenda's business) that hosts agents like Hermes in environments where OpenClaw-class runtimes aren't trusted.

## Positioning

Mothership is **not** a competing agent product. It's the runtime substrate beneath the agent loop: supervisor below the API plane, signed skills with capability declarations, semantic firewall, OS-native secrets, HMAC-chained audit. Nous Research's Hermes is the reference architecture at the agent layer; OpenClaw is the counter-design target. See `use-cases.md` for who ships on top of it and why.

## Read Before You Work Here (in order)

1. **`design-principles.md`** — the 8 non-negotiable principles. **Every spec must cite which principles it honors and flag any tension.** Amending a principle requires a dated entry in its "Principle changes" section tied to new research, a threat model change, or an incident.
2. **`use-cases.md`** — who deploys Mothership, what tasks they can't do today without it, and the V1 beacon portfolio (UC1 + UC2 + UC3). Every spec must cite at least one use case from this doc.
3. **`docs/brainstorm/early-spec.md`** — the full system spec (hardware target, broker-enforced capability model, trust-boundary pre-processor, monitor levels, phases A–D).
4. **`docs/brainstorm/session-notes.md`** — brainstorm history.
5. **`research/hermes-agent.md`** — reference architecture for the agent-layer pattern Mothership hosts. Second canonical copy at `Projects/Research/hermes-agent.md`.
6. **`../../Research/openclaw.md`** (up at the workspace root) — failure-case evidence the design principles are written against.

## Working Rules

- **No implementation before V1 engineering-forcing spec lands.** This repo is design-reviewed, not code-reviewed. Do not scaffold a Python/TS project, add infra, or create agents until the V1 spec derived from the beacon UCs is committed here.
- **Every spec cites principles + use cases.** A spec that advances no use case is out of scope until `use-cases.md` is amended. A spec that violates a principle is rejected unless the principle is amended in the same PR with evidence.
- **Principles outlive iterations on the spec.** If a spec and a principle disagree, the principle wins.
- **Research material is evidence, not law.** `research/hermes-agent.md` and `../../Research/openclaw.md` inform design decisions but don't override principles or use cases.

## Planned Stack (from `docs/brainstorm/early-spec.md`)

Informational only — subject to change as the V1 spec locks.

- **Hardware target:** GMKtek box, 96GB unified memory, Ryzen AI (NPU), 2TB SSD
- **Inference:** Ollama (70B+ primary, 7–14B fast, embedding model). No cloud LLM for sensitive paths.
- **Enforcement:** Message broker as the single enforcement point. All agent ↔ tool, agent ↔ agent, agent ↔ memory, agent ↔ filesystem traffic passes through the broker. No side channels.
- **Capability model:** Agent manifests declare permissions (filesystem, network, shell, messaging, skills). Runtime enforces, not the prompt.
- **Trust boundary pre-processor:** Deterministic input sanitization (delimiters, spotlighting, canary tokens, pattern scrubbing) on every attacker-influenceable input before it reaches a level-3 action path.
- **Sensitive-intent review:** Fast-model second-pass on every sensitive action before routing.
- **Isolation:** Filesystem-level org-context separation (TTL / APIsec / shared).

## Build Phases (aspirational — from `docs/brainstorm/early-spec.md`)

- **Phase A** — Standalone pieces (Ollama, scanning pipeline, RAG)
- **Phase B** — Lightweight orchestration (broker, action agents, trust boundary, org isolation, web UI, Slack bot)
- **Phase C** — Always-on monitoring + behavioral defense
- **Phase D** — Hardening (namespaces/cgroups, TPM, YubiKey, audit log signing)

## What Not to Do

- Do not pick tools, languages, or directory layouts before the V1 spec lands. The current TypeScript-vs-Python / Ollama-vs-llama.cpp / async-vs-sync conversations belong in spec PRs, not in speculative scaffolding.
- Do not reference APIsec data, tools, or accounts here. Mothership is TamperTantrum Labs territory (Brenda's business), multi-tenancy-separated from Gavin's APIsec work.
- Do not port Hermes or OpenClaw code wholesale. Both are studied as patterns; Mothership's differentiation lives at the substrate layer, not the agent layer.
- Do not write a README yet. The repo is intentionally spec-only until V1 engineering begins.
