# Hermes Agent: A Comprehensive Overview

> Source research doc — canonical copy for the meta-workspace. Second copy lives at `Tampertantrum/mothership/research/hermes-agent.md` for project-local use.

## What Is Hermes Agent?

Hermes Agent is an open-source, self-improving AI agent built by Nous Research — the research lab behind the Hermes, Nomos, and Psyche model families. Unlike coding copilots tethered to an IDE or chatbot wrappers around a single API, Hermes is a fully autonomous agent designed to get more capable the longer it runs. It operates under the MIT license and has attracted significant community interest, with over 8,700 GitHub stars and 142 contributors as of early 2026.

The project's tagline — "the agent that grows with you" — reflects its central innovation: a closed learning loop where the agent creates skills from experience, improves them during use, nudges itself to persist knowledge, searches its own past conversations, and builds a deepening model of who you are across sessions.

Hermes is model-agnostic. It works with Nous Portal, OpenRouter (200+ models), OpenAI, Anthropic, z.ai/GLM, Kimi/Moonshot, MiniMax, or any custom endpoint. Switching providers requires a single command (`hermes model`) — no code changes.

## Core Architecture

Hermes Agent is written primarily in Python (91.4% of the codebase) and built around a central orchestration class called `AIAgent` in `run_agent.py`, which is roughly 10,700 lines. This single class serves every entry point — CLI, messaging gateway, API server, ACP (editor integration), and batch processing — keeping platform-specific differences in the entry points rather than the core.

### The Agent Loop

The heart of Hermes is its synchronous agent loop. When a user sends a message, the flow proceeds as follows:

1. **Prompt Assembly** — The `prompt_builder` constructs a system prompt from the agent's personality (SOUL.md), persistent memory (MEMORY.md, USER.md), active skills, context files (AGENTS.md, .hermes.md), tool-use guidance, and model-specific instructions.
2. **Provider Resolution** — A runtime resolver maps the configured `(provider, model)` tuple to the correct `(api_mode, api_key, base_url)`. Hermes supports three API modes: OpenAI-style chat completions, Codex-style responses, and native Anthropic messages.
3. **LLM Inference** — The assembled prompt and conversation history are sent to the chosen model.
4. **Tool Dispatch** — If the model returns tool calls, they are routed through a central tool registry (`tools/registry.py`) for execution, and the results are fed back for another inference pass. This loop continues until the model produces a final text response.
5. **Persistence** — The completed conversation turn is saved to a SQLite session database with FTS5 full-text search indexing.

A key design principle is prompt stability: the system prompt is frozen at session start and never changes mid-conversation, preserving the LLM's prefix cache for performance.

### Entry Points

Hermes provides multiple ways to interact with the same underlying agent:

- **CLI (`cli.py`)** — A full terminal user interface with multiline editing, slash-command autocomplete, conversation history, interrupt-and-redirect, and streaming tool output.
- **Messaging Gateway (`gateway/run.py`)** — A long-running process with 18 platform adapters (Telegram, Discord, Slack, WhatsApp, Signal, Matrix, Mattermost, Email, SMS, and more). Users can talk to the agent from their phone while it works on a remote VM.
- **ACP Adapter** — Exposes Hermes as an editor-native agent via stdio/JSON-RPC for VS Code, Zed, and JetBrains.
- **Batch Runner** — Generates trajectories at scale for evaluation and RL training.

## The Learning Loop

What distinguishes Hermes from other agent frameworks is that its memory, skills, and user modeling systems form a feedback loop that makes it more useful over time.

### Persistent Memory

Hermes has bounded, curated memory that persists across sessions through two files:

- **MEMORY.md** (~800 tokens) — The agent's personal notes: environment facts, discovered conventions, lessons learned, and completed-task diary entries.
- **USER.md** (~500 tokens) — A user profile covering preferences, communication style, technical skill level, and workflow habits.

Both files are injected into the system prompt as a frozen snapshot at session start. The agent manages its own memory via a dedicated `memory` tool — it can add, replace, or remove entries. When memory reaches capacity, the agent consolidates or replaces entries to make room, keeping the most useful information.

The agent saves memory proactively: when it learns user preferences, discovers environment details, receives corrections, or completes significant work, it persists that knowledge without being asked. Trivial or easily re-discoverable facts are skipped.

### Skills System (Procedural Memory)

Skills are on-demand knowledge documents that function as the agent's procedural memory. After completing a complex task (5+ tool calls), hitting errors and finding the working path, or receiving user corrections, the agent autonomously creates a skill capturing the successful approach for future reuse.

Skills follow a progressive disclosure pattern to minimize token usage:

- **Level 0:** A lightweight index listing all skill names, descriptions, and categories (~3k tokens total).
- **Level 1:** The full content of a specific skill, loaded only when needed.
- **Level 2:** A specific reference file within a skill directory.

Each skill is defined by a `SKILL.md` file using a standardized YAML frontmatter format, and can include reference documents, templates, helper scripts, and supplementary assets. Skills can also self-improve during use — when the agent discovers a better approach, it patches the existing skill using a targeted `old_string` → `new_string` replacement.

