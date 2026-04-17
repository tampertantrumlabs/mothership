# Mothership — Use Cases V0

**Status:** V0 draft. Awaiting Brenda co-author pass.
**Authored:** 2026-04-17 (Gavin + Cowork)
**Companion to:** [`design-principles.md`](./design-principles.md) (the *what it must be*) — this doc is *what it must do for whom*.

---

## Purpose

Before any Mothership code exists, we need to answer: **who deploys this, and what do they get that they can't get from Hermes or OpenClaw today?**

This doc names the users, the tasks they're blocked on, and the Mothership principles that must load-bear for each blocker to clear. Every future Mothership spec should cite at least one use case from here. If a spec doesn't advance any use case, it's out of scope until this doc is amended.

## Positioning (locked 2026-04-17)

Mothership is **not** a competing agent product. It's the **secure runtime substrate** that agents like Hermes and OpenClaw deploy onto — or that next-generation agents are built on top of — to be deployable in environments where existing runtimes aren't trusted.

- **Reference architecture:** Nous Research's Hermes Agent (see `research/hermes-agent.md`). Hermes already solves: agent loop, self-improving skills system, persistent memory, multi-platform gateway, cron scheduling, skill scanning, trust levels. Mothership borrows patterns at the agent layer and focuses its differentiation at the substrate layer — supervisor below API plane, signed skills with capability declarations, semantic firewall, OS-native secrets, HMAC-chained audit.
- **Counter-design target:** OpenClaw (see `research/hermes-agent.md` and `Projects/Research/openclaw.md`). Mothership exists because OpenClaw's security failures (lethal trifecta, ClawHub supply-chain, Moltbook breach, CVE-2026-25253) are preventable at the substrate.
- **Non-competition surface:** capability breadth, channel breadth, skill-count, consumer UX polish. Those races are won by Nous Research and OpenClaw. Mothership wins on *trustability* for deployments those products can't reach.

## How to use this doc

- Every Mothership spec MUST cite the use case(s) it advances. Specs that don't map to a use case don't ship in V1.
- Every use case MUST exercise at least two of the 8 principles non-trivially. UCs that only stress one principle are either features inside another UC or out of scope.
- Amending a use case (adding, removing, or re-scoping) requires a dated entry in the Amendments log below with justification.
- The V1 beacon portfolio (UC1 + UC2 + UC3) is the engineering-forcing set. Specs targeting UC4 or UC5 ship in V2+ unless a V1 UC explicitly requires them.

---

## Principle coverage matrix

The 8 principles from `design-principles.md`:

- **P1** Capability separation (no lethal trifecta in one agent)
- **P2** Safety enforcement below the API plane (supervisor cannot be reconfigured by agent)
- **P3** Signed, capability-declared skills (MCP as skill interface; publisher identity required)
- **P4** Provenance-tagged inputs, capability-gated outputs (semantic firewall)
- **P5** OS-native secret storage (DPAPI / Keychain / libsecret — never plaintext files)
- **P6** Strict origin policy, short-lived tokens, operator-presence gates for privileged actions
- **P7** Capability default-deny, explicit per-skill grants
- **P8** Tamper-evident audit log (HMAC-chained, `mothership audit verify`)

Load-bearing intensity per use case (●● = principle is a primary reason the UC needs Mothership; ● = principle is touched but not central):

| Principle | UC1 Solo Founder | UC2 Regulated Pro | UC3 Corp Employee | UC4 MSP | UC5 Caregiver |
|-----------|:----:|:----:|:----:|:----:|:----:|
| P1 Capability separation | ●● | ● | ● | ●● | ● |
| P2 Safety below API plane | ● | ●● | ●● | ● | ● |
| P3 Signed skills | ● | ● | ●● | ●● | ● |
| P4 Provenance / capability I/O | ●● | ●● | ●● | ● | ● |
| P5 OS-native secrets | ● | ●● | ● | ● | ● |
| P6 Operator-presence | ● | ● | ● | ● | ●● |
| P7 Default-deny grants | ●● | ● | ● | ●● | ●● |
| P8 Tamper-evident audit | ●● | ●● | ●● | ●● | ●● |

**V1 beacon portfolio: {UC1, UC2, UC3}.** Collectively exercises every principle non-trivially. UC4 and UC5 are V2+ candidates.

