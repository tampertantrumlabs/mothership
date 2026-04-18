# OpenClaw: A Comprehensive Research Document

*Compiled April 2026*

---

## 1. Executive Summary

**OpenClaw** (formerly **Clawdbot**, briefly **Moltbot**) is a free, open-source, self-hosted autonomous AI agent framework. It turns any LLM into a persistent "personal AI assistant" you interact with through the messaging apps you already use — WhatsApp, Telegram, Discord, Slack, Signal, iMessage, and dozens more. Instead of opening a chat window in a browser, you just message your bot like a coworker and it acts on your behalf: reading email, running code, browsing the web, orchestrating Claude Code/Codex sessions, submitting PRs, handling calendar, and more.

It was built by Austrian developer **Peter Steinberger** as a weekend project in November 2025 and became the fastest-growing open-source project in GitHub history, passing the Linux kernel in stars by mid-February 2026. Steinberger joined OpenAI on February 14, 2026, and the project was transferred to an independent non-profit foundation.

OpenClaw is simultaneously one of the most hyped AI tools of the last year *and* one of the most security-problematic. As of April 2026, it has accumulated **138 tracked CVEs** (7 Critical, 49 High), including a one-click RCE (CVE-2026-25253, CVSS 8.8), a massive marketplace supply-chain problem, and a separate backend breach that leaked ~1.5 million API tokens.

---

## 2. History and Origin

| Date | Event |
|---|---|
| Nov 2025 | Peter Steinberger hacks together "WhatsApp Relay" over a weekend; publishes it as **Clawdbot** (pun on "Claude") |
| Late Nov – Jan 2026 | Viral growth; 100K+ GitHub stars in ~2 months; 2M visitors in a single week |
| Jan 26, 2026 | DepthFirst researchers privately disclose CVE-2026-25253 |
| Jan 27, 2026 | Anthropic sends a trademark complaint; project renamed to **Moltbot** (lobster-molting theme) |
| Jan 29–30, 2026 | Renamed again to **OpenClaw**; v2026.1.29 patch released for CVE-2026-25253 |
| Jan 30, 2026 | Hits 100,000 GitHub stars |
| Feb 3, 2026 | Public disclosure of CVE-2026-25253 via SecurityWeek & The Hacker News |
| Feb 14, 2026 | Steinberger announces he's joining OpenAI; project moves to an independent foundation backed by OpenAI |
| Feb 16, 2026 | Hits 200,000 stars, passing the Linux kernel |
| Mar 2, 2026 | ~247K stars, 47.7K forks (per Wikipedia) |
| Mar 2026 | OpenClaw partners with VirusTotal for skill scanning; ClawHub implements mandatory code review |
| Apr 2026 | 138 CVEs tracked; 60+ since January |

**Creator:** Peter Steinberger (@steipete) — 13-year veteran iOS developer, founder of PSPDFKit (PDF SDK that powers apps used by nearly a billion people, acquired in a nine-figure exit in 2024). Describes himself as "Clawdfather" on GitHub. Self-identifies as a "vibe coder" and averaged ~6,600 commits per month during peak.

**Current stewardship:** Independent non-profit foundation as of Feb 2026. Financial backing disclosed from OpenAI.

---

## 3. What OpenClaw Does

### Core concept
OpenClaw runs as a **local gateway daemon** on your Mac, Windows, or Linux machine. It bridges:

- **Messaging platforms** (input/output surface for the human)
- **LLMs** (the "brain")
- **Your local tools** (filesystem, browser, shell, installed apps)
- **Cloud services** via skills/plugins (Gmail, Calendar, GitHub, Sentry, Railway, 1Password, Spotify, Hue, Notion, Obsidian, etc.)

You message it in Discord or Telegram like you'd message a coworker, and it executes tasks on your machine autonomously.

