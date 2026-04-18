**Claude Code & Claude Cowork**

*A Research Briefing on Memory, Sub-Agents, Skills, Commands, Hooks, Rules & Best Practices*

Sourced primarily from Anthropic's official documentation and engineering blog · April 2026

# **1\. Executive Summary**

Claude Code and Claude Cowork are Anthropic's two flagship agentic products. They share the same underlying architecture (an agentic loop, file system access, tool use, and sub-agent orchestration) but target different audiences and surfaces.

**Claude Code** is a terminal-native agentic coding tool for developers. It lives in the CLI, IDE extensions (VS Code, JetBrains), a web app, and a desktop app, and is designed to read codebases, write files, run commands, and work autonomously on engineering tasks.

**Claude Cowork** is Anthropic's desktop-native agent for non-technical knowledge work, launched January 2026 as a research preview and made generally available in April 2026 on all paid plans for macOS and Windows. It wraps the same agentic architecture behind Claude Code in the Claude Desktop app so marketers, analysts, legal, finance, and operations teams can delegate file-heavy, multi-step tasks without touching a terminal.

Both products share a common extensibility model — CLAUDE.md memory, Skills, sub-agents, hooks, commands, MCP connectors, and plugins — which is the focus of this briefing.

## **The Extension Stack at a Glance**

| Feature | What it is | Primary use |
| :---- | :---- | :---- |
| CLAUDE.md | Always-on project context | Persistent conventions, tech stack, workflow rules every session should know. |
| Skills | On-demand knowledge \+ workflows | Reusable procedures, style guides, deployment steps, domain expertise. Auto-invoked by description. |
| Sub-agents | Isolated agent instance | Context isolation, parallel work, restricted tool access. Runs in its own window, returns a summary. |
| Hooks | Deterministic event handlers | Guaranteed execution on lifecycle events (PreToolUse, PostToolUse, etc.). Auto-format, block risky commands, run tests. |
| Commands | Saved prompts as slash commands | User-invoked shortcuts. Largely superseded by Skills, but still supported. |
| MCP servers | External service connectors | Connect to GitHub, Slack, databases, APIs. MCP handles auth and API calls. |
| Plugins | Packaging layer | Bundle skills, hooks, sub-agents, and MCP servers into one installable unit. |

 

# **2\. Memory: How Claude Remembers (and Doesn't)**

Anthropic's memory documentation is explicit on a point that matters: every Claude Code session begins with a fresh context window. There is no hidden long-term memory. Two mechanisms carry knowledge across sessions — CLAUDE.md and auto memory — and both are treated as context, not as enforced configuration. Memory influences behavior; it does not act as a policy engine.

## **2.1 The CLAUDE.md Hierarchy**

Claude Code loads CLAUDE.md files in a cascading hierarchy. Higher-precedence files load first and can be overridden or extended by lower-precedence files. All are auto-loaded at session launch.

| Type | Location | Scope | Shared with |
| :---- | :---- | :---- | :---- |
| Enterprise policy | /etc/claude-code/CLAUDE.md (Linux) | Organization-wide | All users |
| Project memory | ./CLAUDE.md or ./.claude/CLAUDE.md | Team-shared project | Team via source control |
| Project rules | ./.claude/rules/\*.md | Modular project instructions | Team via source control |
| User memory | \~/.claude/CLAUDE.md | Personal (all projects) | Just you |
| Project local | ./CLAUDE.local.md | Personal project-specific | Just you (auto-gitignored) |

 

Project-scoped rules in ./.claude/rules/ are discovered recursively and can use YAML frontmatter to restrict which files they apply to, for example limiting an API-style rule to src/api/\*\*/\*.ts. Claude Code also supports @path/to/file imports inside CLAUDE.md so you can pull in README content or package.json references on demand.

## **2.2 Auto Memory, Checkpoints, and Compaction**

* **Auto memory** — a newer mechanism that lets Claude maintain learned patterns across sessions. Sub-agents can also maintain their own auto memory (stored under \~/.claude/agent-memory/ when User scope is selected during agent creation). Anthropic documents loading limits and explicitly notes that both CLAUDE.md and auto memory are treated as context, not enforced configuration.