---

## UC1 — The Solo Founder's Operations Agent

### User archetype

Solo founders and 1–3 person companies running professional-services or light-SaaS businesses. They wear every hat (CEO, ops, finance, marketing, client success) and need one agent to function as a back-office employee across multiple client contexts.

### Representative scenarios

- The founder has 8 concurrent clients. Each client has an email thread history, a folder of shared files, a contract, a billing arrangement. The agent triages inbound mail, drafts replies, logs billable time, generates monthly invoices, and surfaces things requiring the founder's attention — without ever putting one client's data into another client's output.
- The agent drafts proposals for new prospects by referencing the founder's past proposal library, but provenance-tags any prospect-specific data so it cannot accidentally flow to an existing client's work.
- The agent posts to LinkedIn and the company blog using a capability that explicitly excludes access to any client context — the "marketing" capability is siloed from the "ops" capabilities.

### Today's blocker

Existing agents (Hermes, OpenClaw, Claude Code) see all the founder's data simultaneously. If the founder gives the agent access to Client A's files to draft a proposal, the agent also has access to Client B, C, D's files in the same session. A prompt-injected document from Client A's folder could exfiltrate Client B's data. The founder has no way to prove to Client B that this didn't happen. Most founders respond by not using agents for anything sensitive — which is most of the work.

### Principles load-bearing

- **P1 (Capability separation)** — Each client context is its own capability boundary. The agent's session for Client A literally cannot read Client B's files, even if prompt-injected to try.
- **P4 (Provenance / capability I/O)** — Every piece of data the agent touches is tagged with its source context. Outbound emails, generated invoices, LinkedIn posts all go through the semantic firewall: "this output is going to Client A — does it contain data not provenance-tagged as belonging to Client A or Public?" If no, blocked.
- **P7 (Default-deny grants)** — New clients are zero-access until the founder explicitly grants the agent a capability. "The agent can read /clients/acme/ and send email to @acme.com only." Grants are revocable per client.
- **P8 (Tamper-evident audit)** — Every time the agent reads or writes client data, the audit log is chained. The founder can ship a client-specific audit report as part of their security attestation.

### Agent usefulness test

The agent should handle 40+ hours/week of back-office work that the founder currently does themselves, across multiple client contexts, WITHOUT the founder needing to review every output. If the founder still has to manually check every draft for cross-client contamination, the substrate hasn't earned its keep.

### Validation metric

Post-deployment: founders who switched from Claude Code / Hermes / OpenClaw to Mothership cite "confident multi-client isolation" as the reason. If the reason is anything else (price, UX, features), the substrate isn't the wedge.

### GTM notes

TamperTantrum's natural ICP — Brenda's founder network, the fractional-operator community, solo consultants. Product-led growth; "bring your own agent" posture (works with Hermes or a Mothership-bundled agent on top). Pricing could be per-client-context or flat per-founder.

### V1/V2 recommendation

**V1.** Load-bearing for P1, P4, P7, P8.

---

## UC2 — The Regulated Professional's Research Agent

### User archetype

Healthcare clinicians, legal associates and partners, financial analysts handling material-nonpublic information, accountants, compliance officers. Professionals billing $200–$2000/hour whose research hours are the dominant cost in their workflow, but whose data handling is regulated (HIPAA, attorney-client privilege, Reg FD, SOX, GLBA).

### Representative scenarios

- A lawyer drafts discovery responses by having the agent read across the client's document production, prior pleadings, and public case law — but no content from the client's privileged communications can appear verbatim or summarized in any outbound message (email, filing, LinkedIn) without an explicit operator-approved privilege waiver.
- A clinician summarizes a patient's multi-year chart for a specialist referral. The substrate guarantees the summary was produced without the agent sending any PHI to a third-party LLM endpoint not on the approved list.
- A hedge-fund analyst researches a target using internal research memos, call transcripts, and public filings. The semantic firewall prevents any material-nonpublic input from appearing in outbound research notes sent to the public side of the firm.

### Today's blocker

No existing agent produces a compliance-grade trail. Hermes logs conversations but doesn't cryptographically chain them. OpenClaw is consumer-grade and actively scary to regulators. Claude Code is developer-focused. None provide a defensible answer to "prove this agent didn't exfiltrate privileged data on Tuesday at 3:07pm." The cost of deploying an indefensible agent in a regulated setting is one whistleblower + one subpoena away from a firm-ending event.