Hermes integrates with a broader Skills Hub ecosystem, supporting installation from multiple sources: official optional skills bundled with Hermes, the skills.sh directory (Vercel's public skills registry), well-known skill endpoints on websites, direct GitHub repositories, and community marketplaces. All hub-installed skills undergo an automated security scan before installation.

### Cross-Session Recall

Beyond the compact memory files, Hermes maintains a SQLite database with FTS5 full-text search across all past sessions. The agent can search its own conversation history and use an auxiliary LLM to summarize relevant results, enabling cross-session recall of detailed information that doesn't fit in the bounded memory stores.

### User Modeling

Hermes optionally integrates with Honcho (by Plastic Labs) for dialectic user modeling — building a deeper, evolving understanding of the user that goes beyond the static USER.md profile.

## Tools and Capabilities

Hermes ships with 47 registered tools organized into 19 toolsets. These cover a wide range of capabilities:

- **Terminal execution** — Run shell commands across 6 backends: local, Docker, SSH, Daytona, Singularity, and Modal. The Daytona and Modal backends offer serverless persistence, where the agent's environment hibernates when idle and wakes on demand.
- **File operations** — Read, write, patch, and search files.
- **Web interaction** — Search, extract content, browse with automation (5 browser backends), and use vision capabilities.
- **Code execution** — A sandboxed `execute_code` tool for running Python scripts that can call other tools via RPC, collapsing multi-step pipelines into zero-context-cost turns.
- **Delegation** — Spawn isolated subagents for parallel workstreams.
- **MCP integration** — Connect to any MCP (Model Context Protocol) server for extended capabilities, with tool filtering for safety.
- **Media** — Image generation, text-to-speech, voice mode (real-time voice interaction in CLI and messaging platforms).

Tool registration happens at import time via a central registry — any `tools/*.py` file with a top-level `registry.register()` call is auto-discovered without needing a manual import list.

## Deployment Model

Hermes is designed to run independently of the user's laptop. Common deployment patterns include:

- **$5 VPS** — A cheap cloud server running Hermes 24/7, accessible via Telegram or Discord.
- **GPU cluster** — For research workloads, RL training, or batch trajectory generation.
- **Serverless** — Daytona and Modal backends allow the agent's environment to hibernate when idle and wake on demand, costing nearly nothing between sessions.
- **Local** — Standard local installation on Linux, macOS, or WSL2.

The messaging gateway enables talking to the agent from any platform while it works on a remote machine. Voice memo transcription and cross-platform conversation continuity are built in.

## Scheduled Automations (Cron)

Hermes includes a first-class cron scheduler for unattended agent tasks. Unlike traditional shell-based cron, these are agent tasks — the scheduler creates a fresh AIAgent instance, injects any attached skills as context, runs the job prompt, and delivers the response to the target platform.

Jobs are defined in natural language and support delivery to any connected messaging platform. Examples include daily reports, nightly backups, and weekly audits.

## Research and Training

Being built by a model training lab, Hermes has research-oriented features baked in:

- **Batch trajectory generation** — Run the agent against many prompts to produce training data at scale.
- **Atropos RL environments** — A full environment framework for evaluation and reinforcement learning, integrated via the `tinker-atropos` submodule.
- **Trajectory compression** — Convert agent interaction logs into ShareGPT-format trajectories for training the next generation of tool-calling models.
- **Multiple tool-call parsers** — Support for different formats used by various model families.

## Security Model

Hermes includes several security layers:

- **Command approval** — Dangerous terminal commands require explicit user approval before execution.
- **DM pairing** — Messaging platforms use an allowlist and DM pairing flow to authorize users.
- **Container isolation** — Terminal backends like Docker, Daytona, and Modal provide sandboxed execution environments.
- **Skills security scanning** — Hub-installed skills undergo automated scanning for data exfiltration, prompt injection, destructive commands, and supply-chain threats.
- **Trust levels** — Skills are classified as builtin, official, trusted, or community, with escalating scrutiny.

## Getting Started

Installation is a single command:

```
curl -fsSL https://raw.githubusercontent.com/NousResearch/hermes-agent/main/scripts/install.sh | bash
```

This works on Linux, macOS, and WSL2, and handles all prerequisites (Python, Node.js, dependencies, and the `hermes` CLI command). After installation:

```
source ~/.bashrc
hermes              # start chatting
hermes model        # choose an LLM provider and model
hermes tools        # configure which tools are enabled
hermes gateway      # start the messaging gateway
```

Full documentation is available at hermes-agent.nousresearch.com/docs.

## Summary

Hermes Agent represents a distinct approach to AI agents: rather than being a stateless tool that starts fresh every session, it maintains and curates its own knowledge over time. Its combination of persistent memory, autonomous skill creation, cross-session search, user modeling, multi-platform presence, and flexible deployment makes it function more like a long-running collaborator than a one-shot assistant. Built by a research lab with model training in mind, it also serves as infrastructure for generating the training data needed to improve the next generation of tool-calling models.

## Quick Facts

| Aspect | Details |
|--------|---------|
| Developer | Nous Research |
| License | MIT |
| Language | Python (91.4%) |
| Stars | 8,700+ |
| Contributors | 142 |
| Tools | 47 built-in, across 19 toolsets |
| Platforms | 18 messaging adapters + CLI + ACP |
| Terminal Backends | 6 (local, Docker, SSH, Daytona, Modal, Singularity) |
| Latest Release | v0.3.0 (March 17, 2026) |