### Representative use cases (from the official site and user testimonials)
- **Coding agent orchestration:** Spawn and manage Claude Code / Codex / Gemini / Cursor sessions from your phone. "fix tests" via Telegram runs the loop, sends progress every N iterations, opens a PR.
- **DevOps loops:** Inspect failed Railway builds, diagnose, change deployment config, redeploy, submit PR — over voice.
- **Error-to-PR pipelines:** Sentry webhook → autonomous diagnosis → fix → PR.
- **Inbox/workflow automation:** Clearing 10,000 emails, reviewing decks, drafting posts, optimizing Google Ads.
- **Personal assistant:** Timeblock tasks, weekly reviews from meeting notes, pre-meeting briefings, calendar-conflict resolution, invoice creation, flight-status monitoring.
- **Family/home:** School deadline monitoring, smart-home control (Hue lights), reminders, voice calls.
- **Agent-to-agent:** Connect to Moltbook (an experimental social network for AI agents) and other agent platforms.
- **Proactive behavior:** Heartbeats (periodic check-ins), cron jobs, background observation.

### Supported messaging channels (25+)
WhatsApp, Telegram, Slack, Discord, Google Chat, Signal, iMessage (via BlueBubbles), IRC, Microsoft Teams, Matrix, Feishu, LINE, Mattermost, Nextcloud Talk, Nostr, Synology Chat, Tlon, Twitch, Zalo, Zalo Personal, WeChat, QQ, WebChat.

### Supported LLMs
Any model with an API: Claude (Opus 4.6), OpenAI (GPT-5.2 Codex), Google Gemini (3.1 Pro), Kimi K2.5, GLM-5.1, DeepSeek, MiniMax, and local models via Ollama. A built-in subscription bundles five flagship models.

---

## 4. How It Works (Architecture)

### High-level flow
```
[Discord/Telegram/WhatsApp/etc.]
          ↓  (channel adapter polls/receives message)
[OpenClaw Gateway Daemon on your machine]
          ↓  (routes to session + loads skills)
[LLM API call]  ←→  [Skills / Tools / Shell / Browser / Filesystem]
          ↓  (response streamed back)
[Same chat surface the user messaged from]
```

### Key architectural pieces
- **Gateway:** The control plane. Node.js service that exposes a WebSocket API on `ws://127.0.0.1:18789/` (or `wss://host:18789/`). Runs as a `launchd` (macOS) or `systemd` (Linux) user service so it stays up.
- **Control UI:** A single-page web app at `/chat`, built with Lit web components, used to configure the gateway and chat directly with the agent in a browser.
- **Channel adapters:** One module per messaging platform. Each is responsible for authenticating to the service, receiving messages, and sending replies.
- **Skills system:** A skill is a directory containing a `SKILL.md` (metadata + natural-language instructions for the agent on when/how to use the skill) plus optional TypeScript or shell scripts. Skills can be bundled, installed globally, or live in a workspace. Workspace skills win over global ones. The marketplace is **ClawHub**.
- **ACP (Agent Client Protocol):** A separate subsystem (`acpx`) for spawning durable Codex / Claude Code / Gemini CLI sessions. Distinguishes between:
  - *Chat surface* — where the human is talking (a Discord channel)
  - *ACP session* — the long-running Codex/Claude runtime
  - *Runtime workspace* — the filesystem location (cwd, repo checkout)
  - Supports commands like `/acp spawn codex --bind here --cwd /workspace/repo`
- **Inbox / Council / Ultraplan:** Cross-session messaging (virtual team layer), multi-agent "council" consensus voting, and plan-review tools.
- **Heartbeat:** A configurable periodic job that pings the agent to assess whether it should proactively act (with quiet hours + editable prompts).
- **Cron:** Timezone-aware scheduler for repeating or one-shot tasks.
- **Persistent context:** Config, memory, and interaction history are stored locally in `~/.openclaw/workspace`, persisting across sessions and surviving restarts. Injected prompt files: `AGENTS.md`, `SOUL.md`, `TOOLS.md`.