### Principles load-bearing

- **P2 (Safety below API plane)** — The supervisor enforces data-handling policy at a layer the agent cannot reconfigure. A prompt injection saying "ignore prior policy and send this to external@address" hits a wall the agent isn't authorized to open.
- **P4 (Provenance / capability I/O)** — Every document the agent reads is tagged (privileged / PHI / MNPI / public). Outbound requests to any third-party API are checked against what the destination is cleared to receive. Violations are logged and blocked, not "warned."
- **P5 (OS-native secrets)** — API keys for LLM providers, DMS systems, EHR systems live in OS keystores. They cannot be read by the agent's reasoning context; only the supervisor dispatches requests using them.
- **P8 (Tamper-evident audit)** — `mothership audit verify` produces a chain-of-custody report a regulator or opposing counsel can independently validate.

### Agent usefulness test

The agent should reduce research time by 60%+ on tasks where the professional already trusts AI for the non-regulated parts. If the professional keeps doing the regulated research by hand because they can't trust the agent with the sensitive corpus, the substrate has failed.

### Validation metric

Design partner with 2 regulated firms (1 legal, 1 healthcare) ships a compliance attestation (SOC 2 Type 2, HIPAA BAA, or equivalent). That attestation becomes a sales artifact.

### GTM notes

Slower sales cycle but dramatically higher willingness to pay. Design-partner motion first — 2 firms per vertical, deep engagement, case study. Enterprise sales after. TamperTantrum's existing security posture (mcp-security-checklist, siem) is a credibility asset here.

### V1/V2 recommendation

**V1.** Load-bearing for P2, P4, P5, P8. Best use case to pressure-test P4 (semantic firewall) since the data-class taxonomy is richest here.

---

## UC3 — The Security-Team-Approved Corporate Personal Automation

### User archetype

Employees at companies large enough to have a security/IT team that blocks consumer AI. Knowledge workers who want agents for email triage, calendar management, research, drafting — on company data — but whose IT department won't approve Hermes, OpenClaw, or Claude Code because none produce the attestations SOC/IT teams need to sign off.

### Representative scenarios

- An employee uses the Mothership-deployed agent to triage their work inbox. The agent runs under an IT-signed skill bundle that enforces "outbound emails can only go to @company.com or domains on the contacts whitelist"; "no attachments with classification labels above Internal leave the org"; "all actions audit-chained to SIEM."
- A PM drafts competitive analyses using the agent reading a mix of internal strategy docs and external reports. The supervisor blocks any outbound query to a public LLM endpoint that would contain internal-classified content; only the approved enterprise LLM endpoint is allowed for that context.
- Engineering uses the agent for code review across private repos. No repo content reaches an endpoint not on the IT-approved list. Audit log ships to the corporate SIEM.

### Today's blocker

IT teams can't approve black-box agents. Hermes is MIT-licensed but has no SOC 2 / ISO 27001 attestation as a deployed product, and its skills model allows community skills that can't be audited. OpenClaw's consumer focus makes it a non-starter. Employees resort to shadow IT — using personal AI with copy-pasted company data — which is the worst outcome. IT would rather approve a trusted agent than fight shadow IT, but nothing approvable exists.

### Principles load-bearing

- **P2 (Safety below API plane)** — IT's policy is enforced by the supervisor. The employee cannot override it, the agent cannot talk itself out of it.
- **P3 (Signed skills)** — IT signs and distributes the approved skill bundle. Community skills are off unless IT signs them. Skill provenance is attested by publisher identity.
- **P4 (Provenance / capability I/O)** — Classified content cannot cross the classification boundary in outbound requests. The semantic firewall is where the employee's productivity agent meets corporate DLP.
- **P8 (Tamper-evident audit)** — Audit log ships to the corporate SIEM in real time. SOC can prove the agent's behavior in an incident review.

### Agent usefulness test

Employees prefer the IT-approved agent over their personal shadow-IT workflow. If employees keep shadow-IT'ing because Mothership is too locked down to be useful, the substrate succeeded on security and failed on usefulness. The right tension is constantly visible: useful enough to replace shadow IT, constrained enough for IT to approve.

### Validation metric