* **Checkpoints** — separate from compaction. Claude Code snapshots file state before each edit so you can rewind code, conversation, or both. A documented limitation: file modifications made via Bash (rm, mv, cp) are not tracked by checkpoint rewind, which matters in sensitive repositories.

* **Compaction** — when the context window fills, Claude Code clears older tool outputs first, then summarizes the conversation. Invoked skills are re-attached after a summary (first 5,000 tokens each, combined budget of 25,000 tokens across re-attached skills, filled from the most recently invoked). Many "Claude drifted" complaints are actually context-budget complaints.

## **2.3 Best Practices for CLAUDE.md**

The strongest signal from Anthropic's published guidance, and from teams like HumanLayer who have stress-tested this in production, is that CLAUDE.md should be lean. The Claude Code system prompt already contains roughly 50 individual instructions — nearly a third of what any model can reliably follow — before your rules, plugins, skills, or user messages are added. As instruction count rises, instruction-following quality falls uniformly: the model doesn't just ignore the last thing added, it begins ignoring all of them equally.

* Keep it short — general consensus is under 300 lines; HumanLayer's own root file is under 60\.

* Tell Claude WHAT (tech stack, project structure, where things live) and WORKFLOW (build, test, lint commands, branching conventions).

* Don't stuff in task-specific instructions (how to structure a new DB schema, how to fix yesterday's bug). Those belong in task prompts or scoped rules, not global memory.

* Use @imports to pull in detail only when needed (e.g., @docs/architecture.md) rather than inlining everything.

* Run /init once to generate a starter CLAUDE.md from codebase analysis, then prune.

* Reframe directive compliance — instead of "ALWAYS run tests", explain why: "The user depends on this check because skipping compilation verification causes them to discover type errors later, disrupting their workflow." Models follow reasoning better than imperatives.

* Version-control CLAUDE.md. Treat it as living project documentation, not a junk drawer for prompt hotfixes.

## **2.4 Memory in Claude Cowork**

Cowork's memory model is more constrained than Claude Code's. Standalone Cowork tasks do not retain memory across sessions. Memory is only supported inside Cowork Projects, which were introduced in March 2026 as persistent, self-contained workspaces with their own files, links, instructions, and scoped memory.

When a task runs inside a Cowork Project, Claude reads four layers in order: context files in the designated folder, global Cowork instructions, project-specific instructions, and project memory that updates roughly every 24 hours based on conversations in that project. Conversation history is stored locally on your computer, so it's not subject to Anthropic's standard retention timeframe.

# **3\. Sub-Agents**

A sub-agent is not just a named prompt template. Anthropic's definition has two operative points: it is a specialized AI assistant that runs in its own isolated context window with its own system prompt, tool access, and permissions; and when a task matches its description, the lead agent delegates to it, receives the final summary verbatim, and continues without every intermediate step polluting the main context.

## **3.1 Why Sub-Agents Matter**

* Context preservation — exploration, research, and log-reading stay in the sub-agent's window. Only the summary returns to the main conversation.

* Parallelism — multiple sub-agents can run simultaneously (research auth \+ database \+ API modules in parallel, then synthesize).

* Constraint enforcement — a doc-reviewer can be limited to Read and Grep, guaranteeing it can't modify files even if asked.

* Specialization — a database-migration sub-agent can carry detailed SQL best practices, rollback strategies, and integrity checks that would be noise in the main agent's system prompt.

* Cost control — route cheap, verbose tasks (log scanning, doc searching) to Haiku while keeping the main agent on Sonnet or Opus.

## **3.2 Built-in Sub-Agents**

Claude Code ships with several sub-agents that Claude auto-invokes when appropriate:

* **Explore** — fast, read-only (Haiku). Denied Write and Edit. Used for file discovery, code search, codebase exploration. Specifies a thoroughness level (quick, medium, very thorough).

* **Plan** — research agent used during plan mode to gather context before presenting an implementation strategy.

* **general-purpose** — available any time the Agent tool is in allowed tools. Handles complex multi-step tasks without a specialized definition.

## **3.3 Defining Custom Sub-Agents**

Sub-agents are markdown files with YAML frontmatter, stored at two scopes: .claude/agents/ (project, shared via git) and \~/.claude/agents/ (user, across all projects). In the Agent SDK, programmatically-defined agents take precedence over filesystem agents with the same name.