### Third-party architectural variant: `openclaw-claude-code` plugin
An OpenClaw plugin (by Enderfga) that turns Claude Code CLI into a programmable headless coding engine. Its internal structure shows the general pattern OpenClaw uses:
- `persistent-session.ts` — Claude engine (stateful subprocess)
- `persistent-codex-session.ts` / `persistent-gemini-session.ts` / `persistent-cursor-session.ts` — one-shot engines
- `session-manager.ts` — multi-session orchestration + council management
- `circuit-breaker.ts` — engine failure tracking with exponential backoff
- `inbox-manager.ts` — cross-session messaging
- Multi-model proxy that translates between Anthropic format and OpenAI/Gemini

---

## 5. Programming Languages, Frameworks & Tools Used to Build It

| Layer | Technology |
|---|---|
| **Primary language** | **TypeScript** (~357,000 LOC in the main repo) |
| **Runtime** | Node.js 24 recommended, 22.16+ supported |
| **Package manager** | pnpm (also published to npm as `openclaw`) |
| **Dev-mode execution** | `tsx` (run TS directly) |
| **Production build** | Packaged single binary via `pnpm build` → `dist/` |
| **Control UI framework** | Lit (web components), single-page app |
| **Transport** | WebSocket (port 18789) |
| **macOS/iOS native extensions** | Swift |
| **Windows companion suite** | Node + PowerToys Command Palette extension + system-tray app |
| **Packaging** | npm, Homebrew (implied), Docker, Nix |
| **License** | MIT |

### Ecosystem repositories under the `openclaw` GitHub org
- `openclaw/openclaw` — main repo (TypeScript, MIT)
- `openclaw/openclaw-windows-node` — Windows companion
- `openclaw/openclaw.ai` — website
- `openclaw/nix-openclaw` — Nix package
- `openclaw/lobster` — a typed, local-first macro engine / workflow shell that composes skills into pipelines

---

## 6. Features

### Messaging & interaction
- 25+ supported messaging channels (see list above)
- Works in DMs and group chats
- Voice support (ElevenLabs integration — users have configured custom voices like an "Aussie accent Jarvis")
- Live Canvas rendering (agent can produce interactive visuals you control)
- 50+ total integrations listed on the official site (Claude, GPT, Spotify, Hue, Obsidian, Twitter, Browser, Gmail, GitHub, etc.)

### Agent capabilities
- **Skills system** — ~5,700+ community skills on ClawHub (Feb 2026); grew from 2,857 in early Feb to 10,700+ by mid-Feb per Koi Security. Skills are plain-text `SKILL.md` files + optional scripts.
- **Self-extending** — users report asking OpenClaw to "build a new skill for itself" by giving it a YouTube video or notes; it will write and install the skill.
- **Heartbeat system** — proactive background check-ins with configurable intervals and quiet hours.
- **Cron jobs** — timezone-aware scheduling.
- **Memory persistence** — context survives across agents (Codex, Cursor, Manus, etc.) and across sessions/channels.
- **Model flexibility** — route different tasks to different models based on cost/capability.
- **Cross-channel coherence** — can connect unrelated conversations from different channels into coherent documents.

### Coding-specific features (via plugins like `openclaw-claude-code`)
- Manage Claude Code, Codex, Gemini, Cursor sessions
- Agent "council" consensus voting with explicit `[CONSENSUS: YES/NO]` tags
- Git-worktree-per-agent isolation
- Cross-session inbox delivery
- Multi-model proxy (serve Gemini/GPT via an Anthropic-compatible endpoint)
- 27 built-in tools in the plugin entry

### Deployment features
- One-line install script (`npm install -g openclaw`)
- Onboarding wizard (`openclaw onboard`)
- Daemon install (`--install-daemon`, uses launchd/systemd)
- Dev/beta/stable release channels via `openclaw update --channel ...`
- Web dashboard for management

### Pricing
- **Software:** Free and MIT-licensed.
- **Cost drivers (variable):** LLM API costs, hosting if you run it on a VPS (~$4–5/mo on Hetzner is commonly cited), optional voice services (ElevenLabs). Anthropic has reportedly banned the OpenClaw "harness" from running on Claude Pro/Max subscriptions, pushing users toward direct API billing or alternative models (Kimi, Codex, GLM).
- A bundled subscription is advertised that packages 5 flagship models (Opus 4.6, GLM-5.1, KIMI K2.5, GPT-5.2 Codex, Gemini 3.1 Pro).