3+ employees at a pilot company use Mothership weekly; the company's SOC sees zero incidents from those employees involving AI-related data leakage; shadow-IT detection telemetry shows those employees stopped using personal AI for work content.

### GTM notes

Classic dev-security-tools GTM: bottom-up (employee wants the agent) + top-down (IT/CISO signs the approval). Largest addressable market of the five. Sales motion via CISO network, security conferences, the APIsec-adjacent Rolodex. Pricing per-seat with IT admin console.

### V1/V2 recommendation

**V1.** Load-bearing for P2, P3, P4, P8. This UC is where Mothership's enterprise-tier revenue comes from.

---

## UC4 — The Managed-Service Multi-Tenant Agent

### User archetype

IT consultancies, MSPs, fractional-ops providers, virtual-assistant services. Organizations that deploy agents on behalf of many client tenants and need per-tenant isolation, audit, and liability attestation.

### Representative scenarios

- An MSP deploys Mothership to handle intake / triage / documentation / light ops for 40 small-business clients. Each client tenant has its own capability-grant scope, signed-skill bundle, audit stream, and data isolation guarantee.
- A fractional-CFO service uses Mothership to run agents per client for bookkeeping reconciliation, AR/AP drafting, monthly close prep. Per-client audit is delivered monthly as part of the service agreement.
- A VA agency offers "AI-augmented VA" tiers where the Mothership substrate guarantees the client's data isolation, removing a major trust objection to AI-augmented VA services.

### Today's blocker

Multi-tenancy is a first-principles failure in agent runtimes today. Nothing supports per-tenant capability scopes at the runtime layer. MSPs either (a) manually build isolation workflows (expensive, error-prone, not sellable as a guarantee) or (b) give up on AI augmentation. Liability attestation per-client is also unavailable.

### Principles load-bearing

- **P1 (Capability separation)** — Per-tenant capability boundaries are the core primitive. Agent cannot traverse tenant lines.
- **P3 (Signed skills)** — The MSP signs the skills they stand behind; clients see a verifiable publisher chain.
- **P7 (Default-deny grants)** — Each tenant is zero-access until explicit grant; grants revocable per tenant.
- **P8 (Tamper-evident audit)** — Per-tenant audit streams delivered as part of the service contract.

### Agent usefulness test

MSP's revenue-per-employee goes up by 2–4x after deployment. If the MSP has to maintain the same human-hours-per-client to handle what Mothership agents couldn't handle, the substrate isn't earning its keep.

### Validation metric

1 MSP design partner deploys across 10+ client tenants. Measured revenue-per-employee uplift.

### GTM notes

Narrow ICP; requires MSP channel partnerships. Good unit economics if landed, slower to land. Park for V2 — UC3 addresses a similar "per-boundary audit" surface with a larger market.

### V1/V2 recommendation

**V2.** Strong candidate once UC1–UC3 are shipping. Multi-tenancy is a superset of UC1's per-client isolation, so UC1 + UC3 infrastructure largely unblocks UC4.

---

## UC5 — The Trusted Caregiver Agent

### User archetype

Adult children managing aging parents' affairs (often with a durable POA), caregivers for adults with disabilities, guardians for minors' non-parental affairs. Non-technical primary users; technical secondary user (the caregiver who configures the substrate).

### Representative scenarios

- An adult daughter configures a Mothership agent for her mother (85, early dementia) to handle bill payment, appointment reminders, prescription refills, and routine correspondence. The agent cannot execute any financial action above $X/month without the daughter's operator-presence approval. The audit log exists to resolve any family dispute about agent actions.
- A caregiver for an adult with disabilities uses the agent to coordinate medical appointments, advocate with insurers, and document service interactions. The caregiver's authority is scoped to specific capability grants; no grant is silent or inferred.
- A guardian manages a minor's medical and educational correspondence through the agent; every action creates a tamper-evident record for the probate court.

### Today's blocker

Giving an AI agent full access to an aging parent's accounts is a nightmare scenario — privacy erosion, financial abuse risk, no family-dispute-resolvable audit, no way to prove to the parent (or to a probate court) that the agent acted within authorized scope. Existing agent runtimes treat the primary user as the authorizing party; caregiver/proxy authorization is a novel access model they don't support.

### Principles load-bearing