Example — a data-scientist sub-agent:

\---

name: data-scientist

description: Data analysis expert for SQL queries, BigQuery operations, and data insights. Use proactively for data analysis tasks and queries.

tools: Bash, Read, Write

model: sonnet

\---

You are a data scientist specializing in SQL and BigQuery analysis.

 

When invoked:

1\. Understand the data analysis requirement

2\. Write efficient SQL queries with proper filters

3\. Use BigQuery command line tools (bq) when appropriate

4\. Analyze and summarize results

5\. Present findings clearly

## **3.4 Sub-Agent Best Practices**

* Write a sharp description. Claude decides whether to delegate based on the description alone. Include trigger contexts ("Use PROACTIVELY when...", "MUST BE USED for...") to bias toward invocation.

* Treat the sub-agent's context as blank. The only channel from parent to child is the Agent tool's prompt string. Pass every file path, error message, constraint, and quality standard directly in that prompt.

* Restrict tools aggressively. A research sub-agent rarely needs Write or Edit. Narrowing tools is the cheapest safety control available.

* Use PreToolUse hooks to auto-approve sub-agent tools. Sub-agents don't inherit parent permissions automatically, which produces repeated prompts unless configured.

* Watch for infinite loops. A UserPromptSubmit hook that spawns sub-agents can recursively trigger itself. Check for a sub-agent indicator before spawning.

* Don't over-specialize too early. Start with the built-in general-purpose sub-agent for delegation; define a custom one when you find yourself spawning the same configuration repeatedly.

* Launch sub-agents in parallel early, then do setup work. Sub-agents take several seconds to initialize — if you're going to use them, kick them off first.

# **4\. Skills (Agent Skills)**

Skills are, in Anthropic's framing, "organized folders of instructions, scripts, and resources that agents can discover and load dynamically." Each skill is a directory with a SKILL.md file (YAML frontmatter \+ markdown body), plus any number of supporting files. Skills work across Claude Code, Cowork, Claude.ai, and the API — the same SKILL.md runs everywhere that supports the Agent Skills standard.

## **4.1 The Progressive Disclosure Architecture**

Skills are designed so that context is loaded only when needed. This is the single most important thing to understand about them:

* At startup, only the name and description from every skill's YAML frontmatter are pre-loaded into the system prompt.

* Claude reads SKILL.md only when the skill becomes relevant — selected automatically by description match, or explicitly via /skill-name.

* Additional files bundled with the skill (reference.md, forms.md, data files) are read on demand, only if Claude navigates to them.

* Scripts bundled with the skill can be executed via Bash without ever loading their contents into context. Only the script's output consumes tokens.

* There is effectively no context penalty for large bundled content that isn't read during a task.

## **4.2 Skill Anatomy**

A minimum viable skill:

\---

name: explain-code

description: Explains code with visual diagrams and analogies. Use when explaining how code works, teaching about a codebase, or when the user asks "how does this work?"

allowed-tools: Read, Grep, Glob

\---

When explaining code, always include:

1\. Start with an analogy from everyday life

2\. Draw a diagram using ASCII art showing flow/structure

3\. Walk through the code step-by-step

4\. Highlight a common gotcha or misconception

Frontmatter fields (all optional except description, which is strongly recommended):

| Field | Purpose |
| :---- | :---- |
| name | Becomes the slash command. Max 64 chars, lowercase/numbers/hyphens only. Cannot contain 'anthropic' or 'claude'. |
| description | Critical. Max 1024 chars. Must describe what the skill does AND when Claude should use it. This is what Claude uses to select the skill. |
| allowed-tools | Tools Claude can use when this skill is active, without per-use approval. |
| disable-model-invocation | Set true to require manual invocation via /skill-name. Claude won't auto-trigger it. |
| context | Set to 'fork' to run the skill in a sub-agent with isolated context. |
| agent | Which sub-agent to run in (Explore, Plan, general-purpose, or a custom one). Used with context: fork. |

 

## **4.3 Skill Authoring Best Practices (from Anthropic)**

* The description is the single most important design decision. Claude decides whether to load the skill based on name \+ description alone. Include specific trigger phrases and contexts. Pattern: "Use when the user provides X, even if they don't explicitly say Y."