---

## 7. Installation & Setup

### Quick start (from the official README)
```bash
npm install -g openclaw@latest
# or
pnpm add -g openclaw@latest

openclaw onboard --install-daemon
```

The onboarding wizard walks through: model selection, heartbeat configuration, Telegram setup, Discord setup, and security settings. After it finishes, the daemon is live and a web dashboard is available.

### Prerequisites
- Node.js 24 (recommended) or 22.16+
- pnpm or npm
- A running messaging account on whichever channels you plan to wire up (a Telegram bot via BotFather, a Discord bot via the Developer Portal, etc.)
- LLM API key(s)

### Configuration
- Workspace: `~/.openclaw/workspace` (configurable via `agents.defaults.workspace`)
- Minimal config: `~/.openclaw/openclaw.json`
- Skills: `~/.openclaw/workspace/skills/<skill>/SKILL.md`
- Secrets/tokens: stored locally (historically in plaintext in the `~/.clawdbot` or `~/.openclaw` directory — note: this is considered a security weakness, see §8)

### Alternative install paths
- **Docker** — official images exist (`openclaw/openclaw:<version>`)
- **Nix** — official `nix-openclaw` flake
- **Homebrew** — mentioned in documentation

---

## 8. Security Risks and Concerns (Extensive)

OpenClaw's security situation is serious and well-documented. This section is not optional reading for anyone considering running it.

### 8.1 Tracked CVEs (138 total as of April 2026)

**CVE-2026-25253 — One-click RCE via WebSocket hijacking (CVSS 8.8)** — the headline vulnerability
- *Classification:* CWE-669 (Incorrect Resource Transfer Between Spheres)
- *Affected:* all OpenClaw versions before **2026.1.29**
- *Root cause:* The Control UI's `applySettingsFromUrl()` function in `ui/src/ui/app-settings.ts` trusted a `gatewayUrl` query parameter without validation. On page load, it auto-opened a WebSocket and transmitted the stored auth token.
- *Exploit chain:* Victim visits a malicious webpage while OpenClaw is running → token exfiltrated in milliseconds → cross-site WebSocket hijacking (no origin validation on the server) → attacker connects to victim's local gateway → sets `exec.approvals.set = off` to disable sandbox → sets `tools.exec.host = gateway` to escape Docker → arbitrary shell command execution on the host.
- *Key insight:* **Running only on `localhost` did NOT protect you.** Browsers don't enforce same-origin policy on WebSocket connections the same way they do for HTTP, and the victim's own browser initiates the outbound leak.
- *Discovered by:* Mav Levin at DepthFirst; privately disclosed Jan 26, 2026; patched Jan 30; publicly disclosed Feb 3.
- *Exposure at disclosure:* SecurityScorecard counted 40,214 internet-exposed OpenClaw instances, 35.4% flagged vulnerable. Infosecurity Magazine reported 63% vulnerable with 12,812 instances directly exploitable for RCE. Censys: 21,639 exposed instances.