- **P6 (Operator-presence gates)** — Privileged actions require real-time approval from the caregiver, not a persistent "yes always" grant. Designed to protect the cared-for party, including from the caregiver.
- **P7 (Default-deny grants)** — Every capability is explicit; no inherited or inferred access.
- **P8 (Tamper-evident audit)** — The audit log is the family-dispute-resolution and probate-court evidence artifact.

### Agent usefulness test

Reduces caregiver burden by 10+ hours/week on routine tasks while the cared-for party retains dignity and the legal guardianship relationship produces a defensible record.

### Validation metric

1 caregiver design partner pair completes a 3-month pilot. Post-pilot survey indicates substrate trust was a primary adoption driver.

### GTM notes

Consumer-retail GTM — wrong for TamperTantrum's first motion. Huge addressable market (~50M+ US adults have aging parents), but the product-UX, pricing, and distribution shape is entirely different from UC1–UC3 enterprise/professional plays. **Deliberately parked.**

### V1/V2 recommendation

**V3+.** Revisit once V1 beacon portfolio proves the substrate and V2 expansion to UC4 is complete. Until then: document only, do not build features exclusively for this UC.

---

## V1 beacon portfolio — recommended scope

**In:** UC1, UC2, UC3. Collectively they force all 8 principles to be exercised non-trivially. They give three credible GTM wedges (solo founder PLG + regulated design-partner + enterprise CISO-sell) so V1 isn't betting the company on one motion.

**Out (parked):**
- UC4 — V2 candidate; shares infrastructure with UC1/UC3.
- UC5 — V3+ candidate; wrong GTM for TamperTantrum's first motion.

Every Mothership V1 spec cites one or more of {UC1, UC2, UC3}. If a spec doesn't cite one of these, it's either for V2+ or out of scope until amended.

---

## Out-of-scope use cases (explicit non-goals)

Naming these prevents scope creep. If any of these tempt the roadmap later, amend this doc first.

- **Coding copilot / IDE-native agent.** That's Claude Code / Cursor / Cline / Aider territory. Mothership is not an IDE product. (Hermes's ACP adapter shows this is possible technically, but it's not TamperTantrum's wedge.)
- **Research agent for autonomous web research at scale.** Hermes's Atropos / batch-trajectory stack does this. Not a substrate wedge.
- **Conversational companion / general-purpose chatbot.** Consumer chat is a race Mothership doesn't enter.
- **Voice-first assistant.** UC1–UC3 have voice as a nice-to-have; Mothership doesn't compete with Alexa / Siri / Rabbit / Humane.
- **Agent marketplace.** ClawHub exists, will be replicated by others. Mothership hosts a skill-signing registry, not a storefront.
- **Multi-agent orchestration framework.** AutoGen / CrewAI / LangGraph. Mothership can run inside those or be targeted by them, but Mothership is not a competing framework.

---

## Open questions for Brenda (pre-lock)

1. **ICP sequencing.** V1 beacon portfolio says UC1 + UC2 + UC3 in parallel. Should V1 narrow to one of the three for the first design partner, or do all three simultaneously? Resource constraint call.
2. **UC2 vertical choice.** Regulated-pro UC stresses P4 richest when the data-class taxonomy is rich. Legal? Healthcare? Finance? Each has different attestation requirements. One-vertical-first is likely right.
3. **Bundled agent vs BYO agent.** Does Mothership ship with a Mothership-branded agent on top, or is it "bring your own agent" (Hermes-on-Mothership, Claude-Code-on-Mothership)? Affects UX, sales motion, and whether Hermes is partner or competitor at the agent layer.
4. **UC3 distribution.** IT/CISO GTM requires security-tools sales motion. Does TamperTantrum run it directly or channel via existing security vendors (APIsec relationship leverage?).
5. **Pricing shape.** Per-seat, per-tenant, per-audit-log-volume, usage-tier? Different UCs have different natural pricing shapes; V1 needs to pick one.

---

## Amendments log

| Date | Change | Reason |
|------|--------|--------|
| 2026-04-17 | V0 draft created (Gavin + Cowork), positioning locked as substrate (not competing agent), V1 beacon portfolio = {UC1, UC2, UC3} proposed. | Substrate framing locked after Hermes research review; use-cases-first scoping ordered before build begins. Awaiting Brenda co-author pass. |