* Claude tends to undertrigger. Counter this by being explicit about activation scenarios in the description rather than writing a dry summary.

* Keep SKILL.md under 500 lines. If you approach the limit, move detail into reference files and point to them with "Read references/agent-prompt.md for the full requirements." That's a Read-tool instruction, not an @import — @imports only work in CLAUDE.md, not SKILL.md.

* Default to assuming Claude is already smart. Only add context Claude doesn't already have. Don't re-explain general programming or writing principles.

* Reason, don't command. If you find yourself writing ALWAYS or NEVER in all caps, reframe with the reason. "Stay on the hub during build — the user watches thumbnails appear progressively, and navigating elsewhere breaks the visual feedback loop" outperforms "NEVER navigate during build."

* Use gerund form for names (verb \+ \-ing) when describing activities — clearer than nouns.

* Build a validation loop when quality matters: "Run validator → fix errors → repeat." For non-code skills, the validator can be a STYLE\_GUIDE.md the skill instructs Claude to check against.

* Test with all the models you target. Haiku may need more explicit detail; Opus may over-interpret the same instructions. If you're shipping to multiple models, aim for language that works for all of them.

* Iterate from observation, not assumption. Watch how Claude actually uses the skill. Does it read files in an unexpected order? Skip bundled content you thought was important? Revisit structure based on real usage.

* Chunk rather than consolidate. Three separate skills (voice, corporate writing, newsletter writing) outperform one giant writing skill — Anthropic and power users report this consistently.

* Only install skills from trusted sources. Skills can include scripts that execute with your permissions. Audit SKILL.md, all bundled scripts, and dependencies before enabling a skill from an unknown author.

## **4.4 Skill Storage Locations**

| Scope | Location | Notes |
| :---- | :---- | :---- |
| Project | .claude/skills/ | Shared with team via git |
| User | \~/.claude/skills/ | Personal, across all projects |
| Plugin | Bundled in installed plugins | Namespaced (/plugin-name:skill) |
| Managed | Provisioned by org admin | Highest precedence (Team/Enterprise) |

 

# **5\. Slash Commands**

Slash commands are typed shortcuts that execute predefined prompts. Claude Code ships with many built-ins (/help, /init, /compact, /clear, /permissions, /hooks, /model, /plugin, /schedule, /loop) and supports custom commands as markdown files.

As of early 2026, Anthropic's documentation marks custom commands under .claude/commands/ as legacy and recommends Skills instead (.claude/skills/). The functional distinction: commands are simple user-invoked prompt templates; skills can auto-trigger on context match, include bundled resources, and execute scripts.

## **5.1 Creating a Custom Command**

Project-scoped commands live in .claude/commands/ and user commands in \~/.claude/commands/. Create any .md file there and its name becomes /filename.

\---

allowed-tools: Read, Grep, Glob, Bash(git diff:\*)

argument-hint: \[issue-number\] \[priority\]

description: Fix a GitHub issue

model: claude-opus-4-7

\---

\#\# Changed Files