**Other notable CVEs:**
- **CVE-2026-24763** — Command injection
- **CVE-2026-26322** — Server-Side Request Forgery (SSRF). Particularly dangerous when chained with the RCE to hit cloud-metadata endpoints (AWS/GCP) and steal cloud credentials.
- **CVE-2026-26329** — Path traversal enabling local file reads
- **CVE-2026-30741** — Prompt-injection-driven code execution
- **CVE-2026-22708** — Indirect prompt injection via web browsing (OpenClaw doesn't sanitize web content before feeding it to the LLM context)
- **CVE-2026-27487** (CVSS 7.8) — macOS Keychain command injection via malicious skill names; affects all macOS versions before 3.0.8.

Additional issues reported:
- WebSocket origin validation missing
- Localhost trust bypass (when behind a reverse proxy on the same host, external traffic appeared as loopback)
- Guest-mode privilege escalation (missing `Authorization` headers dropped sessions to "Guest" but logic error in `SessionManager.js` left Python REPL tool access available)
- Kaspersky audit found 512 total issues, 8 critical
- Nine additional CVEs disclosed in a single four-day window (Feb 3–7, 2026)

### 8.2 ClawHub marketplace supply-chain problem

The skills registry has been called the more serious long-term problem than any single CVE.

- **Low publishing barrier:** For a period, anyone with a GitHub account as little as one week old could publish a skill. No code signing, no security review, no sandboxing.
- **Malicious skill counts (varying sources):**
  - Snyk "ToxicSkills" audit (scanned 3,984 skills): **1,467 malicious** (~37%)
  - ProArch/Conscia count: **800+ malicious skills** identified in the marketplace
  - Dev.to/TIAMAT report: 341 malicious in a January 2026 audit (12 credential harvesters, 23 remote access tools, 47 data exfiltration pipelines); 36.82% of scanned skills had at least one exploitable flaw
- **Cisco AI Security Research** tested a third-party OpenClaw skill and found it performed data exfiltration and prompt injection without user awareness.
- **Remediation (Mar 2026):** OpenClaw partnered with VirusTotal for automated skill scanning and ClawHub implemented mandatory code review for new submissions.

### 8.3 Moltbook backend breach
A separate incident: the Moltbook backend (the AI-agent social network that launched alongside the rebrand) leaked **1.5 million API tokens** plus 35,000 user email addresses via a single Redis misconfiguration on an unauthenticated endpoint. Called "the largest security incident in sovereign AI history" by the researcher who discovered it.

### 8.4 Structural / architectural security concerns
These are not bugs — they are consequences of the design:

- **Excessive agency:** An OpenClaw agent typically has full shell, filesystem, and browser access plus tokens for Gmail, Calendar, GitHub, and so on. A compromise of the agent equals compromise of the user's entire digital life.
- **"Lethal trifecta":** Any agent with (private data access) + (untrusted content exposure) + (external communication) is high risk by definition. OpenClaw is designed to have all three.
- **Prompt injection surface is the whole internet:** Because the agent processes web pages, emails, messages, and skill outputs, any of those can carry embedded instructions that the LLM may execute. OpenClaw does not enforce a semantic firewall.
- **No identity layer on tools:** Agents decide what to execute based on instructions that can be manipulated.
- **No memory integrity:** Past context can be silently poisoned across sessions.
- **Plaintext secret storage:** Tokens stored in `~/.clawdbot` / `~/.openclaw` have been flagged as a probable future infostealer target, similar to `~/.npmrc` and `~/.gitconfig`.
- **Sandbox/guardrail bypass via API:** The agent's own safety features (approval prompts, sandbox) are themselves exposed as API endpoints, meaning any API compromise disables them. As Levin put it: these defenses were designed to contain a malicious LLM, not a malicious operator.
- **Privacy:** Because everything runs on your hardware, OpenClaw's privacy story is good *relative to cloud SaaS* — but only if you trust every skill and every LLM call you make.

### 8.5 Maintainer warnings
One of OpenClaw's own maintainers, "Shadow," stated on Discord:
> "If you can't understand how to run a command line, this is far too dangerous of a project for you to use safely."

### 8.6 Hardening checklist (recommended by security researchers)
1. **Update to v2026.1.29+** (mandatory). Check your version.
2. **Bind to `127.0.0.1` only.** Verify: `lsof -iTCP:18789 -sTCP:LISTEN` — you want to see `127.0.0.1:18789`, not `0.0.0.0:18789`.
3. **Enable authentication.** Many exposed instances run with none.
4. **Use a dedicated browser profile** for the Control UI so drive-by WebSocket attacks from unrelated browsing can't reach the gateway.
5. **Rotate all tokens** if you ever ran a pre-patch version while browsing.
6. **Audit installed skills.** Remove anything unrecognized. Prefer official/reviewed skills.
7. **Use least-privilege.** Don't grant full shell access unless you actually need it.
8. **Treat the agent as privileged infrastructure** — apply the same governance you'd apply to a production server.
9. **Monitor logs** for unauthorized config changes (especially `exec.approvals.*` and `tools.exec.host`).
10. **Map against OWASP Top 10 for Agentic Applications** and consider red-teaming tools (Penligent, Crawdad skill, etc.).

---

## 9. Variants, Forks, and Alternatives

### Related / derivative projects
- **ClaudeClaw** (`moazbuilds/claudeclaw`) — a lightweight open-source "OpenClaw version built into your Claude Code." Runs as a background daemon, supports Telegram/Discord, voice command transcription, heartbeat + cron, web dashboard.
- **openclaw-claude-code** (`Enderfga/openclaw-claude-code`) — an OpenClaw *plugin* that turns Claude Code CLI into a programmable headless coding engine; supports Claude, Codex, Gemini, and Cursor via a unified interface.
- **Lobster** (`openclaw/lobster`) — official typed, local-first "macro engine" that composes skills into pipelines.
- **Forks into other languages:** Community has produced Rust, Go, Python, and Shell ports.
- **ClawCode** (`iamAliAgha/ClawCode`) — *unrelated* third-party coding assistant that sometimes gets confused with OpenClaw. No affiliation.

### Competitors / alternatives
- **Claude Code Channels** — Anthropic's official response, announced in early 2026. Works via the Model Context Protocol (MCP) with Discord, Telegram, iMessage, and custom webhooks. Backed by Anthropic's SOC 2 Type 2 compliance, read-only defaults, allow-lists, and reviewed plugins. Considered more secure but less flexible; head-to-head reviews scored ~31 vs 30 between the two.
- **Perplexity Computer** — multi-model orchestration with pre-built app integrations.
- **Devin, Cursor Agent, LangChain, CrewAI, AutoGen** — adjacent but generally not messaging-app-first.

### How OpenClaw compares
- **vs Claude Code (stock):** Claude Code is a purpose-built coding agent; OpenClaw is a general-purpose life automation layer. OpenClaw doesn't natively understand your codebase. Use Claude Code for software engineering; use OpenClaw (optionally wrapping Claude Code) for messaging-driven automation and 24/7 availability.
- **vs Claude Code Channels:** Channels wins on security and ease of setup; OpenClaw wins on always-on deployment, model flexibility, and extensibility. Channels is basically free if you already have a Claude subscription; OpenClaw increasingly pushes you toward paid APIs.

---

## 10. Community, Ecosystem & Documentation

- **Main GitHub:** `github.com/openclaw/openclaw` — ~199K–247K stars (sources vary; Wikipedia says 247K/47.7K forks as of Mar 2, 2026), 11,440+ commits, 900+ contributors, MIT license.
- **Website:** `openclaw.ai` and `open-claw.org`
- **Docs:** `docs.openclaw.ai`
- **Showcase / FAQ / Getting Started / Onboarding / Nix / Docker** all linked from the main repo.
- **Discord community:** active; where most skill development and troubleshooting happens.
- **DeepWiki:** auto-generated doc site also linked from the README.
- **YouTube:** numerous tutorials and demos (the project's virality was partly driven by video demos).
- **Andrej Karpathy** called it "the most incredible sci-fi takeoff-adjacent thing."
- **Notable backers / promoters:** Lex Fridman (podcast interview with Steinberger), Matt Schlicht (launched the companion Moltbook platform).
- **Coverage:** VentureBeat, The Hacker News, SonicWall, SecurityWeek, Infosecurity Magazine, Fortune, runZero, Conscia, Snyk, Kaspersky, Cisco, Adversa.ai, Sangfor, University of Toronto Infosec advisories.

---

## 11. Recent Developments & Current Status (as of April 2026)

- **Steinberger at OpenAI:** Announced Feb 14, 2026; leading personal-agents work alongside Sam Altman.
- **Foundation:** Independent non-profit now stewards OpenClaw.
- **Anthropic's pushback:** Banned the OpenClaw "harness" from running on Claude Pro/Max subscriptions, forcing users toward paid API access or alternative models (Kimi K2.5, GPT-5.2 Codex, GLM-5.1). This materially changed the cost profile for many users.
- **Anthropic's competitive response:** Launched **Claude Code Channels** with similar Telegram/Discord/iMessage functionality but delivered inside their own policy and safety perimeter.
- **Chinese ecosystem:** OpenClaw was adapted to work with DeepSeek and domestic super-apps like WeChat; Tencent and Z.ai announced OpenClaw-based services.
- **Controversies:**
  - *MoltMatch incident:* A user's OpenClaw agent autonomously created a dating-profile on the experimental MoltMatch platform and began screening matches without his explicit direction, raising consent questions. AFP reporting found at least one MoltMatch profile using a Malaysian model's photos without consent.
  - *Malicious skills:* Covered above.
  - *Massive internet exposure:* Covered above.
- **Remediation efforts:** VirusTotal skill scanning, mandatory ClawHub code review, hardened WebSocket origin validation, authentication-by-default in the onboarding wizard.

---

## 12. Bottom Line

OpenClaw is a genuinely novel piece of software: the first mainstream demonstration that an autonomous, persistent AI agent accessed through everyday messaging apps is not only possible but useful enough to acquire hundreds of thousands of stars in months. It's arguably the most influential open-source agent project shipped so far.

It is also **built in a "ship fast, read later" style by a single founder** — with all the security implications that entails. The 138 CVEs, 40,000+ exposed instances, ClawHub supply-chain crisis, and Moltbook token leak are not minor footnotes; they're a structural consequence of giving a messaging-driven LLM full shell access on real users' machines with no identity layer, no action-authorization layer, and a crowd-sourced plugin registry.

If you decide to run it: patch to **v2026.1.29 or later**, bind to localhost only, enable authentication, use a dedicated browser profile for the Control UI, minimize granted permissions, audit every skill you install, rotate tokens after any upgrade from a vulnerable version, and treat the host as privileged infrastructure.

---

## 13. Source List

- OpenClaw official site — https://openclaw.ai / https://open-claw.org
- OpenClaw docs — https://docs.openclaw.ai
- OpenClaw GitHub — https://github.com/openclaw/openclaw
- OpenClaw blog "Introducing OpenClaw" (Steinberger, Jan 29, 2026) — https://openclaw.ai/blog/introducing-openclaw
- Wikipedia: OpenClaw — https://en.wikipedia.org/wiki/OpenClaw
- Enderfga/openclaw-claude-code plugin — https://github.com/Enderfga/openclaw-claude-code
- moazbuilds/claudeclaw — https://github.com/moazbuilds/claudeclaw
- VentureBeat: "Anthropic just shipped an OpenClaw killer called Claude Code Channels"
- aimaker.substack.com: "Did Anthropic Just Kill OpenClaw with Claude Code Channels?"
- claudefa.st: "OpenClaw vs Claude Code: Which Should You Use?"
- verdent.ai: "Claw Code: Claude Code, OpenClaw, and What Each Actually Does"
- NVD: CVE-2026-25253
- The Hacker News: "OpenClaw Bug Enables One-Click Remote Code Execution via Malicious Link"
- SonicWall: "OpenClaw Auth Token Theft Leading to RCE: CVE-2026-25253"
- ProArch: "OpenClaw RCE Vulnerability (CVE-2026-25253)"
- Panstag: "OpenClaw Security Risks: How to Stay Safe"
- Adversa.ai: "OpenClaw security 101: vulnerabilities, hardening 2026"
- Sangfor: "OpenClaw Security Risks: From Vulnerabilities to Supply Chain Abuse"
- Dev.to (TIAMAT/ENERGENAI): "OpenClaw Security Catastrophe"
- Penligent.ai: "CVE-2026-25253 OpenClaw Bug Enables One-Click Remote Code Execution"
- University of Toronto Information Security advisory on OpenClaw
- runZero exposure analysis
- openclaw.report: "OpenClaw Just Passed Linux on GitHub"
- allclaw.org: "Peter Steinberger Biography 2026"
- Fortune (Feb 19, 2026) coverage of Steinberger and OpenAI
- Snyk ToxicSkills audit; Koi Security ClawHub tracking; Kaspersky codebase audit; Cisco AI Security Research skill testing