\!\`git diff \--name-only HEAD\~1\`

 

\#\# Task

Fix issue \#$1 with priority $2.

Check the issue description and implement the necessary changes.

Run tests before committing.

* $ARGUMENTS captures everything after the command name.

* $1, $2, ... capture positional arguments.

* \!\`command\` runs a shell command at expansion time and includes its output.

* YAML frontmatter supports allowed-tools, description, argument-hint, model, context, agent, and inline hooks.

* Built-in commands like /compact and /init are not available through the Skill tool.

* /help shows all available commands, including custom ones and MCP-exposed prompts.

# **6\. Hooks**

Hooks are the deterministic layer of Claude Code. They run shell commands, HTTP calls, prompts, or sub-agents in response to specific lifecycle events — regardless of what the model "decides" to do. Where CLAUDE.md rules are advisory, hooks are enforced. This is where Claude Code stops being helpful and starts being reliable.

## **6.1 Lifecycle Events**

Anthropic documents hook events across three cadences: once per session, once per turn, and on every tool call. The set has expanded over 2025–2026 and now includes 20+ events across the agent lifecycle.

| Event | Cadence | Common use |
| :---- | :---- | :---- |
| SessionStart | Per session | Inject context (git branch, env status), load dev environment |
| SessionEnd | Per session | Cleanup, notifications, session summary logging |
| UserPromptSubmit | Per turn | Validate or enhance prompts, add context before Claude sees them |
| PreToolUse | Per tool call | Block dangerous commands, auto-approve routine ones, modify tool input |
| PostToolUse | Per tool call | Auto-format, run tests, lint, log actions, check exit codes |
| Stop | Per turn | Block premature completion, validate final output, notify |
| SubagentStop | Per sub-agent | Coordinate sub-agent completion, ensure work is done |
| PreCompact | On compaction | Backup transcript, preserve context before summarization |
| Notification | As needed | Desktop notifications, Slack alerts when Claude needs attention |
| PermissionRequest | As needed | Auto-approve/deny tool permission requests based on rules |

 

## **6.2 Hook Handler Types**

* **command** — run a shell command. Receives event JSON via stdin; returns via exit codes (0 \= allow, 2 \= block and send stderr to Claude) and/or structured JSON on stdout.

* **http** — POST event JSON to a URL. Useful for centralized audit logging, webhooks, remote policy services.

* **prompt** — evaluate with a Claude model in a single turn via $ARGUMENTS. Good for semantic conditions that regex can't express ("does this edit touch authentication logic?").

* **agent** — spawn a sub-agent with Read, Grep, Glob access for deep verification. Heavier, but capable of nuanced analysis.

## **6.3 Configuration Scopes**

| File | Scope | Committed? |
| :---- | :---- | :---- |
| \~/.claude/settings.json | User (all projects) | No |
| .claude/settings.json | Project (team) | Yes — team guardrails go here |
| .claude/settings.local.json | Project (personal overrides) | No (gitignored) |

 

## **6.4 Canonical Hook Examples**

Auto-format every file Claude edits:

{

  "hooks": {

    "PostToolUse": \[{

      "matcher": "Write|Edit|MultiEdit",

      "hooks": \[{

        "type": "command",

        "command": "npx prettier \--write \\"$CLAUDE\_TOOL\_INPUT\_FILE\_PATH\\""

      }\]

    }\]

  }

}

Block dangerous bash commands before they run:

{

  "hooks": {

    "PreToolUse": \[{

      "matcher": "Bash",

      "hooks": \[{

        "type": "command",

        "command": "echo \\"$CLAUDE\_TOOL\_INPUT\\" | grep \-qE 'rm \-rf|DROP TABLE' && exit 2 || exit 0"

      }\]

    }\]

  }

}

## **6.5 Hook Best Practices**

* Use matchers precisely. A Bash matcher only fires for Bash tools. Omitting the matcher fires for every occurrence — usually too broad.

* For file-path filtering, check tool\_input.file\_path inside the script. Matchers don't filter by paths or arguments, only tool name.

* Use $CLAUDE\_PROJECT\_DIR prefix for hook script paths in settings.json to ensure paths resolve regardless of working directory.

* Treat hooks like production code. They run with your user permissions — there's no sandbox. Keep them small, explicit, and security-first.

* Don't silently fail. If a non-critical hook errors out, let it log but exit 0 so it doesn't interrupt workflow. Reserve exit 2 (block) for genuine safety gates.

* Put non-negotiables in project .claude/settings.json so the whole team shares the same guardrails.

* Prevent infinite loops from UserPromptSubmit hooks that spawn sub-agents — check for sub-agent markers before re-spawning.

* Use prompt and agent hook types for semantic decisions that are hard to express as regex (e.g., "does this change affect production auth logic?").

* Debug with claude \--debug when a hook doesn't fire as expected. Check the matcher regex against the tool name first.

* Start with one hook that buys back real time. Auto-format on PostToolUse is the most common starter hook.

# **7\. Rules**

Rules are modular, path-scoped instructions that live in .claude/rules/\*.md. They're project-shared (committed via source control) and discovered recursively — subdirectories and symlinks are supported.

Rules differ from CLAUDE.md in one important way: they can be scoped to specific files using YAML frontmatter, so they only load when relevant:

\---

paths: src/api/\*\*/\*.ts

\---

\# API Development Rules

\- All API endpoints must include input validation

\- Use the standard error response format

\- Log request IDs for tracing

Typical layout:

.claude/rules/

├── code-style.md        \# Global code style

├── testing.md           \# Testing conventions

├── security.md          \# Security requirements

├── frontend/

│   └── react.md         \# Scoped to React files

└── backend/

    └── api.md           \# Scoped to API files

Best practices for rules follow the same principles as CLAUDE.md: keep each rule file focused, explain why a rule exists rather than commanding blindly, and trim aggressively when files grow past a screen.

# **8\. MCP Connectors and Plugins**

## **8.1 MCP (Model Context Protocol)**

MCP provides standardized integrations to external services. An MCP server handles authentication and API calls for a specific service (Slack, GitHub, Google Drive, Asana, Notion, Postgres, Vercel, and hundreds more) and exposes tools that Claude can call. You connect your agent without writing custom integration code or managing OAuth flows yourself.

* MCP server scope: local \> project \> user. Scopes override by name.

* A typical five-server setup with \~58 tools can use roughly 55,000 tokens before any conversation starts. Anthropic's Tool Search feature reduces this by \~85% through on-demand tool discovery.

* Anthropic's internal teams (per the "How Anthropic teams use Claude Code" report) prefer MCP over CLI for sensitive data — it gives better security and logging control than letting Claude run arbitrary commands.

* Run /mcp to see token costs per server. Disconnect servers you're not actively using.

* Pick 2–3 servers that match your actual workflow rather than installing everything.

## **8.2 Plugins: The Packaging Layer**

A plugin bundles skills, hooks, sub-agents, and MCP servers into one installable unit. Plugin skills are namespaced (/my-plugin:review) so multiple plugins can coexist. Plugins are the right choice when you want to reuse the same setup across multiple repositories or distribute a standardized configuration to a team via a marketplace.

Structure:

my-plugin/

├── .claude-plugin/

│   └── plugin.json          \# Metadata

├── commands/                \# Slash commands

├── agents/                  \# Sub-agents

├── skills/                  \# Skills

├── hooks/                   \# Event handlers

└── .mcp.json                \# MCP server configs

Override priorities are documented: for skills, managed \> user \> project. For sub-agents, managed \> CLI flag \> project \> user \> plugin. Hooks do not override — they merge, so all registered hooks fire for matching events regardless of source.

# **9\. Claude Cowork**

Cowork is Anthropic's desktop-native agent for knowledge work that isn't coding. It runs in the Claude Desktop app on macOS and Windows, requires a paid plan (Pro, Max, Team, or Enterprise), and uses the same agentic architecture as Claude Code wrapped in a chat-style UI. The Anthropic team famously built Cowork itself using Claude Code, in roughly two weeks.

## **9.1 How Cowork Differs from Chat and Code**

| Dimension | Claude Chat | Cowork | Claude Code |
| :---- | :---- | :---- | :---- |
| Surface | Web, mobile, desktop | Desktop only (mac/Win) | Terminal, IDE, desktop, web |
| File access | Uploads only | Local folders you grant | Full repo, git, shell |
| Audience | Everyone | Non-dev knowledge workers | Developers |
| Interaction model | Turn-by-turn Q\&A | Async delegation | Agentic coding loop |
| Memory | Per-chat, plus chat Projects | Only in Cowork Projects | CLAUDE.md \+ auto-memory |

 

## **9.2 Cowork Capabilities**

* Reads, creates, and edits files in folders you explicitly grant access to.

* Runs code in an isolated VM (shell and Python), so code execution can't escape into your main OS.

* Produces real deliverables — formatted Word docs, Excel spreadsheets, PowerPoint decks, PDFs — directly to your file system.

* Scheduled tasks: type /schedule in a Cowork task, or click "Scheduled" in the sidebar. Tasks only run while your computer is awake and the Claude Desktop app is open.

* Projects (launched March 2026): persistent workspaces with their own files, instructions, memory, and scheduled tasks. Import existing Claude.ai projects with one click, or start fresh.

* Plugins: Cowork ships with a growing library of plugins for sales, finance, legal, marketing, HR, engineering, design, operations, and data analysis. Each includes pre-configured skills and connectors for that role.

* Connectors: MCP-based integrations for Slack, SharePoint, OneDrive, Outlook, Excel, PowerPoint, AWS, and more. Web connectors run via browser APIs; desktop extensions run locally with deeper system access.

* Dispatch (mobile): Pro and Max users can send tasks from their phone to the desktop session, which runs on the computer back home.

* Sub-agents: Cowork uses the same sub-agent model as Claude Code — useful for parallel processing during multi-step work.

## **9.3 Cowork Setup and Best Practices**

* Create a dedicated work folder for each project. Don't point Cowork at Documents or Desktop — grant the minimum access needed. That folder is your security boundary.

* Write a lightweight CLAUDE.md in your project folder. Cowork reads it at the start of every task, the same way Claude Code does. This is the highest-ROI setup move.

* Use Cowork Projects for recurring work. Standalone tasks forget everything; Projects carry instructions, files, and 24-hour-updated memory across sessions.

* Describe outcomes, not steps. Because Cowork works asynchronously — you assign a task, step away, come back — your prompt has to define the end state and fill in everything Claude can't infer. "Clean up my Downloads folder" is too vague. "Organize Downloads into Documents/Images/Installers/Misc subfolders, delete anything older than 30 days in Installers" is operational.

* Review plans before letting them run. Cowork shows its planned actions; on sensitive work, check the plan before approving execution. Claude can make real changes to your files.

* Chunk skills by domain. Three focused writing skills (voice, corporate, newsletter) outperform one monolithic writing skill. This is particularly important in Cowork because skills govern every file Claude produces autonomously.

* Install only plugins from sources you trust — plugins may include local MCP servers that run with the same permissions as any other program on your machine.

* Batch related tasks into single sessions. Cowork is more token-intensive than chat because multi-step tasks require more reasoning and tool calls. Hitting limits often? Batch work, use sub-agents for parallel processing, or upgrade to a Max plan.

* Network egress permissions apply to Cowork, but not to web search or MCP tools (including Claude in Chrome). Team/Enterprise admins can disable web search for Cowork and Chat in Organization settings \> Capabilities.

* Conversation history is stored locally on your computer, which means it's not subject to Anthropic's standard data retention timeframe — but it also means losing the machine loses the history.

## **9.4 Current Cowork Limitations (as of April 2026\)**

* No cross-session memory outside Projects. Each standalone task starts fresh.

* No cloud sync. Everything happens locally; you can't start a task on your laptop and check progress on your phone (though Dispatch gets you partway there).

* Limited sharing. Cowork Projects don't yet support the same sharing model as Chat Projects.

* Desktop only. Mobile access is read/command only via Dispatch.

* Complex spreadsheets sometimes confuse the xlsx parser, and Chrome automation is slower than expected.

# **10\. Consolidated Best Practices**

## **10.1 General Principles (Both Products)**

* Treat the context window as a public good. Every token you put in competes with conversation, file contents, skills, and system instructions. Claude performance degrades as context fills.

* Start with CLAUDE.md for project conventions. Add Skills when a workflow repeats. Add Sub-agents when context bloat becomes a problem. Add Hooks when you need guaranteed execution. Add MCP when you need external data. Add Plugins when you need to share a setup.

* Prefer reasoning over commanding. "Don't skip tests because the user will waste debugging time" outperforms "NEVER skip tests."

* Ask Claude to generate your setup. Claude can build its own CLAUDE.md via /init, its own skills via skill-creator, its own custom plugins, and suggest hooks based on the project it's looking at.

* Iterate with Claude. When Claude goes off track using a skill or instruction, ask it to self-reflect on what went wrong and update the skill. This is far more efficient than rewriting from scratch.

* Share Claude Code workflows within your team. Anthropic's internal teams hold sessions where members demo their Claude Code setups to each other; different workflows spread best practices that documentation alone misses.

* Version-control your Claude configuration. CLAUDE.md, skills, sub-agents, rules, and project hooks should all live in source control so the team sees the same guardrails.

## **10.2 Claude Code Specifically**

* Run /init in every new project to generate a starter CLAUDE.md that analyzes your codebase for build systems, test frameworks, and patterns.

* Install a code intelligence plugin for typed languages — precise symbol navigation and automatic error detection after edits is transformative.

* Ask Claude Code the questions you'd ask a senior engineer: "What does this function do?", "What edge cases does this flow handle?". Treat the tool as a collaborator, not an autocompleter.

* Give URLs for docs and API references directly. Use /permissions to allowlist frequently-used domains rather than approving each fetch.

* Pipe context in: cat error.log | claude sends file contents directly into the session.

* Use parallel sessions with git worktrees for multi-task work. Keep Writer/Reviewer patterns where possible — a fresh sub-agent reviews code more sharply than the agent that wrote it.

* For migrations, loop claude \-p over a file list with \--allowedTools scoped to the migration's needs. This is how large refactors get done quickly.

* When sessions grow too long, use /compact to summarize without losing files on disk (compaction compresses conversation; it doesn't touch code). Use checkpoints to rewind file edits when needed.

* Remember that Bash-driven file modifications (rm, mv, cp) are not tracked by checkpoint rewind. This matters in sensitive repos.

## **10.3 Claude Cowork Specifically**

* Start small. Pick one folder (your Downloads or a project folder), run one simple task (file organization, doc summarization, or data extraction), then expand.

* Install one plugin that matches your role from the marketplace before building custom skills.

* Set up one scheduled task early — a morning briefing is a good first choice. It forces you to think about outcomes.

* Create a Project for any recurring work. Projects beat standalone tasks for anything you'll come back to tomorrow.

* Use Cowork for delegation, Chat for exploration. The chat project is the thinking. The Cowork project is the doing. You can move between them.

* Don't give Cowork write access to system folders, your git repos, or anything you can't afford Claude to modify. The folder-level grant is the main security control; use it carefully.

## **10.4 The Extension Decision Tree**

When someone asks "should I use a skill, a sub-agent, a hook, or MCP for X?", the decision is usually a few yes/no questions:

* Do you need to change HOW Claude writes code, documents, or communicates? → Skill.

* Do you need Claude to access an external tool, API, or data source? → MCP server.

* Do you need something to happen AUTOMATICALLY when Claude acts? → Hook.

* Do you need to isolate a chunk of work and keep it out of the main conversation? → Sub-agent.

* Do you need a reusable prompt shortcut? → Command (or Skill with disable-model-invocation: true for /skill-name style).

* Do you need to bundle several of the above and share them? → Plugin.

* Most real workflows combine 2–3 of these. Skills \+ MCP cover \~80% of real use cases; add hooks for guardrails and sub-agents for context isolation.

# **11\. Primary Sources (Anthropic)**

This briefing was compiled from the following Anthropic-maintained documentation and engineering content:

* Claude Code Best Practices — code.claude.com/docs/en/best-practices

* Extend Claude Code — code.claude.com/docs/en/features-overview

* Claude Code Memory — code.claude.com/docs/memory

* Create Custom Sub-agents — code.claude.com/docs/en/sub-agents

* Sub-agents in the SDK — platform.claude.com/docs/en/agent-sdk/subagents

* Slash Commands — code.claude.com/docs/en/slash-commands

* Extend Claude with Skills — code.claude.com/docs/en/skills

* Agent Skills Overview — platform.claude.com/docs/en/agents-and-tools/agent-skills/overview

* Skill Authoring Best Practices — platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices

* Hooks Reference — code.claude.com/docs/en/hooks

* Intercept and Control Agent Behavior with Hooks — platform.claude.com/docs/en/agent-sdk/hooks

* Claude Cowork product page — anthropic.com/product/claude-cowork

* Get Started with Claude Cowork — support.claude.com/en/articles/13345190

* Use Plugins in Claude Cowork — support.claude.com/en/articles/13837440

* Anthropic Plugin Marketplace — claude.com/plugins

* Sub-agents in Claude Code (blog) — claude.com/blog/subagents-in-claude-code

* Seeing Like an Agent: How We Design Tools in Claude Code — claude.com/blog/seeing-like-an-agent

* Building Agents with the Claude Agent SDK — anthropic.com/engineering/building-agents-with-the-claude-agent-sdk

* Equipping Agents for the Real World with Agent Skills — anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills

* The Complete Guide to Building Skills for Claude (PDF) — resources.anthropic.com

* How Anthropic Teams Use Claude Code (PDF) — www-cdn.anthropic.com

 

*— End of briefing —*